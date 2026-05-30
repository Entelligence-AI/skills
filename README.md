# Entelligence Skills

[Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) for automated
PR review workflows with [Entelligence AI](https://entelligence.ai), usable from Claude Code and
compatible coding agents. Requires `git` + the [`gh`](https://cli.github.com) CLI, and the
Entelligence PR review app installed on the repo. GitHub only.

## Skills

| Skill | Description |
|-------|-------------|
| [`entloop`](entloop/) | Loop: trigger an Entelligence review, fix every comment, resolve the threads, re-review, until 5/5 "Safe to Merge" confidence and zero unresolved threads. |

## Requirements

| Need | Tool / setup | Install |
|------|--------------|---------|
| Git host CLI | `gh` (authenticated) | [cli.github.com](https://cli.github.com) |
| Reviewer | Entelligence PR review app on the repo | [github.com/apps/entelligence-ai-pr-reviews](https://github.com/apps/entelligence-ai-pr-reviews) |

Authenticate before use: `gh auth login`.

## Install

Clone the repo and symlink the skills you want into your Claude Code skills directory (a skill must
be a direct child of `~/.claude/skills/`):

```bash
git clone https://github.com/Entelligence-AI/skills.git ~/.claude/skills/entelligence
ln -s ~/.claude/skills/entelligence/entloop ~/.claude/skills/entloop
```

Or just copy the one you want:

```bash
cp -r entloop ~/.claude/skills/
```

Per-project installs work too: use `.claude/skills/` inside the repo instead of `~/.claude/skills/`.

Restart Claude Code. The agent discovers the skill by its description; on your PR branch, ask it to
"loop this PR with entloop" (or invoke the skill directly). Because a skill is plain markdown plus
the tools it declares, the same files work with other agents (Cursor and similar) too.

## Related

- [`entelligence-claude-code`](https://github.com/Entelligence-AI/entelligence-claude-code) - the
  `/entelligence-review` skill and MCP server for reviewing a PR or local diff. `entloop` is the
  autofix-loop companion to that review.
- [`entelligence` CLI](https://github.com/Entelligence-AI/cli) - run an Entelligence review on your
  local diff from the terminal.

## License

[MIT](LICENSE)
