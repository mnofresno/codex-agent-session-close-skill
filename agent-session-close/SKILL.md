---
name: "agent-session-close"
description: "Use when the user asks to close a spawned agent, terminate an agent session, end the current Codex instance, or close the current terminal window for the running session on macOS Terminal.app or Linux X11 without affecting unrelated work. Prefer explicit agent IDs and state clearly when the runtime cannot self-terminate or the host window cannot be programmatically closed."
---

# Agent Session Close

Use this skill when the user wants an agent or session closed cleanly and safely.

## Core rule

Close only the exact agent or session the user requested. Never guess.

Invoking `$agent-session-close` by itself counts as an explicit request to close the current session context and, when possible, force-close the current terminal window after the requested task is complete.

The terminal window must be identified by a stable handle captured when the skill starts, not by whichever window is focused at the end.

## What this skill can do

- Close a spawned sub-agent when you have its agent id.
- Close multiple spawned sub-agents when the requested target set is explicit.
- If explicitly requested and the host is Terminal.app on macOS, force-close the Terminal window that owns the current session using a stable `tty`-based lookup, confirming the close sheet automatically when Terminal warns about active processes.
- If explicitly requested and the host is Linux on X11, force-close the terminal window captured at skill start via `WINDOWID`, `xdotool`, or `wmctrl`.
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
4. If the user invokes `$agent-session-close` with no extra target, treat that as a request to close the current session context and force-close the current terminal window if supported.
5. If the user explicitly asks to close the terminal window for the current session, target only the host window captured for this session and only after all requested work is complete.
6. If the target is ambiguous and there is real risk of closing the wrong thing, ask one short question.

## Tool workflow

1. Prefer `close_agent` for spawned agents.
2. If the environment exposes a true session-level close tool, use it only for the exact requested session.
3. If the user explicitly asked to close the current terminal window, or invoked `$agent-session-close` with no extra target, capture a stable host-window handle immediately when the skill starts.
4. Detect the host close path:
   - macOS `Terminal.app`: use `mcp__macos_automator__execute_script`
   - Linux X11: use `exec_command` with `xdotool` or `wmctrl`
5. The macOS Terminal close path must auto-confirm the default close sheet if Terminal warns about active processes in the window.
6. The Linux close path must target only the captured window id and must not depend on the active window at close time.
7. If no self-close tool exists for the current session, still attempt the host window close path when requested or implied by bare skill invocation.
8. If neither a self-close tool nor a supported host window close path is available, say so plainly and stop after a concise final handoff.

## Stable handle capture

Capture the host-window handle before doing the user's requested work.

- macOS Terminal.app: capture the controlling `tty` for the owning shell or Codex process. Prefer interactive `tty`, but do not stop there.
- Linux X11: capture `$WINDOWID` if present; otherwise capture the active window id immediately at skill start and reuse that exact id later

Do not re-resolve the target from focus at the end.

### macOS Terminal.app handle capture order

Use this order and keep the first non-empty match:

1. `tty`
2. `ps -o tty= -p "$PPID"` if the immediate parent carries the terminal `tty`
3. Walk the parent process chain and read `tty` from the first ancestor that is attached to `/dev/ttys*`

`TERM_PROGRAM=Apple_Terminal` and `TERM_SESSION_ID` are useful signals that the host is Terminal.app, but they are not sufficient on their own to pick a window safely. Use them as supporting evidence, not as the sole close target.

If `tty` prints `not a tty`, that does not prove the session is unbound to Terminal.app. In Codex-managed executions, commands often run without an interactive stdin even though the owning shell and Codex process still have a stable `tty` visible through `ps`.

## macOS Terminal close path

Use this only when the user explicitly asks to close the current Terminal window or session window.

Preferred AppleScript:

```applescript
on run argv
  set targetTty to item 1 of argv
  tell application "Terminal"
    repeat with w in windows
      repeat with t in tabs of w
        try
          if tty of t is equal to targetTty then
            close w saving no
            delay 0.2
            tell application "System Events"
              tell process "Terminal"
                if exists sheet 1 of front window then
                  key code 36
                end if
              end tell
            end tell
            return "force-closed-terminal-window-by-tty"
          end if
        end try
      end repeat
    end repeat
  end tell
  return "tty-not-found"
end run
```

Shell-side capture step before work:

```bash
SESSION_TTY="$(tty)"
```

Robust capture step for Codex-managed runtimes:

```bash
capture_session_tty() {
  local candidate=""
  candidate="$(tty 2>/dev/null || true)"
  if [ -n "$candidate" ] && [ "$candidate" != "not a tty" ]; then
    printf '%s\n' "$candidate"
    return 0
  fi

  candidate="$(ps -o tty= -p "${PPID:-}" 2>/dev/null | tr -d ' ' || true)"
  if [ -n "$candidate" ] && [ "$candidate" != "??" ]; then
    printf '/dev/%s\n' "$candidate"
    return 0
  fi

  python3 - <<'PY'
import os
import subprocess

pid = os.getppid()
seen = set()

while pid and pid not in seen:
    seen.add(pid)
    try:
        tty = subprocess.check_output(
            ["ps", "-o", "tty=", "-p", str(pid)], text=True
        ).strip()
    except Exception:
        tty = ""

    if tty and tty != "??":
        print(f"/dev/{tty}")
        raise SystemExit(0)

    try:
        ppid = subprocess.check_output(
            ["ps", "-o", "ppid=", "-p", str(pid)], text=True
        ).strip()
    except Exception:
        break

    if not ppid:
        break
    pid = int(ppid)
PY
}

SESSION_TTY="$(capture_session_tty)"
```

Rules:

- Target `Terminal` only, not every terminal emulator.
- Resolve the owning window from the captured `tty`.
- If `tty` fails but ancestry lookup returns a `tty`, continue with that value instead of reporting failure.
- Run it only after the final user-facing message is ready.
- If Terminal shows the active-process confirmation sheet, accept the default button automatically.
- If the runtime is not hosted in Terminal.app, report the limitation instead of improvising.

## Linux X11 close path

Use this only when the host is Linux with X11 and the user explicitly asks to close the current terminal window or invokes `$agent-session-close` bare.

Preferred capture step before work:

```bash
if [ -n "${WINDOWID:-}" ]; then
  SESSION_WINDOW_ID="$WINDOWID"
elif command -v xdotool >/dev/null 2>&1; then
  SESSION_WINDOW_ID="$(xdotool getactivewindow)"
elif command -v xprop >/dev/null 2>&1; then
  SESSION_WINDOW_ID="$(xprop -root _NET_ACTIVE_WINDOW | awk '{print $5}')"
else
  SESSION_WINDOW_ID=""
fi
```

Preferred close step:

```bash
if [ -n "${SESSION_WINDOW_ID:-}" ] && command -v xdotool >/dev/null 2>&1; then
  xdotool windowclose "$SESSION_WINDOW_ID"
elif [ -n "${SESSION_WINDOW_ID:-}" ] && command -v wmctrl >/dev/null 2>&1; then
  wmctrl -ic "$SESSION_WINDOW_ID"
else
  echo "no-linux-window-close-tool"
  exit 1
fi
```

Rules:

- Support Linux only on X11 with `xdotool` or `wmctrl`.
- Target only the captured window id for this session.
- Do not use the active window at close time.
- Do not claim support for Wayland unless a verified compositor-specific close path is available.
- Run it only after the final user-facing message is ready.

## Response contract

After attempting a close:

- State which agent ids were closed.
- State which requested targets could not be closed and why.
- If the current terminal window was explicitly requested and successfully closed, say that the current terminal window was force-closed.
- If the current session cannot self-terminate, say: `No spawned agents remain open on my side. The current session must be closed by the host or UI.`
- Do not add follow-up offers, optional improvements, or "Do you want me to..." questions after the requested work is complete.
- Do not suggest next steps when this skill is active unless the user explicitly asked for options.
- Do not add follow-up offers, optional improvements, or "Do you want me to..." questions after the requested work is complete.
- Do not suggest next steps when this skill is active unless the user explicitly asked for options.

## Safety checks

- Finish or summarize critical work before closing a spawned agent.
- Prepare the final response before triggering Terminal window close automation.
- Detect the host and capture the stable window handle before doing the requested work.
- Reuse known agent ids from the current thread; do not infer ids from branch names, PR numbers, or filenames.
- If `close_agent` returns an error, surface the exact failure instead of masking it.
- If host window automation fails, surface the exact failure and do not claim the session was closed.

## Examples

- User: `close agent 123`
  Action: call `close_agent` with `123`, then report the result.

- User: `close this instance only`
  Action: do not close unrelated spawned agents. If no session-close tool exists, state the limitation plainly.

- User: `close all the agents you opened for this task`
  Action: close only the tracked ids created for the current task and report each result.

- User: `$agent-session-close`
  Action: treat the invocation itself as an explicit request to close the current session context and force-close the current host terminal window if supported.

- User: `close this codex session and the Terminal window`
  Action: close any explicitly requested spawned agents first, then run the supported host terminal-window close path if available, otherwise state the host limitation plainly.
