# Prototyped Features

A catalog of everything proven in the `poc/` phase of practical-ssh-mcp. Each
bullet is a self-contained proof-of-concept (mostly `go test`-runnable, several
container- or host-backed). Grouped by the architectural layer it belongs to.

## Connections & transport (layer 1)

- **Warm, multiplexed connections** (`connections`) — one warm `*ssh.Client` per
  host, every operation a channel on it, auth/jump-hops paid once; keepalive
  supervisor detects drops and auto-reconnects, surfacing `connection_down` /
  `connection_up` on the reactor. Includes jump-chain (`ProxyJump`) support and
  connection reuse.
- **Forwarding channels** (`channels`) — `forward.tcp` / `reverse.tcp` /
  `forward.unix` / `reverse.unix` / `http.fetch` / `http.serve` / `proxy.socks`:
  direct-tcpip, reverse listeners, remote unix-socket dialing (the docker.sock
  workflow), tunneled HTTP, and a local SOCKS5 proxy — persistent channels feed
  accept/data/close events into the reactor.

## Execution

- **Mild-prefix executor** (`mildexec`) — takes a connect-plan, executes the
  "mild" steps for real (jump → connect → resolve → survey), then halts at the
  first "hot" step (join-fabric/enrol/mint) and returns a consent request.
- **Survey battery** (`survey`) — fixed, read-only, allowlisted capability probes
  run on a host and cached (TTL + invalidate) to save round-trips.
- **Privilege-drop isolation** (`privdrop`) — a named, reusable, verifiable
  isolation spec wrapping every `exec` (`sudo -u … setpriv …`), picking the best
  mechanism available on the remote; per-call, repeatable default, probe, and
  verify.

## Files & transfer

- **Structured remote file ops** (`fileops`) — read/edit/write/stat/list/mkdir/
  rename/remove/chmod over SFTP; **atomic-by-default** writes (temp sibling +
  posix-rename) with a detected fallback to in-place writes for virtual
  filesystems (`/proc`, `/sys`, `/dev`).
- **Bulk transfer** (`transfer`) — background SFTP upload/download emitting
  `transfer_progress` / `transfer_done` events, plus `transfer_sync`: rsync delta
  transfer over the warm SSH connection using the pure-Go gokrazy/rsync client
  (push/pull, delete, dry-run, include/exclude filters).

## Interactive sessions & terminals

- **Persistent PTY sessions** (`sessions`) — a PTY whose process/cwd/env survive
  across calls, with **stream** and **screen** presentation modes that auto-switch
  on the alternate-screen escape, named-key input, and a 2D screen snapshot with
  styles overlay (vt10x emulator).
- **Screen-mode corner cases** (`screenmode`) — real-bash-on-pty probes that the
  scripted-shell sessions POC couldn't reach: DECCKM mode-aware keys, screen-content
  watch, cooked read-until-sentinel + exit code, `^C` via pty, tmux durability.
- **tmux translation layer** (`tmuxlayer`) — tmux as a terminal
  translation/encapsulation layer (emulation offload, mode-correct keys, control-mode
  framing) and as a host-resident session resource the agent attaches/detaches from
  as a token-cost decision.
- **Expect-on-TTY** (`expect-on-tty`) — one streamed-pattern engine in two stances:
  `expect_run` (DRIVE — a GNU-Expect `{expect, send}` conversation) and
  `console_match` (TRIGGER — a session-scoped streamed predicate watching a held-open
  TTY for N named patterns).

## Event reactor & push (layer 2)

- **Event reactor** (`asyncmodel`) — many concurrent producers feed one unified
  event queue; the agent consumes it via PULL (`wait_for_event` / `poll_events`) and
  (where supported) PUSH. Establishes that standard MCP notifications are swallowed by
  Claude Code.
- **SELECT over watches** (`watchselect`) — first-class cancellable watches with
  per-watch readiness and `wait_for_watch` that blocks on a *subset* of watches,
  naming which fired; edge-vs-level triggering.
- **Channel push server** (`pushtest`) — a Go Claude Code *channel* server proving
  the one true server→agent push path (`notifications/claude/channel`, arriving as
  `<channel>` tags), as opposed to swallowed standard MCP notifications.
- **Stop-hook event pump** (`stophook`) — the headless-safe push: a Stop hook drains
  a pending-event queue and `decision:block` re-injects events so the agent keeps
  reacting, with no channels and no human in the loop.

## Automation engine (layer 3)

- **Watches** (`watches`) — *a watch is the agent's own tool call, triggered by a
  predicate instead of by the agent.* Records tool-call chains and fires them
  server-side when a condition holds (edge|level trigger, once|recurring, bounds),
  surviving disconnection and journaled for replay. No DSL.

## Observability & discovery

- **Log follow** (`logfollow`) — `log.follow` composed from tail-`F` + reactor +
  artifact store: follow a rolling log across rotation, emit match-with-context
  events for named patterns (rate-limited, loss markers), firehose kept server-side.
- **Passive HTTP discovery** (`httpdetect`) — discovers HTTP/HTTPS services without
  port-scanning by reading the kernel's listening sockets and fingerprinting (TLS
  certs + SAN vhost discovery, HTTP version, framework/WebSocket/SSE signals,
  health/openapi probes).
- **Service watch** (`servicewatch`) — listening-socket inventory transitions as
  reactor events: `service_up` / `service_down` + diff-derived `cert_rotated`,
  `vhost_added/removed`, `http_fingerprint_drifted`, `expiry_imminent`.
- **Endpoint watch** (`endpointwatch`) — HTTP endpoints as event sources in poll
  (drift detection + edge-triggered health), SSE, and WebSocket modes.
- **systemd integration** (`systemd`) — typed unit operations (Status/List/Start/
  Stop/Restart/…) returning stable structs, plus watchers that feed the reactor
  (`unit_started` / `unit_stopped` / `unit_failed`, `journal_entry`).
- **git as a typed tool surface** (`gitops`) — git working-tree operations returning
  structured Go shapes (typed predicates like `Status.Clean`) via pure-Go go-git
  instead of parsing `--porcelain` / `--format` text.

## Guardrails & access control

- **Guardrails admission controller** (`guardrails`) — open-by-default, opt-in CEL
  ACLs gating direct and autonomous (watch-triggered) `exec` **identically** through
  one gate wrapping the single dispatch point; allow / deny / confirm outcomes,
  journaled.
- **Resource-scope ACLs** (`scopeacl`) — the same admission gate extended to the
  structured tool families whose params name a resource (path, URL, host:port, bind
  address, unix socket): canonicalized resource variables, first-match-wins, default-
  allow with optional trailing default-deny, fail-closed.

## Credentials & auth

- **SSH auth flow mapping** (`sshauth`) — what MCP tool/task flow each SSH auth
  mechanism requires; the one-shot vs elicitation split, with keyboard-interactive
  fanning a single call into N elicitation round-trips (the MFA case).
- **Credentials exploration** (`credentials`) — 37 concept-proving shapes across 9
  themes for keeping secrets out of the LLM context while still usable in tool calls
  (resolver registry / opacity seam, generative credential kinds, a tiny SSH CA).
- **Cert-gated sudo, Go module** (`pamussh`) — real pam-ussh authorization against a
  live OpenSSH+PAM host: in-process CA mints a short-lived principal-scoped cert,
  agent-forwarded and verified by `pam_ussh.so` (built from source).
- **Cert-gated setuid sudo, C module** (`pamusshc`) — a ~360-LOC C PAM module that
  makes setuid `sudo` work through cert-gated auth (the Go `c-shared` module is inert
  under `AT_SECURE`), with correct cert verification (signature + principal + validity
  + fresh challenge round-trip).

## Configuration & host knowledge

- **Config provenance + live reload** (`config`) — `config.show` (effective merged
  config with per-key provenance and shadowed layers) and `config.reload`
  (non-disruptive: re-read/re-fingerprint, atomic swap, agent-readable diff, running
  work uninterrupted).
- **Host knowledge base / connect planner** (`hostkb`) — reframes "config" as host
  inventories that answer the agent's connect-planning questions; library-only,
  read-only modelling of reach/auth/inventory, ssh_config + known_hosts import/export,
  vantage-relative path planning, access pre-flight.

## Reference → dereference (token discipline)

- **Output references** (`outputref`) — any high-volume tool returns inline output
  only up to a budget; overflow is kept server-side as an `output_ref` the agent
  pages (`output_get`) or searches (`output_grep`) — every call budget-capped.
- **Schema references / typed RPC** (`rpc`) — the agent gets a short opaque
  `schema_ref`; the server holds the JSON Schema, wire client, and auth.
  `rpc_call(schema_ref, method, input)` validates input → dispatches → validates
  output, keeping large schemas out of context.
