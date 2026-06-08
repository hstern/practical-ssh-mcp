# practical-ssh-mcp — Rationale

This document captures **the problems we are trying to solve** for the user/agent.
It is deliberately problem-first: no tools, no architecture, no technology choices.
Those come later, designed backwards from what is written here.

## Why this exists

Agents already drive SSH constantly — but only by shelling out to `ssh` / `scp` /
`rsync` through a generic Bash tool. It works, badly. This MCP exists to make SSH
**practical** for an agent: less effort, fewer tokens, more capability, and — the
real prize — work the *server* can carry out on the agent's behalf instead of the
agent babysitting it.

The thesis: this is **not "SSH tools for an agent."** It is an **SSH automation
engine the agent programs declaratively.** The agent states intent once; the
server does the boring, slow, stateful work — probe, reconnect, retry, capture,
watch, react — without the agent in the loop. That is where the effort and token
savings actually come from.

## The status quo and why it hurts

Today every remote action is `Bash → ssh user@host 'command'`, one shot, cold,
unstructured. The pain:

1. **No interactivity / long-running work.** Tailing a log, watching a deploy, a
   REPL, `top`, an installer — none of it fits "run once, get output, exit."
2. **No state across calls.** `cd`, environment, an open shell context — all lost
   between every command. Each call starts from scratch.
3. **Cold every time.** The full handshake — jump hosts, key decryption, auth —
   is paid *per command*. Slow, and pure ceremony in every transcript.
4. **Output blows up the context window.** A 100k-line log or a chatty command
   dumps everything into the conversation. No truncation, paging, or search.
5. **Can't reach remote-only services.** A database, internal API, or Unix socket
   visible only from the remote host requires hand-rolled `ssh -L`/`-D` plumbing.
6. **No real blast-radius control.** The only guardrail is the harness's
   per-command approval prompt; SSH itself enforces nothing about *what* or
   *where*.
7. **File transfer and editing are quoting hell.** Moving files and — worse —
   editing them in place via fragile `sed`/`awk`/`python` one-liners that are
   unreadable, unreviewable, easy to get wrong, and hard to undo.
8. **Fugly commands.** The agent hand-writes `ssh -o ... -J jump user@host 'bash
   -c "..."'` strings. Error-prone to author, expensive to read, ugly in every
   transcript.
9. **Token cost and speed.** Cross-cutting consequence of all of the above:
   boilerplate in, firehose out, handshake repeated, nothing reused.

## What we want the agent to be able to do

End-to-end workflows the system must serve (stated as goals, not mechanisms):

1. **Deploy & supervise.** Push the build, kick the deploy, walk away; be told
   when it finishes or breaks, and have the smoke check run automatically.
2. **Survive a reboot.** Trigger a reboot and have the box auto-reconnected and
   the interrupted work resumed when it returns — no babysitting.
3. **Reach an internal-only service.** Talk to a DB / admin API / Unix socket
   only the remote can see, and get structured results back.
4. **Triage a huge log.** Search and slice a 100k-line log without dragging it
   into context.
5. **Watch many hosts at once.** Tail / monitor a fleet and react to whichever
   one speaks first.
6. **React to inbound activity.** Something hits the remote (a webhook, a
   connection) and the agent responds to it.
7. **Drive a remote daemon.** Operate a remote Unix-socket service (e.g. Docker)
   without opening a shell.
8. **Keep trees in sync.** Reliably mirror a local and remote directory
   (rsync-grade: deltas, deletes, filters, dry-run).
9. **Debug with rolling logs.** Follow live logs (across rotation, across hosts),
   and be notified of *events* in them — matches with surrounding context,
   silence, floods — while the firehose stays out of context.
10. **Drive interactive terminal tools.** Navigate menus, installers, and other
    full-screen TUIs by perceiving the rendered screen and sending keys —
    impossible without a persistent connection.
11. **Edit remote files cleanly.** Make targeted, reviewable, reversible edits to
    remote files the same way edits are made locally — instead of fragile
    in-place `sed`/`awk`/`python`.

## The real value: the server acts on its own

The highest-leverage capability is **autonomy**: the agent registers a standing
intent — *when* this happens, *do* that — and the server runs the loop and only
surfaces the meaningful result. Notify-only ("just tell me when X") is supported,
but it does not, by itself, reduce agent effort. The win is the server probing,
reconnecting, retrying, capturing, and reacting **without the agent spending a
turn or a token on the waiting.**

## Priorities

- **Persistent, multiplexed connections are foundational.** Interactivity, TUIs,
  warm reuse, and reboot-survival all depend on them.
- **Interactive tool driving** (menus / installers) — the biggest single win.
- **Structured remote file editing** — high win; kills the fugly edit commands.
- **Rolling-log debugging** — frequent; must be excellent.
- **Live-dashboard reacting** — valuable, lowest priority; build last.

## Scope boundaries

- **Nothing beyond the MCP process's lifespan.** Runtime state — connections,
  sessions, watches, events, captured output — is in-memory and dies with the
  process. Only configuration persists on disk. No daemon, no cross-session
  durability.
- **Guardrails are open by default**, leaning on the agent harness's existing
  permission system (the status quo — no regression, no new friction).
  MCP-enforced restrictions are an opt-in tightening, most relevant to autonomous
  actions that skip the per-action human prompt.
- **stdio transport only.**
