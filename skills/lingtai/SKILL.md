---
name: lingtai
description: Talk to a LingTai agent network from a coding agent by acting as the human through the shared mailbox in `.lingtai/`. Use when the user asks about their LingTai agents, wants to check or send LingTai mail, check whether agents are alive, or sleep, suspend, refresh, clear or CPR an agent.
version: 0.5.0
---

# LingTai

You act as the **human** of a LingTai network. You share the human's directory `.lingtai/human/` and its mailbox. You
have no agent identity of your own; you are another device the human uses to reach their agents.

**Opt-in.** Only act when the user asks you to do something with the network. Do not read, summarize or poll mail on
your own initiative.

This skill does not describe the protocol. The network describes itself, and those files win over anything you
remember or anything written here.

## 1. Read the entrance before acting

1. Find the project: the nearest directory, from the working directory upward, that contains `.lingtai/`.
2. Read **`.lingtai/human/README.md`**. The LingTai kernel writes it from a versioned template, and it is the
   authoritative protocol: how to read and send mail, check liveness, send signals, and restart agents. Follow it
   exactly. If it is missing, use the fallback in section 3.
3. Discover agents: `.lingtai/*/.agent.json`. The **orchestrator** is the agent whose `admin` is an object with at least
   one truthy value. `admin: null` marks a pseudo-agent such as the human.
4. Read each relevant agent's own entrance, **`.lingtai/<agent>/SKILL.md`**, if present. Read the orchestrator's first.
   The agent writes this file itself: who it is, what it is doing, and what to send it. Do not edit another agent's
   `SKILL.md` or `README.md`.

Re-read the entrance each session rather than relying on memory; kernels upgrade underneath you.

## 2. Rules for the host (always apply)

- **Attribution.** Every message you send carries `"via": "<host>"` in its `identity` block: `claude-code` for Claude
  Code, `codex` for Codex CLI, or a short stable name for any other tool.
- **Read tracking.** After showing mail to the user, write the newest `received_at` you showed to
  `.lingtai/human/.last_read_<host>` (`.last_read_cc` for Claude Code, `.last_read_codex` for Codex). Next time, show
  only newer mail. Different hosts keep separate pointers.
- **Address the orchestrator.** Send to the orchestrator, which delegates to other agents, unless the user names
  another agent.
- **Write in the user's language.** Show agents' replies as they are; translate only when asked.
- **Waiting for a reply.** After sending, watch for delivery and the reply with whatever background facility your host
  has: a Monitor or a background shell loop in Claude Code, or a short polling loop in Codex. Poll every few seconds
  while the user waits; stop as soon as the reply has been shown. For background awareness during other work, check
  every few minutes, and only if the user asked for it.
- **Trust but verify.** Agents can report work as done that never reached disk. Before relaying a "done" on real work
  (files written, exports, translations), check one load-bearing claim with `ls`, `find` or `grep`. If the claim is
  false, tell the user and mail the agent the evidence.
- **Heartbeat is not progress.** An agent can be alive while its LLM calls fail (context overflow, rate limits,
  provider errors). If it goes quiet, read `.lingtai/<agent>/logs/agent.log` and `.lingtai/<agent>/.notification/`
  before sending more mail.
- **Do not touch kernel-owned files**: `.agent.json`, `.agent.heartbeat`, `.agent.lock`, `.status.json`, `README.md`,
  `system/`, `.library/intrinsic/`, `history/`, `logs/`, `.alarm`, and another agent's `mailbox/`.

Deeper documentation: every agent carries the kernel's manuals at
`.lingtai/<agent>/.library/intrinsic/capabilities/<skill>/SKILL.md` (for example `lingtai-kernel-anatomy`, `email`,
`system-manual`), and the TUI installs guides at `~/.lingtai-tui/utilities/<skill>/SKILL.md` (for example
`lingtai-tutorial-guide`, `lingtai-portal-guide`, `lingtai-tui-anatomy`). Read the relevant one before answering
questions about how LingTai works.

## 3. Fallback: no `.lingtai/human/README.md`

Older kernels do not write the entrance. Use this minimal recipe, which matches kernels from 2026-09 onward, and tell
the user that upgrading LingTai (`lingtai-tui self-update`) makes the network describe itself.

**Read mail.** Messages are `.lingtai/human/mailbox/inbox/<id>/message.json`; filed mail is under `archive/`. The id is
the folder name, also in `_mailbox_id`. `from` is the sender's absolute directory; show its basename. Sort by
`received_at`. Kernel replies do not set `in_reply_to`; match a reply by sender, a `Re: <subject>` subject, and arrival
after your send.

**Send mail.** One recipient per message. Write a single file and nothing else:

```bash
cd <project>
ID=$(python3 -c "import secrets,datetime;print(datetime.datetime.now(datetime.timezone.utc).strftime('%Y%m%dT%H%M%S')+'-'+secrets.token_hex(2))")
mkdir -p .lingtai/human/mailbox/outbox/$ID
touch .lingtai/human/.send-marker-$ID     # fixed send time, for reply detection
```

`.lingtai/human/mailbox/outbox/$ID/message.json`:

```json
{
  "id": "<ID>",
  "_mailbox_id": "<ID>",
  "from": "human",
  "to": ["<agent address: the `address` in its .agent.json, i.e. its directory name, never a path>"],
  "cc": [],
  "subject": "<subject>",
  "message": "<body>",
  "type": "normal",
  "received_at": "<UTC RFC3339 timestamp>",
  "attachments": [],
  "identity": {"agent_name": "human", "admin": null, "via": "<host>"},
  "_return_route": {"address": "<absolute path of .lingtai/human>", "sender_agent_id": ""}
}
```

`_return_route` is required: without it the agent cannot reply. The addressed agent claims the message by moving its
folder to `.lingtai/human/mailbox/sent/$ID`; that move is the proof of delivery. Never write into an agent's inbox or
into `human/mailbox/sent/` yourself. If the folder is still in the outbox after about 10 s, the agent is not running.

**Wait for the reply.**

```bash
ls .lingtai/human/mailbox/sent/$ID >/dev/null 2>&1 && echo DELIVERED
find .lingtai/human/mailbox/inbox -name message.json -newer .lingtai/human/.send-marker-$ID
```

Check each new message's sender and subject; agents also send unrelated mail. Delete the marker once the reply is
shown.

**Liveness.** `.lingtai/<agent>/.agent.heartbeat` holds a Unix time as a float; the agent is alive if it is less than
10 s old. Missing or stale means not running; ignore `state` in `.agent.json`, which goes stale when a process dies.
The human is always treated as live.

**Signals** (an empty file in the agent's directory unless noted; the agent must be running):

| File | Effect |
|---|---|
| `.sleep` | Sleep; new mail wakes it |
| `.suspend` | Exit gracefully |
| `.interrupt` | Cancel the current turn |
| `.refresh` | Restart and reload `init.json` |
| `.clear` (body `human`) | Wipe the conversation (forced molt); history is archived |
| `.prompt` (body = text) | Inject the text as a `[system]` message |

**CPR** (start an agent that is not running): resolve the interpreter's environment from `venv_path` in the agent's
`init.json`, else `~/.lingtai-tui/runtime/venv`, then

```bash
<venv>/bin/lingtai-agent run .lingtai/<agent> >> .lingtai/<agent>/logs/agent.log 2>&1 &
```

Never delete `.agent.lock`; the kernel refuses to start a second copy on its own.

**Portal.** The port is in `.lingtai/.portal/port`; open `http://localhost:<port>`. If the file is missing, the user can
start it with `lingtai-portal` in the project directory.
