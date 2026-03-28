# Codex Agent Session Close Skill

Reusable Codex skill for safely closing spawned agents and handling requests to close the current session without touching unrelated agents.

## Contents

- `agent-session-close/SKILL.md`
- `agent-session-close/agents/openai.yaml`

## Install

Copy the `agent-session-close` folder into `~/.codex/skills/`.

## Use

Invoke the skill explicitly with:

```text
$agent-session-close
```

## What it does

- closes spawned sub-agents when the runtime exposes an agent id and a close tool
- closes only the exact targets requested
- states clearly when the current main session cannot self-terminate because the runtime exposes no real session-close API

## Important limitation

This skill does not fake self-termination. If the platform cannot close the current root session from inside the agent runtime, the skill forces the agent to say so plainly instead of claiming success.
