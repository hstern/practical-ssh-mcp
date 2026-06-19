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

End-to-end jobs the system must serve (stated as goals, not mechanisms),
organized by the **minimum that has to be present on the remote host** to serve
them:

- **Tier 0 — sshd only.** A stock OpenSSH server and nothing else; the capability
  lives in the MCP server (the SSH client).
- **Tier 1 — COTS on the remote.** Relies on standard software almost always
  present on a real Linux server (coreutils, systemd, rsync, git, tmux, sudo).
- **Tier 2 — a helper we deploy to the remote.** Reserved for the few jobs stock
  sshd and a quick COTS call genuinely cannot serve.

A job can span tiers: a Tier-0 base case that becomes richer with COTS, or a
Tier-1 COTS dependency with a Tier-2 fallback when that COTS isn't there. The
numbers/letters are stable identifiers, not priority order.

### Tier 0 — needs only sshd

1. **Deploy & supervise.** Push the build, kick the deploy, walk away; be told
   when it finishes or breaks, and have the smoke check run automatically. *(Base
   case; richer when the deploy itself leans on Tier-1 systemd/rsync.)*
2. **Survive a reboot.** Trigger a reboot and have the box auto-reconnected and
   the interrupted work resumed when it returns — no babysitting.
3. **Reach an internal-only service.** Talk to a DB / admin API / Unix socket
   only the remote can see, and get structured results back.
4. **Triage a huge log.** Search and slice a 100k-line log without dragging it
   into context. *(Base case; faster when Tier-1 `grep`/`tail` can pre-filter
   remotely.)*
5. **Watch many hosts at once.** Tail / monitor a fleet and react to whichever
   one speaks first.
6. **React to inbound activity.** Something hits the remote (a webhook, a
   connection) and the agent responds to it.
7. **Drive a remote daemon.** Operate a remote Unix-socket service (e.g. Docker)
   without opening a shell.
10. **Drive interactive terminal tools.** Navigate menus, installers, and other
    full-screen TUIs by perceiving the rendered screen and sending keys —
    impossible without a persistent connection.
11. **Edit remote files cleanly.** Make targeted, reviewable, reversible edits to
    remote files the same way edits are made locally — instead of fragile
    in-place `sed`/`awk`/`python`.
12. **Plan a connection, and survey what the host can do.** Answer *"how do I reach
    this host from here, with what auth, and am I actually ready?"* without burning
    round-trips on trial-and-error — and, once on, survey the operating system and
    which COTS is installed, so we know which Tier-1 capabilities this host actually
    supports. These aren't SSH-protocol operations; they're read-only shell probes
    run over `exec` (hence Tier 0).
17. **Automate a scripted interactive exchange.** Drive a prompt-driven program —
    an installer, a password prompt, a REPL — by matching what it prints and
    sending scripted replies, or just watch a held-open TTY for known patterns.
18. **Talk to a remote API / RPC without carrying its schema.** Call a typed
    remote service (validate in → dispatch → validate out) while the large schema
    and wire client stay server-side, out of context.
A. **Use secrets without exposing them to context.** Reference a credential by
   handle; the server resolves and uses it. Passwords, keys, and tokens never
   enter the conversation or transcript. *(Cross-cutting.)*

### Tier 1 — needs COTS on the remote

8. **Keep trees in sync.** Reliably mirror a local and remote directory
   (rsync-grade: deltas, deletes, filters, dry-run). *(Needs remote `rsync`; see
   Tier 2 when it's absent.)*
9. **Debug with rolling logs.** Follow live logs (across rotation, across hosts),
   and be notified of *events* in them — matches with surrounding context,
   silence, floods — while the firehose stays out of context. *(Needs `tail -F` /
   coreutils.)*
13. **Discover what a host is running — without scanning.** Inventory the services
    and HTTP endpoints actually listening, and identify them (framework, certs,
    vhosts, health/OpenAPI). *(Needs `ss` / `/proc` reads.)*
14. **Be told when a host's service surface shifts.** A unit or listener up/down, a
    cert rotating or near expiry, a vhost appearing, a health flip. *(Poll via
    COTS; low-latency push is Tier 2.)*
15. **Manage remote services without a shell.** Start / stop / restart / inspect
    units and react to unit state (started, stopped, failed, journal lines).
    *(Needs systemd: `systemctl` / `journalctl`.)*
16. **Operate a remote git tree.** Status / branch / diff / commit against a remote
    checkout, returning real answers (*"is it clean?"*). *(Needs remote `git`.)*
19. **Park and resume interactive work at zero idle cost.** Leave a long-running
    interactive session running remotely, pay nothing while away, re-attach with a
    cheap screen snapshot when it matters. *(Needs `tmux` as a host-resident
    session.)*
C. **Confine remote work to least privilege.** Run an operation under a
   constrained user / capability set so a mistake — or a hostile command — cannot
   reach beyond its intended blast radius. *(Needs `sudo` / `setpriv` / `runuser`.
   Cross-cutting.)*

### Tier 2 — needs a helper we deploy to the remote

- **Delta-sync where `rsync` is absent** (variant of #8). The block-delta rolling
  checksum has to run on the remote; the helper stands in for the rsync server.
- **Low-latency push & disconnection-surviving autonomy** (variant of #5 / #9 /
  #14). inotify / journal / `/proc` transitions and log matches pushed without
  client polling, and watches that keep running after the MCP server disconnects.
  *(Note: this collides with the current Scope boundary below — flagged, not
  resolved.)*
- B. **Escalate privilege with short-lived, auditable authorization.** Get root for
  a specific task through a time- and principal-scoped grant rather than standing
  `sudo` or a shared secret. *(Needs a cert-aware PAM module on the remote.)*

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
