# lingtai-mailbox-skill

A universal skill that teaches coding agents how to interact with [LingTai](https://lingtai.ai) agent networks through the filesystem-based mailbox protocol.

Works with any coding agent that can read files: **OpenCode**, **Codex CLI**, **Hermes**, **OpenClaw**, or any tool with file access.

> **Claude Code users:** Use the dedicated plugin instead — [`Lingtai-AI/claude-code-plugin`](https://github.com/Lingtai-AI/claude-code-plugin)

## What it teaches

- Reading and sending mail through `.lingtai/human/mailbox/`
- Agent discovery and liveness checks
- Lifecycle management (sleep, suspend, CPR, refresh)
- Signal files (prompt injection, soul inquiry)
- The orchestrator routing convention

## Install

### OpenCode

```bash
# Global
git clone https://github.com/Lingtai-AI/lingtai-mailbox-skill.git ~/.config/opencode/skills/lingtai-mailbox-skill

# Project-scoped
git clone https://github.com/Lingtai-AI/lingtai-mailbox-skill.git .opencode/skills/lingtai-mailbox-skill
```

### Codex CLI

Copy the skill content into your project's `AGENTS.md`:

```bash
# Append the full protocol reference
cat path/to/lingtai-mailbox-skill/skills/lingtai-mailbox/SKILL.md >> AGENTS.md
```

Or just add a pointer:

```markdown
<!-- In AGENTS.md -->
This project has a LingTai agent network. Read .lingtai/.library/intrinsic/ for protocol docs.
```

### Any other tool

The skill is a single markdown file at `skills/lingtai-mailbox/SKILL.md`. Read it, follow it. The protocol is pure filesystem — no SDK, no API, no dependencies.

## Requirements

- A running LingTai project (`.lingtai/` directory with agents)
- File read/write access
- Python 3 (for UUID generation and liveness checks)

## License

MIT
