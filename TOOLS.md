# practical-ssh-mcp — Planned Tool Surface

**Status: provisional.** This is the functional tool surface designed backwards
from [`RATIONALE.md`](./RATIONALE.md), now organized by the **remote-footprint
tiers** that doc defines. Tool **names, exact parameters, and return shapes are not
final**, and no implementation/technology choices are committed here — this is "what
the agent can call, why, and what the remote host must have for it to work."
Arguments are filled in from what the `poc/` phase actually validated.

## The two components

1. **The MCP server** — agent-facing, and itself the SSH **client**. Every tool
   below is a call into this process; all the smarts live here.
2. **The remote hosts** — sorted by what each tool needs present on the box:
   - **Tier 0 — sshd only.** Works against a stock OpenSSH server, nothing else.
   - **Tier 1 — COTS on the remote.** Needs standard software (coreutils, systemd,
     rsync, git, tmux, sudo) invoked over `exec` / a subsystem.
   - **Tier 2 — a deployed helper.** Needs a binary we push to the remote; reserved
     for the few jobs sshd + a COTS call can't serve.

## Conceptual model

Everything rides three layers (all **client-side**, so the layers themselves are
Tier 0 — the *tier of a tool is set by what it must invoke on the remote*):

1. **Channel multiplexer** — one warm connection per host; every operation is a
   channel on it. Handshake + jump hops + key-decrypt are paid once and reused.
2. **Event reactor** — every channel and every watch feeds **one unified event
   queue** with a single total-order `seq`. The agent consumes it with
   `wait_for_event` / `poll_events`; every tool result carries `pending_events: N`.
   *(Proven: standard MCP notifications are swallowed by Claude Code — **PULL is the
   authoritative path**; push is an optional accelerator. See Events.)*
3. **Automation engine** — watches (`predicate → reaction`) run server-side using
   the same tool calls the agent has. *A watch is the agent's own tool call,
   triggered by a predicate instead of by the agent.*

## Cross-cutting conventions

- **Large output never enters context whole.** Any high-volume tool returns inline
  output only up to a budget; the overflow is an `output_ref` the agent pages/searches.
  Proven budgets: **~30** lines inline on produce, **≤200** lines per `output.get`,
  **≤50** matches per `output.grep` (with a true `total` reported).
- **Secrets never enter context.** Anywhere a tool needs auth it takes a
  **`credential_ref`** — `{kind, name, params?}`, never a raw secret. The server
  dereferences it into the handshake; there is no read-out tool. (See Credentials.)
- **`exec` can be confined.** Any `exec`-shaped call accepts an optional
  **`isolation`** spec (Tier 1 — needs `sudo`/`setpriv`/etc.); `require:true` fails
  closed. (See Tier 1 › Least privilege.)
- **One admission gate, open by default.** Direct *and* watch-triggered calls pass
  through the same CEL gate with an unspoofable `origin` (`agent` | `watch:<id>`);
  outcomes are `allow` / `deny` / `confirm`, all journaled. Off unless configured.
- **`host`** is a configured host profile (preferred) or an ad-hoc target.
- **Events drain, they don't filter in place.** Filtering client-side after a drain
  loses non-matching events; use `wait_for_watch` (subset select) or a server-side
  filter, not drain-then-filter.

---

# Tier 0 — needs only sshd

## Connections — warm, multiplexed

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `connect` | Open/warm a connection (jump hops + auth paid once); persistent ones multiplex. Often implicit — naming a host reuses or creates one. | `host`, `addr?`, `user?`, `jump?: [{name,addr,user}]`, `credential_ref?`, `persistent?` | `{connection_id, via, reused, healthy}` |
| `disconnect` | Close a connection and its channels. | `connection_id` | ok |
| `connections` | List live connections + health. | — | `[{id, host, via, channels, peak_channels, healthy, age}]` |

Emits: `connection_down` / `connection_up` (keepalive doubles as liveness probe).
Jump chains are handshakes over forwarded channels; one warm client multiplexes N
channels.

## Exec — one-shot commands

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `exec` | Run a single command. `wait:true` blocks (bounded) and returns the result; `wait:false` returns a handle and emits an `exit` event. | `host`, `command`, `wait?`, `timeout?`, `isolation?` *(Tier 1)*, output limits | `{stdout, stderr, exit_code, truncated, output_ref?, duration}` |

## Connect-planning & survey (#12)

Read-only; answers "how do I reach X, am I ready, what can it do?" before/while
acting. Planning needs nothing remote; survey runs read-only shell probes over `exec`.

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `host.plan` | Vantage-relative connect plan from federated inventories (ssh_config / known_hosts / cloud fabrics). Names the steps and whether you're ready — without connecting. | `host_ref: {name, vantage}` | `{status: ok\|unreachable\|not_ready, steps:[{kind, target?, readiness_blocker?}], auth:{type,user?,key_ref?,cert_ref?}, readiness:{verdict, remedy?}}` |
| `survey` | Run the fixed, read-only capability allowlist (OS, arch, which COTS is present → which Tier-1 tools this host supports). Cached with TTL + invalidate. | `host`, `facts?: [name]`, `refresh?` | `{facts: {name: value}, cached, age}` |

Step `kind`s: `join-fabric` / `jump` / `connect` / `survey` (mild, auto-run) vs
`enrol` / `mint` (hot — gated, returns a consent request). Survey is **safe by
construction**: named probes only, no raw-command path.

## Sessions — interactive & stateful (PTY)

A persistent PTY whose `cwd`, env, and process survive across calls, with a
client-side terminal emulator in **three modes**.

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `session.open` | Start a persistent PTY session. | `host`, `cols?`, `rows?`, `echo?` | `{session_id}` |
| `session.send` | Send input — command, keystrokes, or signal. | `session_id`, `input` | new output (stream mode) |
| `session.read` | Non-blocking drain since last cursor. | `session_id` | `{data, mode: stream\|screen}` |
| `session.snapshot` | **Screen mode:** rendered terminal as text + cursor (TUIs). `diff:true` returns only changed rows + `rev`. | `session_id`, `diff?`, `styles?` | `{screen[], cursor_x, cursor_y, cursor_visible, settled_ms, rev, styles?}` |
| `session.keys` | Named keys for TUI nav — **DECCKM-aware** (arrows switch `ESC[`↔`ESC O` per the app's cursor-key mode). | `session_id`, `keys: [Up\|Down\|Enter\|F5\|Ctrl-C\|…\|"literal"]` | ok |
| `session.run` | **Cooked mode:** run a command, return clean stdout + real exit code (strips bracketed-paste/CRLF); env/cwd persist. | `session_id`, `command`, `timeout?` | `{stdout, exit_code}` |
| `session.resize` | Set PTY size. | `session_id`, `cols`, `rows` | ok |
| `session.close` | End the session. | `session_id` | ok |

Mode auto-switches on the alternate-screen escape (`ESC[?1049h/l`). Echo must be on
for observability. `Ctrl-C` is `0x03` through the PTY (reliable; the SSH `signal`
request is not). Emits: `output`, `screen_settled`, `exit`.
*(For detach/reattach across calls — parking a session — see Tier 1 › Park & resume.)*

## Expect — scripted interactive (#17)

One streamed-pattern engine, two stances; both are Tier 0 (PTY only).

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `expect.run` | **DRIVE** a conversation: alternating expect/send steps with per-step timeouts; captures regex groups. | `host`, `steps: [{expect: regex, send, timeout?}]`, `timeout?` | `{ok, error?, steps:[{expect, match[], output}]}` |
| `console.match` | **TRIGGER:** watch a held-open TTY for N named patterns; returns which fired + captures. Session-scoped (owns the live PTY); supports a `prime` for consoles that won't speak until poked. | `session_id\|host`, `patterns: [{name, regex}]`, `prime?`, `timeout?` | `{matched, name?, capture[], buffer, timed_out}` |

For DRIVE a timeout is failure (`ok:false`); for TRIGGER it's "not yet"
(`matched:false, timed_out:true`, no error).

## Files — structured remote editing (#11)

Over the SFTP subsystem (ships with OpenSSH). Edits are **atomic by default** (temp
sibling + posix-rename), auto-falling back to in-place writes on virtual filesystems
(`/proc`, `/sys`, `/dev`); the result reports `atomic` + a `reason` when it fell back.

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `file.read` | Read a remote file (or range). | `host`, `path`, `max_bytes?` | `{content, truncated, output_ref?}` |
| `file.edit` | Structured edits: `str_replace` (unique, or `all:true`), `append`, `prepend`, line insert/delete/replace, regex replace, or apply a unified diff. `dry_run` returns the would-be content. | `host`, `path`, `edits:[{op, …}]`, `dry_run?`, `backup?` | `{applied, new_content?, atomic, reason?}` |
| `file.write` | Create/overwrite a file. | `host`, `path`, `content`, `mode?`, `backup?` | `{atomic, bytes, reason?}` |
| `file.stat` | Existence / size / mode / mtime / kind. | `host`, `path` | `{exists, size?, mode?, kind?, mtime?}` |
| `file.list` / `file.mkdir` / `file.rename` / `file.remove` / `file.chmod` | SFTP basics (`rename` is atomic posix-rename). | `host`, `path` (+ `to`, `mode`) | ok / entries |

`str_replace` on a non-unique match errors unless `all:true`.

## Channels & forwarding — general-purpose plumbing (#3, #6, #7)

Primitives, not prescriptive — the workflow composes them. All Tier 0 (SSH protocol).

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `forward.tcp` | Reach a remote-visible `host:port` via a local endpoint (`-L`). | `host`, `remote_addr`, `local_bind?` | `{id, bound}` |
| `forward.unix` | Reach a **remote Unix socket** (docker.sock, a DB socket) via a local endpoint. | `host`, `remote_socket`, `local_bind?` | `{id, bound}` |
| `reverse.tcp` | Remote listens; each inbound connection becomes an **`accept` event** (`-R`). | `host`, `remote_bind`, `target?` | `{id, bound}` |
| `reverse.unix` | As above for a remote Unix socket. | `host`, `remote_socket`, `target?` | `{id, bound}` |
| `proxy.socks` | Local **SOCKS5** proxy dialing dynamically through the remote (`-D`). | `host`, `local_bind?` | `{id, endpoint}` |
| `http.fetch` | HTTP(S) request **dialed through the connection** (or at a forwarded `unix_socket`); structured result. | `host`, `method`, `url`, `headers?`, `body?`, `unix_socket?` | `{status, headers, body\|output_ref}` |
| `agent.forward` | Expose the local SSH agent to the remote session (needed for cert-gated flows). | `host`, `socket?` | `{forwarded}` |
| `http.serve` | Agent-hosted HTTP endpoint, optionally exposed on the remote via a reverse channel; inbound requests become events. *(Most advanced; built last.)* | `bind`, `expose_on?` | `{id, bound}` |
| `forward.list` / `forward.close` | Manage active forwards/listeners/proxies. | `id?` | list / ok |

## Transfer — whole-file (#1)

Whole-file SFTP is Tier 0; **delta `sync` is Tier 1** (needs remote `rsync`) — see below.

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `transfer.upload` | Send a file (local path **or inline content**) to the remote. | `host`, `remote_path`, `local_path\|content` | `{transfer_id}`; big ⇒ progress events |
| `transfer.download` | Fetch a remote file (or inline if small). | `host`, `remote_path`, `local_path?` | `{transfer_id}` / content / `output_ref` |
| `transfer.list` | Active transfers. | — | `[{id, kind, src, dest, bytes_done, bytes_total, done, error?}]` |

Emits `transfer_progress` (rate-limited ~50ms) / `transfer_done` / `transfer_error`.

## RPC — typed remote service without its schema (#18)

The agent gets an opaque `schema_ref`; the server holds the JSON Schema, wire client,
and **auth** (bearer tokens never reach the agent). Dialed through the connection.

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `rpc.methods` | List callable methods (names + descriptions, **never schemas**). | `schema_ref` | `{methods:[{name, description}]}` |
| `rpc.call` | Validate input → dispatch → validate output. | `schema_ref`, `method`, `input` | `{output?, validated, error?}` |

Validation errors are tagged by side (`input` vs `output`) so the agent knows whether
to retry corrected input or report server misbehavior; unknown ref/method lists the
catalog inline.

## Output artifacts

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `output.get` | Page a server-side `output_ref` (≤200 lines/call). | `output_ref`, `head?\|tail?\|from?\|to?` | `{lines[], from, to, total_lines, truncated}` |
| `output.grep` | Search within an `output_ref` (≤50 matches/call). | `output_ref`, `pattern`, `before?`, `after?` | `{matches:[{line_no, text, before?, after?}], total_matches, truncated}` |
| `output.info` | Size/lines/created of an artifact. | `output_ref` | `{lines, bytes, created}` |

## Events — the spine

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `wait_for_event` | **Blocking** long-poll: next event from any source (or timeout — not an error, re-ask). One outstanding call, zero polling cost. | `timeout?`, `filter?` | `{events:[{seq, source, kind, …}], timed_out, pending_events}` |
| `poll_events` | **Non-blocking** drain of everything queued. | — | `{events[], pending_events}` |

Every other tool result includes `pending_events: N`. Event `kind`s in use: `match`,
`output`, `exit`, `accept`, `transfer_progress`, `transfer_done`, `connection_down`,
`connection_up`, `watch_fired`, `screen_settled`, `silence`.

## Watches — autonomous automation

The automation engine. The reaction is a **flat chain of the same tool calls**, no
DSL. The engine is Tier 0; individual predicates may pull in Tier 1 (`log_match`,
`fs_watch` need `tail`/`inotifywait`) or Tier 2 (low-latency push).

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `watch.create` | Register `predicate → reaction`. Notify-only = empty reaction. | `host`, `predicate`, `reaction?: [step]`, `trigger: edge\|level`, `fire: once\|recurring`, `bounds?`, `timeout?` | `{watch_id}` |
| `watch.list` / `watch.get` / `watch.cancel` | Inspect / cancel; cancel kills any in-flight reaction. | `watch_id?` | list / detail / ok |
| `wait_for_watch` | **SELECT:** block on a *subset* of watches, naming which fired (clears only those). | `watch_ids?`, `timeout?` | `{firings:[{watch_id, target, state}], pending_watches}` |
| `journal` | What did my watches/reactions do (esp. while away)? | `since?` | `[{watch_fired\|reaction_started\|reaction_step\|reaction_done\|needs_attention, …}]` |

- **Predicates:** `reachable`, `port_open`, `ssh_auth_ok`, `file_exists`,
  `mtime_changed`, `value_threshold`, `exec(command, check: ==\|matches\|exit_code)`,
  `log_match(path, regex)` *(T1)*, `silence(duration)`, `fs_watch(path, events, regex?)`
  *(T1)*, and `cel(expr)` for composition (`&&`/`||`/`!` over host primitives).
- **Reaction step:** `{do: exec\|notify\|write\|sleep\|tui\|wait, host?, when?: cel,
  expect?: cel, until?: {cel, max_retries}, on_fail: abort\|continue}`. `wait` blocks
  the chain on a predicate (cross-host orchestration); `exec` is the compute escape hatch.
- **Bounds (opt-in):** `rate_limit {max_fires, per_ms}`, `circuit_breaker
  {trips_after, within_ms}` (disables + emits `needs_attention`), `max_runtime`,
  `max_fires_total`. Suppressed firings are *counted* (`suppressed: N` loss marker).
- **`trigger`:** `level` fires at creation if already true; `edge` only on transition.
- Explicit watches **survive disconnection** — they target a host profile, not a live
  channel ("tell me when the host is back up" works through a reboot).

## Credentials & auth (cross-cutting; A)

No standalone credential tool — auth rides `connect` and any tool needing a secret,
via a `credential_ref`. The secret is resolved server-side; **the grep-invariant holds
(no secret in the transcript)**.

- **`credential_ref: {kind, name, params?}`**, `kind` ∈ `stored` (static),
  `fetch` (exec `vault`/`op`/`pass`/`step` at use), `mint` (external CA or in-process
  CA → short-lived cert), `generate` (TOTP/HOTP/derived).
- **Auth shape on `connect`:** one-shot for `password` / `publickey` / `certificate` /
  `agent` / `gssapi` / `hostbased`; **`keyboard-interactive` fans one call into N
  elicitation round-trips** (the MFA case — `Password:` then `Verification code:`).
- Headless keystore: kernel keyring + tmpfs work; OS Secret Service does **not**.

## Config

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `config.show` | Effective merged config with **per-key provenance** (`source` + shadowed layers) and host identity fingerprints. | — | `{generation, settings:{key:{value, source, shadowed[]}}, hosts}` |
| `config.reload` | Re-read **non-disruptively** (build-validate-atomic-swap; running work uninterrupted; also via SIGHUP). Bad config is rejected, generation rolls back. | `config_data?` | `{generation, diff:{settings_changed, hosts_added/removed/updated}, error?}` |

---

# Tier 1 — needs COTS on the remote

## Transfer › sync — rsync delta (#8)

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `transfer.sync` | **rsync-grade** directory mirror over the warm SSH connection (pure-Go gokrazy/rsync client). Needs remote `rsync` (execs `rsync --server`). | `host`, `local`, `remote`, `direction: push\|pull`, `delete?`, `dry_run?`, `include?[]`, `exclude?[]` | `{ok, direction, dry_run, bytes_written?, files_size?, stderr?}` |

Filters are wildcard patterns (no leading `-`). When the box has no `rsync`, see Tier 2.

## Rolling logs (#9)

Needs `tail -F` (coreutils); follows across rotation. Firehose stays server-side as an
`output_ref`; only matches (with context, rate-limited) enter the event queue.

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `log.follow` | Follow a log, emit **match-with-context** events for named patterns. | `host`, `path`, `patterns: [{name, regex, before?, after?, max_fires?, per_ms?}]` | `{follow_id, output_ref}` |
| `log.unfollow` / `follow.list` | Stop / list. | `follow_id?` | ok / list |

Match event: `{pattern, line, line_no, before[], after[], output_ref, suppressed}`.
Per-pattern rate limits (one noisy pattern doesn't starve rare ones); quiet-log flush
emits a short after-context rather than blocking.

## Service & endpoint discovery (#13, #14)

Passive — reads the kernel's listening sockets (`ss` / `/proc/net`), never port-scans;
fingerprints client-side. Transitions surface as reactor events.

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `http.detect` | Inventory listening HTTP(S) services + fingerprint (TLS leaf SHA256, SAN vhosts, HTTP version, framework/WS/SSE, health/openapi), with PID/cwd attribution. | `host`, `depth: minimal\|standard\|aggressive`, `vhost_hints?[]` | `{services:[{port, pid?, process?, cwd?, tls?, vhosts[], http_version, frameworks[], health?, openapi?}]}` |
| `service.watch` | Watch the listening-socket inventory; emit transitions. | `host`, `poll_interval?`, `expiry_threshold?` | `{watch_id}` → events |
| `endpoint.watch` | Watch one HTTP endpoint in `poll` (drift + edge-health), `sse`, or `ws` mode. | `host`, `url`, `mode: poll\|sse\|ws`, `interval?`, `redact_keys?[]`, `unhealthy_after?` | `{watch_id}` → events |

Service events: `service_up` / `service_down`, `cert_rotated`, `vhost_added` /
`vhost_removed`, `expiry_imminent` (edge), `http_fingerprint_drifted`.
Endpoint events: `endpoint_drifted`, `endpoint_healthy` / `endpoint_unhealthy` (edge),
`sse_event`, `ws_message`, `connection_lost` / `connection_restored`.
*(TLS leaf SHA256 is the stable identity for rotation detection.)*

## Service management (#15)

Needs systemd (`systemctl` via D-Bus, `journalctl --output=json --follow` over `exec`).
Returns stable typed structs; transitions feed the reactor. *(The `svcmgr-research`
note maps this same surface onto launchd/rc.d via shell-out, polling-only, for later.)*

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `systemd.status` | One unit's state. | `host`, `scope: system\|user`, `unit` | `{name, description, load_state, active_state, sub_state, unit_file_state, main_pid?, memory?, tasks?}` |
| `systemd.list` | Units matching globs. | `host`, `scope`, `patterns?[]` | `{units:[UnitStatus]}` |
| `systemd.start` / `.stop` / `.restart` / `.reload` / `.enable` / `.disable` | Lifecycle ops. | `host`, `scope`, `unit` | `{ok}` / error |
| `systemd.watch` | Emit `unit_started` / `unit_stopped` / `unit_failed` on state change. | `host`, `scope`, `patterns[]` | `{watch_id}` → events |
| `systemd.journal` | Follow a unit's journal; emit `journal_entry`. | `host`, `scope`, `unit` | `{watch_id}` → events |

`active_state`/`load_state`/`sub_state` are systemd's own strings (an agent predicate
`active_state == "active"` reads identically to `systemctl is-active`).

## Remote git (#16)

Pure-Go `go-git` against an on-disk `.git` worktree; returns typed shapes, no porcelain
parsing. *(Operates where the repo is — i.e. needs the remote checkout; remote auth for
push not yet wired.)*

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `git.status` | Branch + cleanliness + per-file index/worktree codes. | `repo_ref` | `{branch, clean, files:[{path, index, worktree}]}` |
| `git.log` / `git.branches` / `git.diff` | History / branches / diff. | `repo_ref`, `limit?` / `from`, `to` | commits / branches / `{files:[{path, op, lines_added, lines_removed, patch?}]}` |
| `git.create_branch` / `git.checkout` / `git.add` / `git.commit` | Mutations (each is one explicit intent — no auto-stash, no auto-checkout). | `repo_ref` (+ `name`, `start_point`, `branch`, `paths?`, `subject`, `body?`) | typed result |

`status.clean` and the subject/body split are precomputed server-side; status codes
are stable enum strings. `repo_ref` is registered; an unknown ref lists the known ones.

## Park & resume interactive work (#19)

Needs `tmux` as a host-resident session: deliberate **detach = zero idle cost**,
**re-attach = O(screen)** snapshot. tmux also serves as a terminal *translation layer*
(its `capture-pane`/`send-keys` offload our vt emulation and the DECCKM key-encoding).
Auto-detected; falls back to the in-process Tier-0 emulator when absent.

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `session.detach` | Leave a session running on the host; stop paying for it. | `session_id` | ok |
| `session.attach` | Re-attach (returns a current-screen snapshot). | `session_id\|name` | `{screen, …}` |
| `session.parked` | List parked sessions (prefix-scoped). | `prefix?` | `[{name, created, clients}]` |
| `session.reap` | Bounded GC of parked sessions beyond a cap (prefix-scoped; never touches unrelated sessions). | `prefix`, `max_count` | ok |

## Least privilege — `isolation` on `exec` (cross-cutting; C)

Needs `sudo` / `setpriv` / `runuser` (or `bwrap` / `firejail` / `systemd-run` /
`prlimit` / `ulimit`). Attaches to any `exec`; picks the best mechanism available.

- **`isolation: {mechanism: auto|…, run_as?, groups?, drop_caps?, no_new_privs?,
  rlimits?: {as_kb, nproc, fsize_kb}, read_only?, rw_paths?, require?}`**
- `auto` prefers richest + least-setuid: `systemd-run > bwrap > setpriv > firejail >
  runuser > sudo > prlimit > ulimit`. `require:true` ⇒ fail closed (no unwrapped run).

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `isolation.probe` | Which mechanisms exist on the host. | `host` | `{available: {setpriv, sudo, bwrap, …}}` |
| `isolation.verify` | Confirm a spec actually drops (uid/gid, no_new_privs, CapEff). | `host`, `isolation` | `{uid, gid, no_new_privs, cap_eff}` |
| `isolation.set` | Set a repeatable default spec for a host. | `host`, `isolation` | ok |

**Honest limit:** per-exec wrap is not a cage if the agent is root. Real confinement
needs (1) dropping once at login into an unprivileged session and (2) the guardrails
gate denying re-escalation (`sudo`/`su`/`pkexec`/`doas`). The gate's resource-scoped
ACLs canonicalize each tool's resource variables (paths, URLs, `host:port`, binds,
sockets) — local resources are MCP-authoritative; remote ones are client intent atop
the server's own `PermitOpen`/perms.

---

# Tier 2 — needs a deployed remote helper

Reserved for the few jobs sshd + a COTS call cannot serve. **Not yet designed** — listed
so the surface is complete. The push/autonomy item collides with the current Scope
boundary ("no daemon, nothing beyond the MCP process's lifespan") and is a strategic
decision, not a committed feature.

| Capability | Why a helper | Sketch |
|---|---|---|
| **Delta-sync without `rsync`** (variant of #8) | The block-delta rolling checksum must run on the remote; with no `rsync`, the helper is the server side. | `transfer.sync` transparently uses the helper when `survey` reports no `rsync`. |
| **Low-latency push & disconnection-surviving autonomy** (variant of #5/#9/#14) | inotify / journal / `/proc` transitions pushed without client polling; watches that keep running after the MCP server disconnects. | A resident agent feeding the reactor; conflicts with the no-daemon scope. |
| **Cert-gated privilege escalation** (B) | Root for a specific task via a short-lived, principal-scoped cert verified by a cert-aware PAM module — not standing `sudo`. | `connect`/escalate with `credential_ref{kind:mint}` + `agent.forward`; needs `pam_ussh` (a **C** module — the Go `c-shared` build is inert under setuid `sudo`) and explicit CA-trust verification (x/crypto's `CheckCert` does **not** check `IsUserAuthority`). |
