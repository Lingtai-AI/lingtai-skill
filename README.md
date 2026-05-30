# lingtai-skill

The canonical skill describing how to interact with a [LingTai](https://lingtai.ai) agent network through the shared human mailbox.

This is a single Markdown file at `skills/lingtai/SKILL.md` documenting the filesystem mailbox protocol — reading and sending mail, agent discovery, liveness checks, lifecycle management, and signal files. The protocol is pure filesystem; no SDK, no API, no dependencies.

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

The skill describes a protocol, not a tool. Both the Claude Code plugin and the Codex CLI plugin consume the same SKILL.md verbatim — keeping it here means there is exactly one source of truth, and the host-specific plugin repos own only their host-specific glue (hooks, manifests, install scripts).

## Requirements

- A running LingTai project (`.lingtai/` directory with agents)
- File read/write access
- Python 3 (for UUID generation and liveness checks)

## License

Apache-2.0
