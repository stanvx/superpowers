# Copilot CLI Workflow Patterns

## Request Efficiency

Copilot CLI bills per user message (request), not per token. Design interactions to minimize round-trips:

- **Batch clarifying questions up front.** Ask everything you need to know in one message rather than one question at a time.
- **Pass context generously into subagent `task` calls.** Include the full plan section, relevant file contents, and error messages in the initial dispatch. A subagent that needs to ask for clarification costs an extra request.
- **Use `/compact` reactively, not proactively.** Compact only when approaching context limits, not as a routine step.
- **Prefer depth in one response over back-and-forth.** A thorough first answer is cheaper than a dialogue.

## `/research` Integration

When `brainstorming` is invoked on a topic where deep background would improve the design, offer a `/research` pre-flight:

> *"Would a `/research` report on [TOPIC] help before we design? I can run it now and reference the findings when forming questions."*

This is **opt-in only** — never trigger `/research` automatically.

If the user accepts:
1. Run `/research TOPIC`
2. Reference the findings in the clarifying questions and design sections
3. Cite the report when presenting the design: *"Based on the `/research` findings: ..."*

The `/research` command uses a hard-coded model (Claude Sonnet 4.5) and is not affected by `/model` selection. It produces a saved Markdown report in `~/.copilot/session-state/<session-id>/research/`.

## `/fleet` Quick-Reference

`/fleet` dispatches multiple agents in parallel. Each agent gets its own context window.

**Basic syntax:**
```
/fleet [brief description] — Agent 1: [task with full context]. Agent 2: [task with full context].
```

**When to use:**
- 2+ tasks that write to different files and have no dependency between them
- Research across unrelated domains

**When NOT to use:**
- Tasks write to the same files (agents conflict)
- You need one result before knowing the next task
- Fewer than 2 independent domains

For full `/fleet` patterns including staged dispatch and agent prompt guidelines, invoke `superpowers:dispatching-parallel-agents`.
