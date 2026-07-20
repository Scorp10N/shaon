# authveil: credential-isolating proxy for shaon (design)

**Date:** 2026-07-20
**Status:** Approved for planning

## Context

`shaon` automates a personal Hilan HR/payroll account and is driven both by a
human CLI and by AI agents through its MCP server (see the 2026-07-10
security review, `shaon-security-review.md`). That review found the codebase
itself clean, but two real gaps surfaced when trying to actually run it
against the owner's account:

1. The account requires TOTP MFA on login. `shaon` has no MFA support at all,
   and deliberately so — `CONTRIBUTING.md` scopes out CAPTCHA/MFA bypass as
   off-limits for the project. Extending `shaon` itself to solve MFA would
   mean forking that explicit boundary.
2. Even if MFA were solved, `shaon`'s current model means the real Hilan
   password, TOTP seed, subdomain, and employee ID all live in `shaon`'s own
   config/keychain — directly readable by anything with shell access to the
   machine `shaon` runs on, including an AI coding agent invoked in that
   session (demonstrated directly in this repo: an agent session read
   `~/.shaon/config.toml` in plain text without any special privilege).

The goal of this design is to remove both problems by moving all Hilan
authentication (password + TOTP MFA) and the real subdomain out of `shaon`'s
reach entirely, onto a separate service on a separate host that `shaon` talks
to over a private network. `shaon` itself keeps working unmodified from the
CLI/MCP-tool-surface point of view; it just never touches a real credential
again.

## Architecture

```
[shaon CLI/MCP]  --HTTPS (Tailscale, private)-->  [authveil container]  --HTTPS-->  [real hilan.co.il]
  (dev machine)                                     (MyLocalLLM docker host)
  config: proxy_url = https://authveil.<tailnet>/
          subdomain = "work"  (opaque local label only —
                                never the real company subdomain)
```

`authveil` is a new, standalone project: a generic credential-isolating
reverse proxy for legacy web portals that only support username+password+TOTP
login (no OIDC/SAML — otherwise you'd just use a normal SSO broker, and this
wouldn't be needed). It holds the real credentials and MFA secret for a
target portal, logs in and refreshes sessions on its own, and reverse-proxies
an authenticated session to a client that never sees the real domain or
credentials. Hilan is its first adapter; the design keeps the Hilan-specific
logic behind an adapter interface so the same proxy can front other
non-SSO portals later.

We looked at reusing an existing open-source broker instead of building this
(Teleport Application Access, Pomerium, oauth2-proxy, Boundary, Infisical,
Vault Agent, and MCP-specific projects like `zerocreds-mcp` / `secrets-mcp` /
`qred-mcp-proxy`) — confirmed via independent research (Antigravity). None of
them fit: they all broker access *into* something that already speaks
OIDC/SAML, or they inject secrets into a process/tool-call rather than
autonomously logging into a legacy web form and reverse-proxying the
resulting session. `zerocreds-mcp` is the closest conceptual match (browser
automation so an AI client never sees raw credentials) but is an agent
tool-runner, not an HTTP reverse proxy. The standard real-world shape for
this exact problem is a bespoke sidecar, which is what `authveil` is.

## Components

### 1. `shaon` (this fork) — small patch, not a rewrite

- New optional `proxy_url: Option<String>` field on `Config`
  (`crates/provider-hilan/src/config.rs`).
- `HilanClient::new` (`crates/provider-hilan/src/client.rs:148`) uses
  `proxy_url` verbatim as `base_url` when set, instead of building
  `https://{subdomain}.hilan.co.il`.
- When `proxy_url` is set, `session_candidate` is forced `true` for the
  client's lifetime and `login()` / `get_password()` are never called. No
  `shaon auth`, no keychain password entry, ever, in this mode. `subdomain`
  becomes a purely local label used only for cache-directory naming
  (`subdomain_dir`) — it can be any string the user picks (e.g. `"work"`)
  and never needs to resemble the real company subdomain.
- If the upstream (authveil's) session ever lapses and it can't recover,
  existing error surfaces propagate as-is (the same `calendar_read_failed`
  style errors already seen when a session is invalid) — `shaon` reports the
  failure and does not attempt to re-authenticate itself.

### 2. `authveil` (new repo)

- Depends on `provider-hilan` as a git dependency pinned to this fork's
  commit, reusing its already-reviewed Hilan login/cookie/wire-protocol code
  (`HilanClient::login_with_password`, cookie encryption, etc.) rather than
  reimplementing it.
- `PortalAdapter` trait (`login`, `is_session_expired`, `refresh`) —
  Hilan is the first, only implementation to start.
- TOTP: generates the current code from a stored seed using standard RFC
  6238 (e.g. the `totp-lite` crate) and submits it as part of login. **The
  exact request/response shape of Hilan's MFA step is currently unknown** —
  `PROTOCOL.md` and `LoginResponse` (`client.rs:90`) only document the
  no-MFA path. First implementation step is capturing the real wire traffic
  from a manual browser login (devtools network tab) before writing any
  adapter code against assumptions.
- Transparent reverse proxy: forwards whatever arrives on its listen port to
  `https://{real-subdomain}.hilan.co.il` with the real authenticated session
  cookies injected, streaming the response back unmodified. Proactively
  re-authenticates (password + TOTP) when the upstream session expires, so
  `shaon` never observes an auth failure in normal operation.
- Reachable only over a private Tailscale address (reusing the same
  Tailscale-sidecar pattern already used for `resume-funnel`), not the
  public internet.

### 3. Secrets

Reuses the existing MyLocalLLM convention exactly, no new tooling:

- `secrets.enc.yaml` + `.sops.yaml` in the `authveil` repo (sops + age,
  reusing the existing TPM-sealed age recipient, or minting a
  project-specific one) holding: real subdomain, employee ID, password, TOTP
  seed. Ciphertext only — safe to commit.
- `write-secrets.sh` decrypts at container start into
  `/dev/shm/authveil-secrets/` (tmpfs, RAM-only, `chmod 700`/`600`) —
  mirrors `MyLocalLLM/scripts/write-secrets.sh`.
- Wired into `docker-compose.yml` via a top-level `secrets:` block, consumed
  as `/run/secrets/*` inside the `authveil` container — standard Docker
  file-based secrets, not swarm.
- Nothing plaintext ever touches persistent disk on the proxy host; nothing
  at all (plaintext or ciphertext) touches `shaon`'s host.

## Residual threat model

Moving auth out of `shaon` doesn't eliminate risk, it relocates it: anything
that can reach `authveil` over the Tailscale network gets a fully
authenticated Hilan session with no password needed — the proxy itself *is*
the credential now, in network-reachability form rather than
password+TOTP form. Mitigations already in place or carried over unchanged:

- `authveil` is reachable only over the private Tailscale network, never a
  public address.
- `shaon`'s existing write-gating is untouched and still applies on top: the
  MCP server's `execute` flag defaults to `false` in code, and CLI writes
  require `--execute`. A compromised or confused agent talking to `shaon`
  can read through the proxy but still cannot submit a real attendance
  change without an explicit execute flag.
- Session lifetime on the `authveil` side should be kept as short as
  practical (subject to Hilan's own session/cookie limits) rather than
  cached indefinitely, so a leaked reachability window is bounded.

## Verification plan

1. Capture real Hilan MFA wire traffic (browser devtools, manual login)
   before writing any TOTP-submission code in the adapter — do not guess at
   the request/response shape.
2. Unit-test `authveil`'s reverse-proxy and session-refresh logic against a
   mock Hilan server (httpmock or similar), independent of real credentials.
3. Deploy the container on the MyLocalLLM stack over Tailscale, point a
   `shaon` config at it with `proxy_url` set, and run the same read-only
   `shaon attendance status` check attempted earlier in this session against
   the real account, end to end.

## Explicitly out of scope for this design

- Any adapter other than Hilan (the trait exists for future reuse, but only
  Hilan gets implemented here).
- Automating non-TOTP MFA types (SMS/push) — not needed for this account,
  and meaningfully harder to automate safely.
- Any change to `shaon`'s own write-gating model — it is reused as-is.
