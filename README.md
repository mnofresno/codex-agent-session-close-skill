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

A bare invocation counts as an explicit request to finish the current task and then force-close the current session context and, when supported, the current terminal window.

## Host support

- macOS `Terminal.app`: supported, with automatic confirmation of the active-process close sheet
- Linux X11: supported through `xdotool` or `wmctrl`
- Linux Wayland: not claimed as supported unless a compositor-specific close path is added later

## What it does

- closes spawned sub-agents when the runtime exposes an agent id and a close tool
- closes only the exact targets requested
- can force-close the current front terminal window on supported macOS and Linux hosts
- auto-confirms Terminal's active-process close sheet on macOS when it appears
- states clearly when the current main session cannot self-terminate because the runtime exposes no real session-close API

## Important limitations

- macOS close is scoped to the front `Terminal.app` window only
- Linux close is scoped to the current active X11 window only
- it does not touch unrelated windows or tabs
- it does not fake self-termination if the host window cannot be programmatically closed
