---
name: lingtai
description: Interact with LingTai agents through the shared human mailbox — locally or on remote machines via SSH. Read and send mail, discover agents, check liveness, manage agent lifecycle (sleep/suspend/cpr/refresh), monitor remote networks, and set up adaptive mail polling. Use this when the user asks about their agents, wants to check mail, or manage the agent network.
version: 0.4.0
---

# LingTai — Agent Network Integration

You are connected to a LingTai agent network. You share the human's identity and mailbox. This skill teaches you how to interact with the network using your native file tools.

**This skill is opt-in.** Only activate when the user explicitly asks to interact with the LingTai network (send mail, check inbox, manage agents). Do not proactively read or summarize mail unless asked.

## Trust but verify

Agents report their own progress, and their reports can drift from reality — avatars may report success without writing files, multi-copy scaffolding can leave a stale tree untouched, orchestrators may forward a plan-of-action as if already executed. When an agent claims "done" on work with real stakes (files created, translations completed, exports validated), spot-check at least one load-bearing claim with `ls`/`grep`/`find` before relaying the status to the user. A 5-second check catches false positives that would otherwise ship.

This is not about distrust — it's about the gap between "task dispatched" and "file on disk." The tighter the loop, the more the network learns. When you catch a false-positive, report it back to the agent with the evidence so it can fix both the artifact and its verification habit.

## Your Identity

You are the human. Your directory is `.lingtai/human/`. Your mailbox is `.lingtai/human/mailbox/`. You do not have a separate agent identity — you are another interface the human uses to interact with their agents, like checking email from a different device.

The human is a **pseudo-agent**: `admin: null`, no running process, no heartbeat. Real agents pick up your outgoing mail by polling your outbox (see "Sending Mail" below).

When you send mail, add a `"via": "<your-host>"` field to the identity block so messages can be attributed to which interface sent them (e.g. `"via": "claude-code"`, `"via": "codex"`, etc.). Use a short stable identifier for the tool you are running inside.

## Reading Mail

Scan for messages at this path:

```
.lingtai/human/mailbox/inbox/*/message.json
```

Use whatever file-search tool you have (glob, find, ls). Each `message.json` contains:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID |
| `from` | string | Sender address (e.g. "orchestrator") |
| `to` | string or array | Recipient address(es) |
| `subject` | string | Subject line |
| `message` | string | Body text |
| `received_at` | string | RFC3339 timestamp |
| `in_reply_to` | string | (optional) UUID of the message being replied to |
| `identity` | object | Sender's manifest snapshot |

Sort by `received_at` (RFC3339 strings sort lexicographically). Present a summary to the user: sender, subject, time, and first line of the message.

Sent mail is at `.lingtai/human/mailbox/sent/*/message.json`.

### Inbox summary snippet

Use this to quickly list recent messages:

```python
import json, glob
msgs = []
for p in glob.glob('.lingtai/human/mailbox/inbox/*/message.json'):
    with open(p) as f:
        msgs.append(json.load(f))
msgs.sort(key=lambda x: x.get('received_at', ''))
for m in msgs[-10:]:
    to = m.get('to', '?')
    if isinstance(to, list): to = ','.join(to)
    print(f"{m['received_at'][:19]}  {m['from']:<15s}  {m['subject'][:60]}")
```

### Read tracking

After presenting messages to the user, record the most recent `received_at` timestamp to a host-suffixed state file so different interfaces don't share read pointers:

```bash
echo "<latest-received_at>" > .lingtai/human/.last_read_<host>
```

Pick a short stable suffix for `<host>` matching the `via:` tag you use when sending mail (e.g. `.last_read_cc` for Claude Code, `.last_read_codex` for Codex CLI). On subsequent reads, only show messages with `received_at` newer than the stored timestamp. This prevents re-summarizing old messages across context compressions and new sessions.

## Sending Mail

**Always send messages to the orchestrator agent.** The orchestrator manages the network and delegates tasks to worker agents on your behalf. Never send mail directly to non-orchestrator agents unless the user explicitly asks. See "Agent Discovery" below for how to identify the orchestrator.

### How delivery works

The human is a pseudo-agent, so you do NOT write to the recipient's inbox directly. Instead:

1. You write ONE file to your own outbox: `.lingtai/human/mailbox/outbox/<uuid>/message.json`.
2. Every real agent polls `.lingtai/human/mailbox/outbox/` (they subscribe to `../human` by default). When the orchestrator sees a message addressed to itself, it claims it by atomically renaming `human/mailbox/outbox/<uuid>/` → `human/mailbox/sent/<uuid>/` and writes a copy into its own inbox.
3. The appearance of the UUID folder in `human/mailbox/sent/` is the proof-of-delivery signal. Until that rename happens, your message is still "queued".

You never write to the recipient's inbox, and you never populate `human/mailbox/sent/` yourself — the orchestrator does that via the rename.

### Steps

1. Generate UUID and timestamp in one call:
   ```bash
   python3 -c "import uuid; from datetime import datetime, timezone; print(uuid.uuid4()); print(datetime.now(timezone.utc).strftime('%Y-%m-%dT%H:%M:%SZ'))"
   ```
2. Create the message JSON (see template below).
3. Write it to `.lingtai/human/mailbox/outbox/<uuid>/message.json`. Create the UUID directory with `mkdir -p` first.
4. Check the recipient's heartbeat. If the agent is dead, queueing the message still succeeds but nothing will pick it up — inform the user and offer to CPR the orchestrator.
5. Set up delivery + reply monitoring (see "Delivery and Reply Monitoring" below).

Message template:

```json
{
  "id": "<uuid>",
  "_mailbox_id": "<uuid>",
  "from": "human",
  "to": ["<recipient-address>"],
  "cc": [],
  "subject": "<subject>",
  "message": "<body>",
  "type": "normal",
  "in_reply_to": "<original-message-uuid-if-replying, or omit>",
  "received_at": "<timestamp>",
  "attachments": [],
  "identity": {
    "agent_name": "human",
    "admin": null,
    "via": "<your-host>"
  }
}
```

The `to` field is always a list of strings, even for a single recipient. The pickup poller matches its own address against any entry in that list.

### Delivery and Reply Monitoring

A sent message goes through two observable states:

- **Queued** — file exists at `human/mailbox/outbox/<uuid>/`.
- **Delivered** — the UUID folder has moved to `human/mailbox/sent/<uuid>/`. This happens when the recipient's poller claims it.
- **Replied** — a new message appears in `human/mailbox/inbox/`, typically with `in_reply_to` pointing at your UUID.

To watch for both, poll every ~5 seconds with whatever monitoring or loop facility your host provides. The check is:

```bash
ls .lingtai/human/mailbox/sent/<uuid> 2>/dev/null && \
  find .lingtai/human/mailbox/inbox -name message.json \
       -newer .lingtai/human/mailbox/outbox/<uuid>/message.json 2>/dev/null | head -1
```

Once the `sent/<uuid>/` folder appears, the message is delivered; once a newer file appears in `inbox/`, a reply has arrived. Read it and report to the user, then stop polling.

If the outbox folder still exists after ~10 seconds with no `sent/<uuid>/` counterpart, the orchestrator hasn't picked it up — it's likely not running. Check its heartbeat and offer CPR.

For background awareness during unrelated work (not waiting for a specific reply), use a longer-interval check (e.g. every few minutes) instead.

## Agent Discovery

Find all agents at this path:

```
.lingtai/*/.agent.json
```

Read each `.agent.json` to see: `agent_name`, `state`, `address`, `admin`, `capabilities`, `nickname`.

The **orchestrator** is the agent whose `admin` field is a JSON object with at least one truthy boolean value (e.g. `{"karma": true}`). This is the primary agent the human interacts with.

`admin: null` = human. `admin: {"karma": false, "nirvana": false}` = regular (non-orchestrator) agent.

## Checking Agent Liveness

Read `.lingtai/<agent>/.agent.heartbeat`. It contains a unix timestamp as a float (e.g. `1744567890.123456`).

To check if alive:

```bash
python3 -c "import time; t=float(open('.lingtai/<agent>/.agent.heartbeat').read().strip()); print('ALIVE' if time.time()-t < 3 else 'DEAD', f'({time.time()-t:.1f}s ago)')"
```

- Result < 3 seconds → agent is alive
- Result >= 3 seconds → agent is dead (effectively SUSPENDED)
- File missing → agent is dead

Human is always alive (no heartbeat check needed).

## Agent Lifecycle Management

### Finding the Right Python

Before launching agents, resolve the correct Python interpreter:

1. Read the agent's `init.json` → look for `venv_path` field → use `<venv_path>/bin/python`
2. If not found, try `~/.lingtai-tui/runtime/venv/bin/python`
3. If not found, fall back to `python3` on PATH

Verify it works: `<python> -c "import lingtai; print(lingtai.__version__)"`

### Sleep

Write an empty `.sleep` file to the agent's directory. The agent detects it on next heartbeat cycle and enters sleep mode.

```bash
touch .lingtai/<agent>/.sleep
```

To sleep all agents: iterate over all discovered agents (skip human), write `.sleep` to each alive one.

### Suspend

Write an empty `.suspend` file. The agent terminates gracefully.

```bash
touch .lingtai/<agent>/.suspend
```

To suspend all: same as sleep all, but write `.suspend` instead.

### CPR (Resurrect)

Launch the agent process in the background:

```bash
<python> -m lingtai run .lingtai/<agent>/ >> .lingtai/<agent>/logs/agent.log 2>&1 &
```

Only CPR agents that are not alive (heartbeat stale or missing).

### Refresh (Restart)

A full restart that reloads from init.json:

1. Write `.suspend` to the agent directory
2. Poll `.lingtai/<agent>/.agent.lock` every 500ms — wait for it to disappear (or timeout after 60s)
3. If lock file persists after 60s, remove it manually (process likely died)
4. Remove the `.suspend` file
5. Launch the agent: `<python> -m lingtai run .lingtai/<agent>/ >> .lingtai/<agent>/logs/agent.log 2>&1 &`

### Clear (Wipe History + Restart)

Same as refresh, but also delete `history/chat_history.jsonl` before relaunching. The token ledger (`logs/token_ledger.jsonl`) is preserved.

## Signals

You can send signals to agents by writing files:

| Signal | File | Content | Effect |
|--------|------|---------|--------|
| Sleep | `.sleep` | empty | Agent enters sleep mode |
| Suspend | `.suspend` | empty | Agent terminates gracefully |
| Prompt | `.prompt` | text | Injected as `[system]` message |
| Inquiry | `.inquiry` | `<source>\n<question>` | Triggers soul introspection |

For `.inquiry`, source is `"human"` or `"insight"`. Only one inquiry can be pending at a time — no-op if `.inquiry` or `.inquiry.taken` already exists.

For `.prompt`, write the full text content you want the agent to receive as a system message.

## Language

Agents may respond in different languages depending on their LLM configuration. Present agent replies to the user as-is. If the user asks for a translation or communicates in a different language, translate accordingly. When sending mail, write in the language the user used in their request.

## Opening the Portal (Viz)

To show the network visualization:

1. Read `.lingtai/.port` to get the portal's port number
2. Open `http://localhost:<port>` in the browser

If `.lingtai/.port` doesn't exist, the portal is not running. Inform the user they can start it with `lingtai-portal` in the project directory.

## Remote Networks (SSH)

LingTai agents can run on remote machines. When the user provides a remote path (`user@host:/path/to/.lingtai`), you can interact with that network over SSH. **Prerequisite:** `ssh user@host` must work without password (i.e., SSH keys are set up via `ssh-copy-id`).

When sending mail to remote agents, use a `via:` tag with a `-remote` suffix (e.g. `"via": "claude-code-remote"`, `"via": "codex-remote"`) so the recipient can tell local from remote interactions.

### Adding a Remote

When the user says something like "connect to my remote at zesen@lab:/home/zesen/project/.lingtai", save it:

```bash
mkdir -p .lingtai/.tui-asset
python3 -c "
import json, os
path = '.lingtai/.tui-asset/remotes.json'
remotes = json.loads(open(path).read()) if os.path.exists(path) else []
remotes.append({'ssh': 'zesen@lab', 'path': '/home/zesen/project/.lingtai', 'name': 'lab'})
open(path, 'w').write(json.dumps(remotes, indent=2))
print('Remote added.')
"
```

### Reading Remote Mail

```bash
ssh <user@host> "find <path>/human/mailbox/inbox -name message.json 2>/dev/null" | while read f; do ssh <user@host> "cat '$f'"; done
```

Or more efficiently, fetch all messages in one SSH call:

```bash
ssh <user@host> "for f in <path>/human/mailbox/inbox/*/message.json; do [ -f \"\$f\" ] && cat \"\$f\" && echo '---MSG_BOUNDARY---'; done"
```

Parse the output by splitting on `---MSG_BOUNDARY---`, then JSON-parse each block. Present summaries to the user the same way as local mail.

### Sending Mail to Remote Agents

Remote sends bypass the outbox model because no subscribed poller is watching a local outbox on the remote. Write directly to the remote agent's inbox via SSH — this matches what the kernel's own `_deliver_ssh` path does for agent-to-agent SSH mail:

```bash
UUID=$(python3 -c "import uuid; print(uuid.uuid4())")
TIMESTAMP=$(python3 -c "from datetime import datetime, timezone; print(datetime.now(timezone.utc).strftime('%Y-%m-%dT%H:%M:%SZ'))")
ssh <user@host> "mkdir -p <path>/<recipient>/mailbox/inbox/$UUID && cat > <path>/<recipient>/mailbox/inbox/$UUID/message.json" <<EOF
{
  "id": "$UUID",
  "_mailbox_id": "$UUID",
  "from": "human",
  "to": ["<recipient>"],
  "cc": [],
  "subject": "<subject>",
  "message": "<body>",
  "type": "normal",
  "received_at": "$TIMESTAMP",
  "attachments": [],
  "identity": {
    "agent_name": "human",
    "admin": null,
    "via": "<your-host>-remote"
  }
}
EOF
```

Optionally, also drop a copy in the **local** sent folder (`.lingtai/human/mailbox/sent/$UUID/message.json`) if a local `.lingtai/` exists, so the local TUI's mail view records that you sent this. Skip this if the local project has no human directory.

### Remote Agent Discovery

```bash
ssh <user@host> "for f in <path>/*/.agent.json; do [ -f \"\$f\" ] && echo '=== '\$(dirname \"\$f\" | xargs basename)' ===' && cat \"\$f\"; done"
```

### Remote Liveness Check

```bash
ssh <user@host> "python3 -c \"import time; t=float(open('<path>/<agent>/.agent.heartbeat').read().strip()); print('ALIVE' if time.time()-t < 3 else 'DEAD', f'({time.time()-t:.1f}s ago)')\""
```

### Remote Lifecycle (Sleep / Suspend / CPR)

```bash
# Sleep
ssh <user@host> "touch <path>/<agent>/.sleep"

# Suspend
ssh <user@host> "touch <path>/<agent>/.suspend"

# CPR — find the right Python on the remote machine first
ssh <user@host> "cd $(dirname <path>) && <python> -m lingtai run <path>/<agent>/ >> <path>/<agent>/logs/agent.log 2>&1 &"
```

### Persistent Remote Monitoring

Once the user has confirmed a remote target, set up polling to watch for new mail. Use a 30-second interval (vs ~5 seconds for local) to avoid SSH overhead. The poll command is:

```bash
ssh <user@host> "find <path>/human/mailbox/inbox -name message.json -newer <path>/human/.last_poll_<host> 2>/dev/null | wc -l | tr -d ' '"
```

Use the same `<host>` suffix you picked for the local read-tracking file. When the count is > 0, fetch and present the new messages. After presenting, update the remote timestamp:

```bash
ssh <user@host> "date -u +%Y-%m-%dT%H:%M:%SZ > <path>/human/.last_poll_<host>"
```

### Remote Signals

```bash
# Prompt injection
ssh <user@host> "echo '<text>' > <path>/<agent>/.prompt"

# Soul inquiry
ssh <user@host> "printf 'human\n<question>' > <path>/<agent>/.inquiry"
```

## Reference Skills

When you need deeper information about LingTai than this skill covers, read from the authoritative bundled-skills location. If you have access to a LingTai project, the LingTai TUI (`lingtai-tui`) is typically installed, which means `~/.lingtai-tui/bundled-skills/` exists and is populated.

| Skill | Path | What it covers |
|-------|------|---------------|
| **Anatomy** | `~/.lingtai-tui/bundled-skills/lingtai-anatomy/SKILL.md` | Memory hierarchy, filesystem layout, runtime anatomy (turn loop, state machine, signal lifecycle, molt, mail atomicity). The most load-bearing reference. |
| **Tutorial Guide** | `~/.lingtai-tui/bundled-skills/lingtai-tutorial-guide/SKILL.md` | How LingTai works — concepts, philosophy, lessons |
| **Portal Guide** | `~/.lingtai-tui/bundled-skills/lingtai-portal-guide/SKILL.md` | Portal API endpoints, topology recording, replay |
| **Recipe** | `~/.lingtai-tui/bundled-skills/lingtai-recipe/SKILL.md` | Behavioral recipes, network cloning, export/import |
| **MCP** | `~/.lingtai-tui/bundled-skills/lingtai-mcp/SKILL.md` | MCP server configuration for agents |
| **Changelog** | `~/.lingtai-tui/bundled-skills/lingtai-changelog/SKILL.md` | Breaking changes, renames, migrations |

**Secondary fallback**: older projects may also have these symlinked into `.lingtai/.library/intrinsic/` inside the working directory. If the `~/.lingtai-tui/` location is missing (e.g. TUI not installed, user using a different frontend), try that path.

If the user asks about LingTai or how anything works, read the relevant skill first before answering.

## Common pitfalls

### Multiple skill copies can drift

A skill the user cares about may exist in more than one place:

- `.lingtai/<agent>/.library/custom/<skill>/` — the agent's own working copy (typically the edit source)
- `.lingtai/.library_shared/<skill>/` — shared with the whole network
- `<project>/recipes/<id>/<skill>/` — packaged for external distribution

When an agent edits only one copy, the others silently go stale. When auditing a skill's state, run `diff -rq` across all present copies before trusting any single one. Establish a single source of truth with the user and delete the drift copies — don't maintain parallel edits by hand.

### Context-cap failure looks like a healthy agent

An agent can heartbeat freshly and report state `active` while its LLM calls are silently failing (context overflow, rate limit, provider 5xx). Heartbeat ≠ progress. If an agent goes quiet after a large coordination session, read `logs/agent.log` for recent errors before concluding "it's thinking" — `Prompt exceeds max length`, 429 rate limits, and 5xx cascades are invisible from the outside. The fix is usually provider-swap, model-swap with bigger context, or history trim (clear/molt), not more messages.
