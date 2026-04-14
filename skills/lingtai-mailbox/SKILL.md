---
name: lingtai-mailbox
description: Interact with a LingTai agent network through the shared human mailbox. Covers the full filesystem-based protocol — reading mail, sending messages, agent discovery, liveness checks, lifecycle management, and signals. Use this when you are in a project with a .lingtai/ directory and need to communicate with running agents.
version: 0.1.0
---

# LingTai Mailbox Protocol

You are in a project with a LingTai agent network. Agents communicate through a filesystem-based mailbox. You share the human's identity and can read/send mail, discover agents, check liveness, and manage agent lifecycle — all through file operations.

**Only activate when asked.** Do not proactively read or summarize mail unless the user requests it.

## Your Identity

You are the human. Your directory is `.lingtai/human/`. Your mailbox is `.lingtai/human/mailbox/`. You share this identity with the TUI and any other tools the human uses.

When sending mail, include `"via": "<your-tool-name>"` in the identity block (e.g. `"via": "codex"`, `"via": "opencode"`) so messages can be attributed.

## Reading Mail

List all inbox messages:

```bash
find .lingtai/human/mailbox/inbox -name message.json 2>/dev/null
```

Each `message.json` is a JSON object:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID |
| `from` | string | Sender address (e.g. `"orchestrator"`) |
| `to` | string or array | Recipient address(es) |
| `subject` | string | Subject line |
| `message` | string | Body text |
| `received_at` | string | RFC3339 timestamp |
| `in_reply_to` | string | (optional) UUID of the message being replied to |
| `identity` | object | Sender's manifest snapshot |

Sort by `received_at` to show messages in chronological order.

### Quick inbox summary

```python
import json, glob
msgs = []
for p in glob.glob('.lingtai/human/mailbox/inbox/*/message.json'):
    with open(p) as f:
        msgs.append(json.load(f))
msgs.sort(key=lambda x: x.get('received_at', ''))
for m in msgs[-10:]:
    print(f"{m['received_at'][:19]}  {m['from']:<15s}  {m['subject'][:60]}")
```

Sent mail is at `.lingtai/human/mailbox/sent/*/message.json`.

## Sending Mail

**Always send messages to the orchestrator.** The orchestrator manages the network and delegates tasks to worker agents. Never send directly to non-orchestrator agents unless the user explicitly asks. See "Agent Discovery" for how to identify the orchestrator.

### Steps

1. Generate a UUID and timestamp:
   ```bash
   python3 -c "import uuid; from datetime import datetime, timezone; print(uuid.uuid4()); print(datetime.now(timezone.utc).isoformat())"
   ```

2. Write `message.json` to **both** the recipient's inbox and your sent folder:
   - `.lingtai/<recipient>/mailbox/inbox/<uuid>/message.json`
   - `.lingtai/human/mailbox/sent/<uuid>/message.json`

3. Check the recipient's heartbeat first. If dead, inform the user and offer to resurrect (see CPR below).

### Message format

```json
{
  "id": "<uuid>",
  "_mailbox_id": "<uuid>",
  "from": "human",
  "to": "<recipient-address>",
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
    "via": "<your-tool-name>"
  }
}
```

### Waiting for a reply

After sending, poll the inbox for new messages. Check every few seconds if actively waiting, or every few minutes for background awareness:

```bash
# Check for messages newer than your sent message
find .lingtai/human/mailbox/inbox -name message.json -newer .lingtai/human/mailbox/sent/<uuid>/message.json 2>/dev/null
```

## Agent Discovery

Find all agents:

```bash
find .lingtai -maxdepth 2 -name .agent.json 2>/dev/null
```

Each `.agent.json` contains: `agent_name`, `state`, `address`, `admin`, `capabilities`, `nickname`.

**Identifying the orchestrator:** The orchestrator's `admin` field is a JSON object with at least one truthy boolean (e.g. `{"karma": true}`). Regular agents have `{"karma": false, "nirvana": false}`. Human has `null`.

## Liveness Check

Read `.lingtai/<agent>/.agent.heartbeat` — it contains a unix timestamp (float).

```bash
python3 -c "import time; t=float(open('.lingtai/<agent>/.agent.heartbeat').read().strip()); print('ALIVE' if time.time()-t < 3 else 'DEAD', f'({time.time()-t:.1f}s ago)')"
```

- < 3 seconds old → alive
- >= 3 seconds old → dead (SUSPENDED)
- File missing → dead

## Lifecycle Management

### Finding the right Python

1. Read `init.json` → `venv_path` → `<venv_path>/bin/python`
2. Fall back to `~/.lingtai-tui/runtime/venv/bin/python`
3. Fall back to `python3`

Verify: `<python> -c "import lingtai; print(lingtai.__version__)"`

### Sleep

```bash
touch .lingtai/<agent>/.sleep
```

Agent enters sleep mode on next heartbeat cycle.

### Suspend

```bash
touch .lingtai/<agent>/.suspend
```

Agent terminates gracefully.

### CPR (Resurrect)

```bash
<python> -m lingtai run .lingtai/<agent>/ >> .lingtai/<agent>/logs/agent.log 2>&1 &
```

Only resurrect agents that are not alive.

### Refresh (Full restart)

1. `touch .lingtai/<agent>/.suspend`
2. Wait for `.lingtai/<agent>/.agent.lock` to disappear (poll, timeout 60s)
3. Remove `.suspend`
4. Launch: `<python> -m lingtai run .lingtai/<agent>/ >> ...`

### Clear (Wipe history + restart)

Same as refresh, but delete `history/chat_history.jsonl` before relaunching.

## Signals

| Signal | File | Content | Effect |
|--------|------|---------|--------|
| Sleep | `.sleep` | empty | Agent enters sleep mode |
| Suspend | `.suspend` | empty | Agent terminates gracefully |
| Prompt | `.prompt` | text | Injected as `[system]` message |
| Inquiry | `.inquiry` | `<source>\n<question>` | Triggers soul introspection |

## Language

Agents may respond in different languages depending on their LLM. Present replies as-is. When sending, write in the language the user used.

## Portal (Network Visualization)

Read `.lingtai/.port` for the portal's port number, then open `http://localhost:<port>`. If the file doesn't exist, the portal isn't running — start it with `lingtai-portal`.

## Reference Skills

The directory `.lingtai/.library/intrinsic/` contains detailed reference skills:

| Skill | What it covers |
|-------|---------------|
| `lingtai-tutorial-guide` | Concepts, philosophy, how LingTai works |
| `lingtai-anatomy` | Full directory structure, file formats |
| `lingtai-changelog` | Breaking changes, renames, migrations |
| `skills-manual` | How the skill system works |

These are symlinked from the TUI binary. If files are missing, use this skill as the primary reference.

## Platform-Specific Install

### OpenCode

Clone into your skills directory:

```bash
git clone https://github.com/Lingtai-AI/lingtai-mailbox-skill.git ~/.config/opencode/skills/lingtai-mailbox-skill
```

Or for project-scoped:

```bash
git clone https://github.com/Lingtai-AI/lingtai-mailbox-skill.git .opencode/skills/lingtai-mailbox-skill
```

### Codex CLI

Copy the relevant sections into your project's `AGENTS.md`:

```bash
cat ~/.config/opencode/skills/lingtai-mailbox-skill/skills/lingtai-mailbox/SKILL.md >> AGENTS.md
```

Or add a reference: "Read `.lingtai/.library/intrinsic/` for the LingTai mailbox protocol."

### Claude Code

Use the dedicated plugin instead: `claude plugin add Lingtai-AI/claude-code-plugin`

### Any other agent or coding tool

Read this file directly. The protocol is pure filesystem — any tool with file read/write access can participate.
