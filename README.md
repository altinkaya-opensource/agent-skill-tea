# agent-skill-tea

An [Agent Skill](https://agentskills.io) that teaches AI coding agents to use
**[`tea`](https://gitea.com/gitea/tea)** — the command-line client for **Gitea** and
**Forgejo** git servers. Think of it as the `gh` (GitHub CLI) equivalent for everything
that isn't GitHub.

`tea` is niche, so agents tend to fall back on `gh` muscle memory and get the syntax wrong.
This skill encodes the verified command surface (tea 0.14.x), the gotchas that silently
break automation — e.g. a PR/issue body is `--description`, **not** `--body`; you view an
item by its bare index (`tea pr 42`), not `tea pr view` — and a full `gh → tea` cheat sheet.

## Install

Clone straight into your agent's skills directory (the folder **must** be named `tea` to
match the skill's `name`):

```bash
# Claude Code, user-level (every project)
git clone https://git.altinkaya.com/altinkaya-opensource/agent-skill-tea.git ~/.claude/skills/tea

# Cross-client (Codex and other skills-compatible agents)
git clone https://git.altinkaya.com/altinkaya-opensource/agent-skill-tea.git ~/.agents/skills/tea
```

You can also scope it to a single repo at `<project>/.claude/skills/tea/`. Restart the agent;
it auto-activates (progressive disclosure) whenever you work in a Gitea/Forgejo repository.

## Contents

- `SKILL.md` — the skill: auth setup, PR/issue/repo/release workflows, the gotchas, and the
  `gh → tea` command map.

## License

No license yet — add one if you intend others to reuse it.
