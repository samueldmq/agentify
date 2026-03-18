# /agentify

**Your project, your agents, your rules.**

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that helps you set up an agentic workflow tailored to your project. Run `/agentify`, answer a few questions about your stack and preferences, and it generates a team of specialized agents ready to work for you — from planning to merge.

## What it does for you

```
You create an issue → Architect plans → Dev codes → Reviewer checks → QA validates → Merged
```

It generates everything you need:

- **Your agents** (`.claude/agents/*.md`) — pick the roles you want: architect, dev, reviewer, QA, or mix your own
- **Your dispatcher** (`.claude/skills/agile/SKILL.md`) — checks your task board and routes work to the right agent
- **Your board labels** — status labels on your GitHub Issues, Linear, or Jira to track work through your pipeline
- **Your CLAUDE.md** — documents the workflow so your whole project stays in sync

## Get started

1. Copy `SKILL.md` into `.claude/skills/agentify/SKILL.md` in your project
2. Run `/agentify` in Claude Code
3. Answer the setup questions (~2 min)
4. Create an issue and let your agents handle it

## Works with your stack

- **Task boards**: GitHub Issues, Linear, Jira
- **Languages**: JavaScript/TypeScript, Python, Rust, Go, Java/Kotlin
- **Role presets**: Full team, solo dev, minimal, or custom — you choose
