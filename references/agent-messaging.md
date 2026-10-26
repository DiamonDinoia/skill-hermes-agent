# Agent messaging: siblings talking, children asking the manager

Load when the work needs delegate children or spawned `hermes chat` agents to exchange messages, or when a child must pause on a manager answer mid-run and continue.

## The native routes (no sibling channel)

- Parent to live child: `delegate_task(action="steer")`.
- Child to parent: the final report, plus an `[ESCALATE-TO-MANAGER]` line; a child's `clarify` is denied in children (tools/delegate_tool_config.py auto-deny list), so a child cannot block on a user question mid-run.
- Sibling to sibling: the manager relays. Carry agent A's output into a steer of the live agent B, or into a fresh follow-up brief.

These are capability claims about a moving upstream. Pin a sha and re-read the source (`git show <sha>:<path>`) before designing around them.

## The MCP bus pattern

When manager relay costs too many rounds, a small stdio MCP server over one shared SQLite file (WAL mode, busy_timeout) gives every agent a mailbox:

- `send(name, to, body)`, `inbox(name, wait_s)` where the wait returns early when mail lands. `name` is the caller's own name, an explicit argument on every call (see the identity pitfall below).
- `ask_parent(name, question, timeout_s)` polls the DB between `asyncio.sleep` intervals until an answer row exists or the timeout passes; polls with `await`, never a blocking sleep, so the event loop stays live.
- The manager answers through a small CLI on the same DB (`pending`, `answer ID TEXT`), which keeps the reply outside MCP and inside the manager's own terminal. The CLI exits nonzero when an answer matches no open question (a successful-looking no-op reads as delivered in scripts), and prints a one-line error with exit 2 on bad numeric input.

Source anchors the design depends on (verify at a pinned sha before relying on them):

- `hermes chat` processes load `mcp_servers` from config: the `discover_mcp_tools()` call in hermes_cli/main.py.
- delegate children inherit the parent's MCP toolsets: tools/delegate_tool_toolsets.py (inherit default true in delegate_tool_config.py); DELEGATE_BLOCKED_TOOLS carries no MCP entries.
- The per-server MCP tool timeout is unclamped (tools/mcp_tool_common.py, default 300): set `timeout: 3600` on the bus server or a blocking `ask_parent` is killed at five minutes.
- MCP elicitation is auto-denied in delegate children (tools/delegate_tool_config.py): run this server with elicitation disabled, or an accidental elicitation in a child resolves as deny/cancel and the ask dies silently.
- A profile (`-p NAME`, which sets HERMES_HOME before config load in hermes_cli/main.py) overrides config.yaml: prototype on a scratch profile so the live config stays untouched.

## Rules that survived review (each exists because its absence failed a probe)

- Exactly-once delivery: `inbox` claims unread rows for the caller atomically (`UPDATE ... RETURNING` in one write) and marks them read. An unclaimed read delivers one message to every concurrent waiter with the same name and again on the next call. A `log` command still lists everything, and a non-positive `log N` must exit 2, because SQLite treats a negative LIMIT as unbounded and dumps the whole table.
- `UPDATE ... RETURNING` needs SQLite >= 3.35; check `sqlite3.sqlite_version` on the target runtime before shipping it.
- The ask/timeout race settles on one conditional write: on timeout the ask path expires its question with `UPDATE ... WHERE id=? AND answered_ts IS NULL`; when that changes 0 rows an answer already landed, so it re-reads the row and returns the answer. Then a CLI `answered id=N` always proves the asker received it, and an answer after expiry fails CLI-side.
- Run the schema (`CREATE TABLE IF NOT EXISTS` plus `ALTER TABLE` migrations) on every connection, not only at startup: a deleted DB file then recovers on the next call. Two connections migrating one old DB race on the ALTER; catch the duplicate-column error and continue. Matching a SQLite error by message text is fragile on localized builds, so prefer a pragma check where one exists.
- Clamp `wait_s`/`timeout_s` below the server's MCP tool timeout (with `timeout: 3600`, clamp to 3500): a wait past the tool timeout is killed mid-block and returns nothing.
- After replacing the server file, restart the gateway: the gateway-owned server process keeps the old code, and rows written under the old version (never marked read) are redelivered after the swap.
- Pin the runtime's dep versions in the server README (the built server needed `mcp>=1.26`; older releases rename the stdio module). Gateways load MCP modules before other imports, so sys.path resolution differs from a direct run; test through the gateway, not only by hand.

## Third-party servers

The agent-mail MCP servers surveyed were pull-only (fetch inbox); none offered a blocking ask-and-answer. Prefer adopting one that adds that over maintaining a local server, and re-survey before building since the area moves.

## Identity pitfall

The gateway owns one stdio server process per profile, shared by every session and delegate child in it, so env vars are profile-global: an env-based agent name (for example AGENTBUS_AGENT) resolves identically for all of them and cannot tell siblings apart. The fix is to pass the caller's own name as an explicit argument on every bus call (`send(name, to, body)`); each agent names itself at the call site and `send(to)` routing then works per agent. This replaces the earlier design where clean per-agent names needed one `hermes chat` child per name, each launched with its own env value. Answers still match questions by question id.

## Testing a bus

- Unit: two identities exchange mail; `inbox(wait_s)` returns early on a cross-process message; `ask_parent` blocks, a second process answers via the CLI, and the ask returns that text; the timeout path; two concurrent writers hit no `database is locked`. Drive the real server over stdio with the mcp client SDK in at least one test, not only by importing functions.
- Mutation: make `ask_parent` return immediately; the blocking test must fail; revert to green.
- End to end: two `hermes chat -Q` children on the bus exchange a message, and one blocks on `ask_parent` and continues on the manager's CLI answer within the same session. Negative control: the same scenario with the server removed from the profile must fail.
- Races settle on logic, not timing: drive the answer-vs-timeout window by calling the racy write's own code from a second connection at the point before the other side's write (serial-order controls on both sides), never by racing real sleeps. Timeout and clamp paths run on tiny timeouts or a fake clock, not wall clock.
- Mutation set, each must fail loud before revert: drop the read-marking claim (the exactly-once test), drop the conditional guard on the timeout write (the race-coherence test, both orders), make `ask_parent` return immediately (the blocking test).
