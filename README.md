# Codex Bridge

[![Mentioned in Awesome Codex CLI](https://awesome.re/mentioned-badge.svg)](https://github.com/RoggeOhta/awesome-codex-cli)

### Your live Claude Code and Codex CLI sessions talk to each other. Named conversations stay separate in one folder.

You run `claude` and `codex` in the same folder. Tell each pair to use the same bridge name. They exchange messages through `.codex-bridge/<name>/chat.md`. The original unnamed bridge still uses `.codex-bridge/chat.md`.

<p align="center"><img src="docs/screenshot.png" alt="Codex (right) opens with @claude; Claude (left) receives the message through the channel and answers with a comparison table" width="900"></p>

<p align="center"><img src="docs/architecture.svg" alt="Two terminals, one shared file: each side's Stop hook appends its reply to chat.md; Claude's channel and Codex's queued wake deliver the other side's message" width="800"></p>

## Install

Needs `node` on the PATH of a non-interactive shell.

**Claude Code**

```
/plugin marketplace add abhishekgahlot2/codex-claude-bridge
```
```
/plugin install codex-bridge@codex-claude-bridge
```

Then launch Claude with the channel enabled (Channels are a research preview; the flag is required):

```bash
claude --dangerously-load-development-channels plugin:codex-bridge@codex-claude-bridge
```

**Codex CLI**

```bash
codex plugin marketplace add abhishekgahlot2/codex-claude-bridge
codex plugin add codex-bridge@codex-claude-bridge
```

Run `codex`, open `/hooks`, trust its two hooks.

## Use

In a Codex pane:

```
B-a: redis vs memcached, keep going until you agree
```

In the matching Claude pane, say `B-a: ...` once to join. Codex opens with `@claude a: ...`, then returns idle. Claude's pane shows `← codex-bridge: ...` and its answer. That answer wakes the saved Codex session through `codex queue`; its prompt hook reads the reply. The two agents alternate until one ends a message with `[DONE]`.

During a longer Codex task, send a progress update without ending the task with `tell`:

```bash
node bridge.mjs tell --bridge a "P2.2 started; I am decoding the animation frames."
```

This appends a message as Codex, which Claude's channel can deliver while Codex continues. An addressed assistant reply such as `@claude a: ...` is still a turn-ending response.

From the Claude pane, start with `B-a: ...`. Codex must have joined `a` once; the bridge then wakes its saved session through `codex queue`. Use `b` for a second pair. The unnamed chat is `B-default:`.

**Position rule.** `B-<name>:` in a prompt and the plain `@claude:` / `@codex:` in a reply work only as the very first characters of the message. The named reply marker `@claude <name>:` / `@codex <name>:` may open any line: text before it is dropped and the peer receives what follows. Placed mid-sentence or inside a quote on a line that does not start with it, it is ignored silently and nothing reaches the peer.

### Codex to Codex

Codex sessions do not exchange messages through the Claude bridge. Send directly to a named Codex session with `codex queue`:

```bash
codex queue --thread corpus --message "Bonjour corpus, ici gamma. Est-ce que tu me reçois ?"
```

The target may be an exact session name or UUID. To send the reply back, the other session runs `codex queue` with this session's name or UUID as `--thread`.

`codex queue` writes to `~/.codex`. In the default `workspace-write` sandbox, that path is protected and the command can fail with `unable to open database file`. Keep `workspace-write`, and configure Approve for me so an eligible request can be approved automatically:

```toml
approval_policy = "on-request"
approvals_reviewer = "auto_review"
sandbox_mode = "workspace-write"
```

Restart both sessions after changing `~/.codex/config.toml`. This keeps the sandbox boundary; it does not grant full access.

Watch the transcript from anywhere:

```bash
tail -f .codex-bridge/a/chat.md
```

To send a note into one open named chat from the terminal:

```bash
node bridge.mjs say --bridge a "Please check the latest result"
```

Use `tell` from Codex when the message should appear as Codex and the current task should continue:

```bash
node bridge.mjs tell --bridge a "The audit is underway; I will send the result when it is done."
```

To give an idle Codex session a task from a sandbox that cannot run `codex queue`, start one relay in a normal terminal for the project:

```bash
node ~/codex-bridge/bridge.mjs relay --project ~
```

The relay watches all named bridges in that project and works independently of Claude sessions. Keep it running while you want automatic delivery. Then, from a Codex sandbox in the same project, use:

```bash
node bridge.mjs send --bridge a "Audit the report and reply briefly"
```

The relay passes the request to `codex queue` using the saved Codex session ID. A project-wide `flock` allows only one relay at a time; it atomically renames each request to `inflight-*.json` before sending. A failed send gets three attempts with 1 s and 3 s waits between them. Requests that still fail, or target a replaced Codex thread, move to that bridge's `failed/` folder with a reason, and the chat receives a warning. Claude Code does not need to be open for this path. For a one-time attempt, use `node bridge.mjs relay --once --project ~` from a normal terminal.

For automatic restart after a crash or terminal closure, install the included `codex-bridge-relay.service` as a user service. Check its `ExecStart` Node path before installing it. This is optional; it uses the same relay command and does not launch Claude Code.

To clear one named chat and its read positions while keeping the paired session IDs and pending requests:

```bash
node ~/codex-bridge/bridge.mjs clear --bridge a
```

Run this from the project folder (`~` in the example). A new addressed conversation also clears that chat automatically. Individual conversations stop after 40 messages.

Add `--queue` to also delete requests still waiting in `queue-*.json`. Requests already claimed as `inflight-*.json` and past failures in `failed/` are preserved.

## How it works

`bridge.mjs` does three jobs. Each named bridge has its own chat and state files:

1. **Prompt hook** (both tools). A prompt such as "B-a:" joins that session to `a` and tells the agent to use `@claude a:` only when writing to the other agent; replies to the user stay unprefixed. Each bridge keeps the session IDs of its current Claude and Codex members.
2. **Stop hook** (both tools). Forwards only replies explicitly addressed to the other agent with `@<agent> <bridge>:` (or `@<agent>:` for the default bridge). Replies to the user stay local, including after a bridge message. Codex immediately delivers any unread blocks as `{"decision":"block","reason":"<that message>"}`, or returns idle if none are waiting. It does not hold a turn open for future messages, including listen-only replies. Use `tell` from Codex during a task to post an update without ending its turn. Claude wakes the saved Codex session through `codex queue` when it sends a reply. Claude's Stop hook always returns immediately.
3. **Channel** (Claude only). A dependency-free MCP server the plugin starts inside Claude Code. It follows the bridge joined by its parent Claude process and pushes new blocks into that session. Without a live channel, Claude catches up on unread messages at its next prompt.

Each side keeps a cursor in `.codex-bridge/<side>.json`, so nothing is dropped while an agent is mid-turn and nothing is shown twice. Conversations cap at 40 messages. The folder is gitignored automatically.

## Diagnostics

Each active bridge writes delivery events to `.codex-bridge/<name>/events.jsonl` (or `.codex-bridge/events.jsonl` for the default bridge). The log records hook calls, unread counts and cursor positions, Codex queue outcomes, and message block headers. It omits prompt and message bodies. Wake IDs are local log fields and are never added to the queued prompt. At 1 MB, the current log rotates to `events.previous.jsonl`, replacing the older copy. Diagnostic write failures do not affect bridge delivery.

## File format

```
## codex @ 2026-09-08T14:15:01.957Z
Let's compare Redis and Memcached.

## claude @ 2026-09-08T14:16:30.079Z
Redis by default. [DONE]
```

| Marker | Meaning |
|--------|---------|
| `## <claude\|codex\|user\|bridge> @ <ISO-8601 UTC>` | Start of a message. `user` is you (`node bridge.mjs say "..."`), `bridge` is the cap notice. |
| `@claude` / `@codex` at the start of a reply | Open the bridge with this message. |
| `[WAITING]` at the start or end of a reply | Do not send this reply to the peer; listen only. This does not replace a needed response to the human user. |
| `[DONE]` at the start or end of a reply | End the peer exchange after this reply. The receiver can still respond to the human user. |

Messages arriving through the bridge are written by the peer agent, even though Claude receives them as user messages. Claude should answer the human user normally when a received message needs attention, rather than replying with bare `[WAITING]`.

## Limitations

- Idle delivery to Codex requires a working `codex queue` command and the correct saved thread ID. Without an enabled channel, Claude receives unread messages at its next prompt.
- Assistant replies are sent at the end of a turn. Codex can send mid-task progress updates with `tell`.
- Both agents must run in the same folder on the same machine.
- Markers (`B-<name>:`, `@claude`, `@codex`) are recognised only at the start of the message; a mid-message `@` is not forwarded.
- Each named chat is delivered only to its current pair of sessions. Join both sessions before expecting idle delivery.
- Claude needs the development-channels flag until custom channels leave preview.

## This vs v0.1

v0.1 was a blocking Codex MCP tool, a Claude channel, and an HTTP server with a web UI.

**Better now**
- No server, no port, no web UI. The markdown file is the transcript; `tail -f` is the viewer.
- Folder-scoped. Other projects and sessions are untouched.
- The Stop hook captures explicitly addressed replies; v0.1 lost replies when Claude omitted `reply_to`.
- Claude → Codex uses `codex queue` after the target Codex session joins the bridge.
- A cursor per side: nothing dropped while an agent is mid-turn, nothing shown twice.
- Codex returns idle between messages; Claude wakes its session when a reply arrives.

**Worse now**
- Two hooks to trust in Codex, and per-folder config without the plugins. v0.1 was one global config.
- Replies to received bridge messages must be explicitly addressed. An addressed reply also starts or resumes an exchange.
- Only the final message of a turn is sent.

**Same in both:** Claude needs the channels launch flag.

## Without the plugins

Clone the repo. Put the two hooks from `hooks/hooks.json` into `<folder>/.claude/settings.local.json` and `<folder>/.codex/hooks.json` with the clone path in place of `${CLAUDE_PLUGIN_ROOT}`, and launch Claude with `--dangerously-load-development-channels server:codex-bridge --mcp-config <a file declaring the codex-bridge server: node <clone>/bridge.mjs channel>`.

## Tests

```bash
node --test
```

## Working setup and what is set aside (2026-10-06)

What works, verified: Claude to Codex through `codex queue --thread <session name or UUID>`; Codex to Claude through `node bridge.mjs say --bridge <chat> '<text>'`, including from a session that is not a member of the chat (checked with a session named gamma); 48 tests. Say in each task which command to reply with, and sign the text with the worker's name: the channel labels every incoming message `sender="user"`.

Set aside on purpose, may come back:

- **Relay as a systemd service.** The `relay` command and a user unit exist but the service is not installed or tested. The relay only runs from a normal terminal, because `codex queue` fails inside the Codex sandbox (`~/.codex` is read-only there).
- **Codex to Codex.** Direct delivery is verified with `codex queue --thread <session name or UUID> --message "..."`. Each sender must be able to obtain an approved elevation for the command because `codex queue` writes to `~/.codex`; `workspace-write` alone protects that path. This path does not use the Claude bridge transcript.
- **Several Codex workers as chat members.** A chat has one Claude member and one Codex member. Extra workers are not members; they report with `say`.
- **Session identity.** `/clear` in Codex creates a new thread with a new UUID, which orphans the chat membership. Address sessions by name instead; rename a session with `/rename <name>` in the Codex TUI (verified).
- **`/clear` in Claude.** It also changes the session id, while the channel process keeps running. The channel therefore rereads `~/.claude/sessions/<parent pid>.json` on every tick instead of trusting `CLAUDE_CODE_SESSION_ID`, which stays frozen at launch. A channel started before this fix keeps the old behaviour until Claude restarts.
- **At-least-once delivery.** A relay that dies after sending but before deleting `inflight-*` resends the request on restart.
- **Silent channel switch.** A prompt naming another chat rebinds the session without any refusal, and the old chat keeps its stale `member.json`.

## Earlier design

[v0.1.0](https://github.com/abhishekgahlot2/codex-claude-bridge/releases/tag/v0.1.0) used a blocking Codex MCP tool, a local HTTP server, and a web UI. One-directional in practice.

## License

MIT
