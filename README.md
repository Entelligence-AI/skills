# Entelligence Skills

A collection of [Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
for Claude Code and compatible coding agents, by [Entelligence AI](https://entelligence.ai).

Each skill lives in its own directory with a `SKILL.md` (and optional `references/`). Drop a skill
into your agent's skills folder and it becomes available to your coding agent.

## Skills

### [`entloop`](entloop/SKILL.md)

Iteratively fixes a GitHub pull request until Entelligence gives it a **5/5 "Safe to Merge"**
confidence score with zero unresolved review threads. It triggers an Entelligence review, applies
each comment's committable suggestion or "Prompt to fix with AI", resolves the threads, pushes,
re-triggers the review, and repeats (up to 5 iterations).

Requires `git`, an authenticated [`gh`](https://cli.github.com/), and the
[Entelligence PR review app](https://github.com/apps/entelligence-ai-pr-reviews) installed on the
repo. GitHub only.

## Install

A skill is just a directory containing a `SKILL.md`. Install one by copying its directory into your
Claude Code skills folder:

```bash
git clone https://github.com/Entelligence-AI/skills.git

# user-level (available in every repo)
cp -r skills/entloop ~/.claude/skills/

# or per-project
mkdir -p .claude/skills && cp -r skills/entloop .claude/skills/
```

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
