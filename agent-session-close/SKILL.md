---
name: "agent-session-close"
description: "Use when the user asks to close a spawned agent, terminate an agent session, end the current Codex instance, or close the current Terminal window for the running session without affecting unrelated work. Prefer explicit agent IDs and state clearly when the runtime cannot self-terminate or the host is not Terminal.app."
---

# Agent Session Close

Use this skill when the user wants an agent or session closed cleanly and safely.

## Core rule

Close only the exact agent or session the user requested. Never guess.

Invoking `$agent-session-close` by itself counts as an explicit request to close the current session context and, when possible, force-close the current Terminal.app front window after the requested task is complete.

## What this skill can do

- Close a spawned sub-agent when you have its agent id.
- Close multiple spawned sub-agents when the requested target set is explicit.
- If explicitly requested and the host is Terminal.app on macOS, force-close the front Terminal window that owns the current session via AppleScript/JXA, confirming the close sheet automatically when Terminal warns about active processes.
- Cleanly report that the current or root session cannot self-terminate when the runtime exposes no session-close tool and the host window cannot be programmatically closed.

## What this skill must not pretend to do

- Do not claim the current main conversation was closed unless a real session-close tool exists in the current runtime.
- Do not claim the Terminal window was closed unless the automation call succeeded.
- Do not close unrelated agents just because they look stale.
- Do not use repo actions, shell exits, or destructive cleanup as a fake replacement for closing an agent session.

## Target selection

1. If the user provides an agent id, close exactly that id.
2. If the user asks to close "the agent from this task" and you created exactly one spawned agent for the task, close that one.
3. If the user asks to close the current instance only, do not close any other agents unless they explicitly asked for them too.
4. If the user invokes `$agent-session-close` with no extra target, treat that as a request to close the current session context and force-close the current Terminal.app front window if supported.
5. If the user explicitly asks to close the Terminal window for the current session, target only the front Terminal.app window and only after all requested work is complete.
6. If the target is ambiguous and there is real risk of closing the wrong thing, ask one short question.

## Tool workflow

1. Prefer `close_agent` for spawned agents.
2. If the environment exposes a true session-level close tool, use it only for the exact requested session.
3. If the user explicitly asked to close the current Terminal window, or invoked `$agent-session-close` with no extra target, and macOS automation is available, use `mcp__macos_automator__execute_script` to force-close only the front Terminal.app window.
4. The Terminal close path must auto-confirm the default close sheet if Terminal warns about active processes in the window.
5. If no self-close tool exists for the current session, still attempt the Terminal front-window close path when requested or implied by bare skill invocation.
6. If neither a self-close tool nor Terminal close path is available, say so plainly and stop after a concise final handoff.

## macOS Terminal close path

Use this only when the user explicitly asks to close the current Terminal window or session window.

Preferred AppleScript:

```applescript
tell application "Terminal"
  if not (exists front window) then return "no-front-window"
  close front window saving no
end tell

delay 0.2

tell application "System Events"
  tell process "Terminal"
    if exists sheet 1 of front window then
      key code 36
    end if
  end tell
end tell

return "force-closed-front-window"
```

Rules:

- Target `Terminal` only, not every terminal emulator.
- Close only `front window`.
- Do not iterate all windows or all tabs.
- Run it only after the final user-facing message is ready.
- If Terminal shows the active-process confirmation sheet, accept the default button automatically.
- If the runtime is not hosted in Terminal.app, report the limitation instead of improvising.

## Response contract

After attempting a close:

- State which agent ids were closed.
- State which requested targets could not be closed and why.
- If the current Terminal window was explicitly requested and successfully closed, say that the front Terminal window was force-closed.
- If the current session cannot self-terminate, say: `No spawned agents remain open on my side. The current session must be closed by the host or UI.`

## Safety checks

- Finish or summarize critical work before closing a spawned agent.
- Prepare the final response before triggering Terminal window close automation.
- Reuse known agent ids from the current thread; do not infer ids from branch names, PR numbers, or filenames.
- If `close_agent` returns an error, surface the exact failure instead of masking it.
- If Terminal automation fails, surface the exact failure and do not claim the session was closed.

## Examples

- User: `close agent 123`
  Action: call `close_agent` with `123`, then report the result.

- User: `close this instance only`
  Action: do not close unrelated spawned agents. If no session-close tool exists, state the limitation plainly.

- User: `close all the agents you opened for this task`
  Action: close only the tracked ids created for the current task and report each result.

- User: `$agent-session-close`
  Action: treat the invocation itself as an explicit request to close the current session context and force-close the front Terminal.app window if supported.

- User: `close this codex session and the Terminal window`
  Action: close any explicitly requested spawned agents first, then run Terminal front-window close automation if available, otherwise state the host limitation plainly.
