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
- can close the current front Terminal.app window on macOS when the user explicitly asks for it
- states clearly when the current main session cannot self-terminate because the runtime exposes no real session-close API

## Important limitations

- Terminal window close is scoped to the front `Terminal.app` window only
- it does not touch unrelated windows or tabs
- it does not fake self-termination if the host is not Terminal.app or the runtime cannot close the root session from inside the agent
