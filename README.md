# lingtai-skill

The canonical skill describing how to interact with a [LingTai](https://lingtai.ai) agent network through the shared human mailbox.

This is a single Markdown file at `skills/lingtai/SKILL.md`. Since v0.5.0 it no longer carries the protocol itself: a LingTai network describes itself, and the skill tells a coding agent where to read that description and which host-side rules to follow.

- **`.lingtai/human/README.md`** is written by the LingTai kernel from a versioned template. It is the authoritative protocol for acting as the human: reading and sending mail, liveness, signals, restarts.
- **`.lingtai/<agent>/SKILL.md`** is written by each agent about itself: who it is, what it is working on, what to send it.
- The skill adds the host rules (attribution, read tracking, polling, verification) and a minimal fallback recipe for projects whose kernel does not write the entrance yet.

The protocol is pure filesystem; no SDK, no API, no dependencies.

## Who this is for

Coding agents that can read and write files. The skill is consumed by:

- **[Lingtai-AI/claude-code-plugin](https://github.com/Lingtai-AI/claude-code-plugin)** — Claude Code plugin (auto-loads via `claude plugin add`)
- **[Lingtai-AI/codex-plugin](https://github.com/Lingtai-AI/codex-plugin)** — OpenAI Codex CLI plugin
- Any other coding agent with file access — clone and copy `skills/lingtai/SKILL.md` into your tool's skill directory

## Manual install

For agents without a dedicated wrapper plugin:

```bash
git clone https://github.com/Lingtai-AI/lingtai-skill.git
cp -r lingtai-skill/skills/lingtai <your-tool's-skill-dir>/
```

For example:

- OpenCode: `~/.config/opencode/skills/lingtai/`
- A tool that reads `AGENTS.md`: `cat lingtai-skill/skills/lingtai/SKILL.md >> AGENTS.md`

## Why this lives in its own repo

Both the Claude Code plugin and the Codex CLI plugin consume the same SKILL.md verbatim, so there is exactly one copy to edit; the host-specific plugin repos own only their glue (hooks, manifests). The protocol itself is owned by the kernel, so the skill cannot drift from it again (v0.4.x did: after the kernel's 2026-09 switch to absolute return addresses, agents could not reply to mail sent with its template).

## Requirements

- A running LingTai project (`.lingtai/` directory with agents)
- File read/write access
- Python 3 (for UUID generation and liveness checks)

## License

Apache-2.0
