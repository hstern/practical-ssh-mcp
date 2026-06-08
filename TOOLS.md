# practical-ssh-mcp — Planned Tool Surface

**Status: provisional.** This is the functional tool surface designed backwards
from [`RATIONALE.md`](./RATIONALE.md). Names, exact parameters, and return shapes
are **not final** — and no implementation/technology choices are made here. This
is "what the agent can call and why," not "how it's built."

## Conceptual model

Everything rides three layers:

1. **Channel multiplexer** — one warm connection per host; every operation is a
   channel on it (session, exec, forward, reverse, socks, http). Handshake + jump
   hops + key-decrypt are paid once and reused.
2. **Event reactor** — every channel and every watch feeds **one unified event
   queue**. The agent consumes it with `wait_for_event` / `poll_events`, and every
   tool result carries a `pending_events: N` hint.
3. **Automation engine** — watches (`event → condition → reaction`) run
   server-side, autonomously, using the same primitives the agent has.

Cross-cutting conventions:

- **Large output never enters context whole.** Any tool that can produce a lot of
  text returns it inline only up to a budget; the overflow is kept server-side as
  an `output_ref` the agent can page/search.
- **`host`** refers to a configured host profile (preferred) or an ad-hoc target.
- **Guardrails are open by default**; opt-in ACLs gate direct and autonomous
  actions identically.

---

## Connections — warm, multiplexed

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `connect` | Open/warm a connection (jump hops + auth paid once); persistent ones multiplex. Often implicit — naming a host reuses or creates one. | `host`, `persistent?` | `connection_id`, status |
| `disconnect` | Close a connection and its channels. | `connection_id` | ok |
| `connections` | List live connections + health. | — | `[{id, host, channels, healthy, age}]` |

Emits events: connection **dropped** / **reconnected**.

## Exec — one-shot commands

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `exec` | Run a single command. `wait:true` blocks (bounded) and returns the result; `wait:false` returns a handle and emits an **exit** event. | `host`, `command`, `wait?`, `timeout?`, output limits | `{stdout, stderr, exit_code, truncated, output_ref?, duration}` |

## Sessions — interactive & stateful (PTY)

A persistent shell/PTY whose `cwd`, env, and process state survive across calls.
Carries a server-side terminal emulator with **two presentation modes**.

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `session.open` | Start a persistent PTY session. | `host`, `cols?`, `rows?` | `session_id` |
| `session.send` | Send input — a command, keystrokes, or a signal (Ctrl-C). | `session_id`, `input` | new output (stream mode) |
| `session.read` | Non-blocking drain of buffered output since last read, paged. | `session_id`, `max?`, `cursor?` | output chunk + cursor |
| `session.snapshot` | **Screen mode:** the rendered terminal as plain text + cursor (for TUIs). | `session_id` | `{screen, cursor, cols, rows}` |
| `session.keys` | Send named keys for TUI navigation (`Down`, `Enter`, `PageUp`, `F5`, `Ctrl-C`, literal text). | `session_id`, `keys[]` | ok |
| `session.resize` | Set PTY size (TUIs render to it). | `session_id`, `cols`, `rows` | ok |
| `session.close` | End the session. | `session_id` | ok |

Mode auto-switches: entering the alternate screen ⇒ screen mode; exiting ⇒ stream
mode. Emits events: **output available**, **screen settled** (no redraw for N ms),
**process exit**.

## Events — the spine

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `wait_for_event` | **Blocking** long-poll: return the next event from any source (or timeout). One outstanding call, zero polling cost. | `timeout?`, `filter?` | `{source, kind, ...payload}` |
| `poll_events` | **Non-blocking** drain: return everything queued and return immediately (use while busy with other work). | `filter?` | `[events]` |

Every other tool result includes `pending_events: N`. Event `kind`s include:
`match`, `output`, `exit`, `accept` (inbound connection), `transfer_progress`,
`transfer_done`, `connection_down/up`, `watch_fired`, `screen_settled`, `silence`.

## Watches — autonomous automation

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `watch.create` | Register `predicate → reaction`. Reaction runs server-side using any primitive in this surface; notify-only = empty reaction. | `predicate`, `reaction?`, `fire: once\|recurring`, `bounds {max_retries, max_runtime, rate_limit}`, `timeout?` | `watch_id` |
| `watch.list` | What am I currently watching? | — | `[{id, predicate, fire, state}]` |
| `watch.get` | Inspect one watch. | `watch_id` | watch detail |
| `watch.cancel` | Cancel a watch; kills any in-flight reaction. | `watch_id` | ok |
| `journal` | What did my watches/reactions do (esp. while I was away)? | `since?` | `[actions + results]` |

Predicates include: `reachable`, `port_open`, `ssh_auth_ok`, `exec(...)==…`,
`http_probe(...)==…`, `file_exists`, `mtime_changed`, `log_match(regex)`,
`silence(duration)`, `value_threshold`. Explicit watches **survive disconnection**
(they target a configured host, not a live channel) — e.g. "tell me when the host
is back up" works through a reboot.

## Channels & forwarding — general-purpose plumbing

These are primitives, **not prescriptive** — the agent's workflow decides how to
compose them.

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `forward.tcp` | Reach a remote-visible `host:port` via a local endpoint (`-L`). | `host`, `remote_addr`, `local_bind?` | local endpoint |
| `forward.unix` | Reach a **remote Unix socket** (e.g. `docker.sock`, a DB socket) via a local endpoint. | `host`, `remote_socket`, `local_bind?` | local endpoint |
| `reverse.tcp` | Remote listens; each inbound connection becomes an **`accept` event** (`-R`). | `host`, `remote_bind`, `target?` | listener id |
| `reverse.unix` | As above for a remote Unix socket. | `host`, `remote_socket`, `target?` | listener id |
| `proxy.socks` | Local **SOCKS5** proxy dialing dynamically through the remote (`-D`). Point any client at it. | `host`, `local_bind?` | proxy endpoint |
| `http.fetch` | Perform an HTTP(S) request **dialed through the connection** (as if from the remote); structured result. | `host`, `method`, `url`, `headers?`, `body?` | `{status, headers, body\|output_ref}` |
| `http.serve` | Stand up an agent-hosted HTTP endpoint, optionally exposed on the remote via a reverse channel; inbound requests become events. *(Most advanced; built last.)* | `bind`, `expose_on?` | endpoint id |
| `forward.list` / `forward.close` | Manage active forwards/listeners/proxies. | `id?` | list / ok |

## Transfer

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `transfer.upload` | Send a file (local path **or inline content**) to the remote. | `host`, `dest`, `src\|content` | result; big ⇒ background + progress events |
| `transfer.download` | Fetch a remote file to local (or inline if small). | `host`, `src`, `dest?` | result / content / `output_ref` |
| `transfer.sync` | **rsync** (must-have): directory trees, delta transfer, `--delete`, include/exclude filters, dry-run. Reuses the warm connection as transport. | `host`, `src`, `dest`, `direction`, `delete?`, `filters?`, `dry_run?` | summary + progress events |

## Files — structured remote editing

Kills fugly in-place `sed`/`awk`/`python`. Mirrors the local Edit tool. Edits are
**atomic** (temp + rename), with optional `.bak` backup and **diff preview**.

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `file.read` | Read a remote file (or range). | `host`, `path`, `range?` | content / `output_ref` |
| `file.edit` | Apply structured edits: exact `str_replace` (unique or all), insert/delete/replace line range, regex replace, append/prepend, or apply a unified diff. | `host`, `path`, `edits[]`, `backup?`, `dry_run?` | diff + applied? |
| `file.write` | Create/overwrite a file. | `host`, `path`, `content`, `backup?` | ok |
| `file.stat` | Existence / size / mtime / mode. | `host`, `path` | stat |

## Output artifacts

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `output.get` | Page through a server-side `output_ref` without pulling it whole. | `output_ref`, `range\|head\|tail` | slice |
| `output.grep` | Search within an `output_ref`. | `output_ref`, `pattern`, `context?` | matches + context |

## Config

| Tool | Purpose | Key inputs | Returns |
|---|---|---|---|
| `config.show` | Effective merged config (files + env + CLI + overrides). | — | config |
| `config.reload` | Re-read config **non-disruptively** (also triggerable via SIGHUP); re-reads keys/certs. | — | reload summary |

## Convenience — composed workflows

| Tool | Purpose | Composes |
|---|---|---|
| `log.follow` | Follow a rolling log (across rotation) and emit **match-with-context** events for named patterns; supports silence + flap rate-limit; firehose stays in `output_ref`. | session/`tail -F` + watches + output_ref |
