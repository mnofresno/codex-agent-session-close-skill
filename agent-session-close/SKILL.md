---
name: "agent-session-close"
description: "Use when the user asks to close a spawned agent, terminate an agent session, end the current Codex instance, or cleanly shut down agents without affecting unrelated work. Prefer explicit agent IDs and state clearly when the current or root session cannot self-terminate with available tools."
---

# Agent Session Close

Use this skill when the user wants an agent or session closed cleanly and safely.

## Core rule

Close only the exact agent or session the user requested. Never guess.

## What this skill can do

- Close a spawned sub-agent when you have its agent id.
- Close multiple spawned sub-agents when the requested target set is explicit.
- Cleanly report that the current or root session cannot self-terminate when the runtime exposes no session-close tool.

## What this skill must not pretend to do

- Do not claim the current main conversation was closed unless a real session-close tool exists in the current runtime.
- Do not close unrelated agents just because they look stale.
- Do not use repo actions, shell exits, or destructive cleanup as a fake replacement for closing an agent session.

## Target selection

1. If the user provides an agent id, close exactly that id.
2. If the user asks to close "the agent from this task" and you created exactly one spawned agent for the task, close that one.
3. If the user asks to close the current instance only, do not close any other agents unless they explicitly asked for them too.
4. If the target is ambiguous and there is real risk of closing the wrong thing, ask one short question.

## Tool workflow

1. Prefer `close_agent` for spawned agents.
2. If the environment exposes a true session-level close tool, use it only for the exact requested session.
3. If no self-close tool exists for the current session, say so plainly and stop after a concise final handoff.

## Response contract

After attempting a close:

- State which agent ids were closed.
- State which requested targets could not be closed and why.
- If the current session cannot self-terminate, say: `No spawned agents remain open on my side. The current session must be closed by the host or UI.`

## Safety checks

- Finish or summarize critical work before closing a spawned agent.
- Reuse known agent ids from the current thread; do not infer ids from branch names, PR numbers, or filenames.
- If `close_agent` returns an error, surface the exact failure instead of masking it.

## Examples

- User: `close agent 123`
  Action: call `close_agent` with `123`, then report the result.

- User: `close this instance only`
  Action: do not close unrelated spawned agents. If no session-close tool exists, state the limitation plainly.

- User: `close all the agents you opened for this task`
  Action: close only the tracked ids created for the current task and report each result.
