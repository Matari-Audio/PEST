# REAPER control transport

Research current on 2026-09-02. `reaper-mcp` was inspected at commit
[`b4c0487`](https://github.com/danishaft/reaper-mcp/tree/b4c0487010ff5d5033ce37484da6a56b9c6a8f27).

## Decision

Use a **local, per-scenario JSON file spool driven by a deferred Lua ReaScript**.
Allow exactly one in-flight request. The Rust runner owns the REAPER child
process, timeout clock, transcript, and shutdown deadline.

This is the smallest option that meets the requirements without another runtime
or a native extension. Lua 5.4 is embedded in REAPER, while Python requires a
separate installation; deferred Lua scripts are explicitly supported for
background work ([REAPER ReaScript overview](https://www.reaper.fm/sdk/reascript/reascript.php#reascript_req),
[`reaper.defer`](https://www.reaper.fm/sdk/reascript/reascripthelp.html#lua_defer)).
The stock API can create and enumerate spool directories
([`RecursiveCreateDirectory`](https://www.reaper.fm/sdk/reascript/reascripthelp.html#RecursiveCreateDirectory),
[`EnumerateFiles`](https://www.reaper.fm/sdk/reascript/reascripthelp.html#EnumerateFiles)),
and standard Lua supplies file I/O and rename
([Lua 5.4 `io.open`](https://www.lua.org/manual/5.4/manual.html#pdf-io.open),
[`os.rename`](https://www.lua.org/manual/5.4/manual.html#pdf-os.rename)).

`reaper-mcp` is direct proof of this shape: its runner publishes JSON through a
temporary file and rename, waits by request ID with a monotonic timeout, validates
the typed response, and distinguishes a stale heartbeat from a command timeout
([runner source](https://github.com/danishaft/reaper-mcp/blob/b4c0487010ff5d5033ce37484da6a56b9c6a8f27/src/reaper_mcp/bridge/file_bridge.py#L81-L151),
[validation and publication](https://github.com/danishaft/reaper-mcp/blob/b4c0487010ff5d5033ce37484da6a56b9c6a8f27/src/reaper_mcp/bridge/file_bridge.py#L209-L235),
[Lua poller](https://github.com/danishaft/reaper-mcp/blob/b4c0487010ff5d5033ce37484da6a56b9c6a8f27/lua/reaper_mcp_bridge.lua#L2397-L2432)).
Its live REAPER acceptance is currently Linux-only, so this is source-level
cross-platform evidence, not yet proof on macOS and Windows
([project status](https://github.com/danishaft/reaper-mcp/blob/b4c0487010ff5d5033ce37484da6a56b9c6a8f27/README.md#L117-L128)).

## Minimal protocol

Each isolated scenario gets a fresh local directory:

```text
bridge/
  heartbeat
  requests/000001-<id>.ready.json
  requests/000001-<id>.active.json
  responses/000001-<id>.json
```

1. The runner writes a uniquely named temporary file, closes it, then renames it
   to `.ready.json`. Request fields are `protocol`, `session`, `seq`, `id`,
   `command`, and typed `args`.
2. Lua accepts only the expected sequence, renames `ready` to `active`, validates
   the envelope, executes it, and atomically publishes one immutable response.
3. The runner accepts only a response with the same protocol, session, sequence,
   and ID. The response is a validated `ok/result` or `error` union, following
   the proven `reaper-mcp` shape
   ([models](https://github.com/danishaft/reaper-mcp/blob/b4c0487010ff5d5033ce37484da6a56b9c6a8f27/src/reaper_mcp/models/bridge.py#L29-L71)).
4. Files remain in the scenario artifacts. The Run Receipt records their exact
   contents and relative timing, making replay a fresh isolated scenario fed the
   same ordered envelopes, never a retry in the damaged process.

One in-flight request makes order explicit and avoids relying on filesystem
enumeration order. It also removes the need for a queue, locks, async jobs,
idempotency storage, or multi-client arbitration.

## Failure and atomicity contract

- Atomic publication means readers ignore temporary names and only observe a
  closed file renamed to a unique, previously absent final name on the same local
  filesystem. It prevents partial JSON from being treated as a message; it does
  **not** promise power-loss durability.
- The `ready`, `active`, and `response` filenames are durable phase evidence.
  The heartbeat is advisory liveness evidence; the owned child process and its
  exit status are authoritative.
- Command execution is **not transactional or exactly-once**. Any timeout or
  process exit after request publication but before a valid response makes a
  mutation's outcome unknown. PEST fails the scenario, captures the spool,
  heartbeat, child exit status, stdout/stderr, and crash artifacts, kills any
  remaining process tree, and never retries in that scenario. `reaper-mcp`
  already applies the same `outcome_uncertain` rule to published mutating
  requests ([source](https://github.com/danishaft/reaper-mcp/blob/b4c0487010ff5d5033ce37484da6a56b9c6a8f27/src/reaper_mcp/bridge/file_bridge.py#L101-L123)).
- `shutdown` is an ordinary final request. Lua publishes `accepted`, then on the
  next deferred callback invokes REAPER's native quit action; `reaper.atexit`
  writes a best-effort stopped marker
  ([`atexit`](https://www.reaper.fm/sdk/reascript/reascripthelp.html#lua_atexit),
  [`Main_OnCommand`](https://www.reaper.fm/sdk/reascript/reascripthelp.html#Main_OnCommand)).
  Success requires the owned REAPER process to exit cleanly before the deadline.
  Forced process-tree termination is deterministic cleanup but fails the
  scenario.

## Rejected alternatives

| Transport | Why not V1 |
| --- | --- |
| Local TCP | REAPER exposes TCP primitives to **EEL2**, not standard Lua ([official EEL networking docs](https://www.reaper.fm/sdk/reascript/reascripthelp.html#eel_tcp_listen)). Moving the bridge to EEL2 adds framing, partial-I/O, JSON parsing, port allocation, and authentication work for no testing benefit. |
| Unix socket / Windows named pipe | Two platform protocols and blocking-I/O edge cases; stock Lua offers no portable nonblocking pipe API. |
| `SetExtState` / shared memory | REAPER-side persistence, not an external typed request/response channel. |
| HTTP or helper process | Requires another server/runtime or makes Lua launch and supervise a helper. The file spool is already the helper-free local IPC. |

Do not copy `reaper-mcp` wholesale. Reuse the transport lessons and, if source is
copied, its [MIT notice](https://github.com/danishaft/reaper-mcp/blob/b4c0487010ff5d5033ce37484da6a56b9c6a8f27/LICENSE);
PEST only needs the stop-and-wait subset. Add a faster transport only if measured
file polling materially limits the suite.
