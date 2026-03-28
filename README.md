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

- macOS `Terminal.app`: supported, using a stable `tty` lookup to find the owning window
- Linux X11: supported through a captured window id plus `xdotool` or `wmctrl`
- Linux Wayland: not claimed as supported unless a compositor-specific close path is added later

## Important fix

The skill no longer relies on whichever window is focused at the end. It now captures a stable handle at skill start and closes that exact window later.

## Important limitations

- macOS close is scoped to the Terminal window that owns the captured `tty`
- Linux close is scoped to the captured X11 window id for this session
- it does not touch unrelated windows or tabs
- it does not fake self-termination if the host window cannot be programmatically closed

## Follow-up prompts

This repository contains the skill-specific suppression for post-task follow-up offers when `$agent-session-close` is active.

If you also want Codex to stop suggesting "Do you want me to..." style next steps globally, add a rule in your home or project `AGENTS.md` that overrides default next-step suggestions in final responses.
