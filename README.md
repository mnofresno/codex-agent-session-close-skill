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

A bare invocation now counts as an explicit request to close the current session context and, when supported, the current front `Terminal.app` window.

## What it does

- closes spawned sub-agents when the runtime exposes an agent id and a close tool
- closes only the exact targets requested
- can close the current front Terminal.app window on macOS when the user explicitly asks for it, or when the skill is invoked by itself
- states clearly when the current main session cannot self-terminate because the runtime exposes no real session-close API

## Important limitations

- Terminal window close is scoped to the front `Terminal.app` window only
- it does not touch unrelated windows or tabs
- it does not fake self-termination if the host is not Terminal.app or the runtime cannot close the root session from inside the agent
