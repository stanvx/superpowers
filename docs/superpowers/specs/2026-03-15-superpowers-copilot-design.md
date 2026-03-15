# superpowers-copilot Design Spec

**Date:** 2026-03-15  
**Status:** Approved  
**Author:** Trent Stanton

---

## Problem & Goal

Superpowers is a complete software development workflow plugin for coding agents, built around composable skills. The existing plugin targets Claude Code, Cursor, Codex, and Gemini CLI. This fork — `superpowers-copilot` — creates a distributable Copilot CLI plugin that brings all 15 skills to Copilot CLI users, with `/fleet`-native parallel agent dispatch and request-efficient conversational patterns as first-class concerns.

---

## Approach: Layered Fork (Approach C)

A thin adaptation layer covers the 12 simpler skills via a single tool-mapping reference file. The three highest-leverage skills (`dispatching-parallel-agents`, `subagent-driven-development`, `using-superpowers`) get deeper Copilot CLI-native rewrites. A new cross-skill reference (`copilot-workflow.md`) documents `/fleet` patterns, `/research` integration, and request-efficiency principles.

---

## Architecture

### Repository

New repo: `superpowers-copilot` (fork of `superpowers-stanvx`).

```
superpowers-copilot/
├── plugin.json                               ← Copilot CLI plugin manifest
├── AGENTS.md                                 ← Auto-loaded custom instructions
├── README.md                                 ← Updated with Copilot CLI install
├── skills/
│   ├── using-superpowers/
│   │   ├── SKILL.md                          ← Add "In Copilot CLI" section
│   │   └── references/
│   │       ├── copilot-cli-tools.md          ← NEW: tool name mapping
│   │       └── copilot-workflow.md           ← NEW: /fleet, /research, request patterns
│   ├── dispatching-parallel-agents/
│   │   └── SKILL.md                          ← Full /fleet rewrite
│   ├── subagent-driven-development/
│   │   ├── SKILL.md                          ← Copilot CLI task handoff adaptation
│   │   ├── implementer-prompt.md             ← Tool names updated
│   │   ├── spec-reviewer-prompt.md           ← Tool names updated
│   │   └── code-quality-reviewer-prompt.md   ← Tool names updated
│   └── [12 other skills]                     ← Unchanged; benefit from tool mapping
├── hooks/
│   └── hooks.json                            ← COPILOT_PLUGIN_ROOT path variable
├── agents/
│   └── code-reviewer.agent.md                ← Unchanged
└── docs/
    └── superpowers/specs/
        └── 2026-03-15-superpowers-copilot-design.md
```

**Total scope:** 3 new files, 7 modified files.

---

## Components

### 1. `plugin.json` (NEW)

Copilot CLI plugin manifest at the repository root.

```json
{
  "name": "superpowers-copilot",
  "description": "Complete dev workflow for Copilot CLI: TDD, /fleet-native parallel agents, subagent handoffs, debugging, and more",
  "version": "1.0.0",
  "author": { "name": "Trent Stanton" },
  "homepage": "https://github.com/trentstanton/superpowers-copilot",
  "license": "MIT",
  "keywords": ["skills", "tdd", "fleet", "subagents", "debugging", "workflows"]
}
```

### 2. `AGENTS.md` (NEW)

Auto-loaded by Copilot CLI at session start. Single instruction:

> This repo uses the superpowers-copilot plugin. Immediately invoke the `using-superpowers` skill before any response or action.

Safety net for programmatic invocations where the session-start hook may not fire.

### 3. `skills/using-superpowers/references/copilot-cli-tools.md` (NEW)

Tool name mapping table. Loaded by `using-superpowers` so all other skills implicitly benefit.

| Skill references | Copilot CLI tool |
|---|---|
| `Skill` | `skill` (same) |
| `Task` (subagent dispatch) | `task` (same) |
| `Read` / `view` | `view` (same) |
| `Write` / `create` | `create` (same) |
| `Edit` | `edit` (same) |
| `Bash` | `bash` (same) |
| `Grep` | `grep` (same) |
| `Glob` | `glob` (same) |
| `TodoWrite` | `sql` (session database) |
| `WebSearch` | `web_search` |
| `WebFetch` | `web_fetch` |

Note: Copilot CLI tool names are nearly identical to Claude Code. The primary divergence is `TodoWrite` → `sql` (session DB), which only affects the process skills.

### 4. `skills/using-superpowers/references/copilot-workflow.md` (NEW)

Cross-skill reference covering three topics any skill can cite:

**Request efficiency** — Copilot CLI bills per user message (not per token). Skills should guide the agent to ask all clarifying questions up front, batch context generously into subagent `task` calls, and use `/compact` only when approaching context limits rather than proactively.

**`/research` integration** — When `brainstorming` is invoked, an optional pre-flight offer: *"Would a `/research` report on this topic help before we design?"* If accepted, the agent fires `/research TOPIC` and references the report when forming clarifying questions. Opt-in only, not automatic.

**`/fleet` quick-reference** — Condensed parallel dispatch pattern for skills that need to mention fleet without loading the full `dispatching-parallel-agents` skill.

### 5. `skills/using-superpowers/SKILL.md` (MODIFIED)

Adds a **"In Copilot CLI"** section:

- Access skills via the `skill` tool
- Load `copilot-cli-tools.md` for the tool name mapping
- Load `copilot-workflow.md` for `/fleet`, `/research`, and request-efficiency guidance
- Remainder of file unchanged

### 6. `skills/dispatching-parallel-agents/SKILL.md` (MODIFIED — deep rewrite)

The current skill teaches a `Task()` dispatch pattern suited for Claude Code. This is rewritten around `/fleet` as the native primitive.

**Pattern 1 — Direct `/fleet` dispatch** (2–4 independent tasks):
```
/fleet Fix the 3 test failures — Agent 1: fix agent-tool-abort.test.ts (timing issues). Agent 2: fix batch-completion.test.ts (event structure). Agent 3: fix race-conditions.test.ts (async wait).
```
Each fleet agent gets its own context window, works in parallel, returns a summary. The main agent reviews and integrates.

**Pattern 2 — Staged fleet with handoff** (tasks with a dependency seam):
```
Stage 1: /fleet [research + plan tasks in parallel]
Stage 2: /fleet [implementation tasks, using Stage 1 outputs]
```

The decision flowchart structure is preserved from the original; leaf nodes name `/fleet` explicitly instead of `Task()`.

**When NOT to use `/fleet`:**
- Tasks write to the same files (agents would conflict)
- You need to see one result before knowing the next task
- Fewer than 2 independent domains (sequential is simpler)

### 7. `skills/subagent-driven-development/SKILL.md` (MODIFIED)

The three-stage review loop (implementer → spec-reviewer → code-quality-reviewer) maps directly onto Copilot CLI's `task` tool. Logic is unchanged; mechanics are updated.

**Handoff state machine:**
```
Main agent
  └─ task(implementer-prompt.md + task spec + relevant file context)
       └─ Implementer implements, tests, commits, self-reviews
  └─ task(spec-reviewer-prompt.md + spec + diff)
       └─ Returns: APPROVED or issues list
  [if issues] └─ task(implementer, fix-list)
  └─ task(code-quality-reviewer-prompt.md + diff)
       └─ Returns: APPROVED or issues list
  └─ Mark complete in sql todos table
```

**Copilot CLI adaptations:**
- All three subagent prompt files updated with Copilot CLI tool names
- `TodoWrite` → `sql` in all prompts
- SKILL.md gains a request-efficiency note: each `task` call is one request — pass context generously rather than having the subagent call back for clarifications

### 8. Subagent prompt files (MODIFIED)

`implementer-prompt.md`, `spec-reviewer-prompt.md`, `code-quality-reviewer-prompt.md`: update tool references (`TodoWrite` → `sql`, confirm other tool names match `copilot-cli-tools.md`).

### 9. `hooks/hooks.json` (MODIFIED)

Update path variable from `CLAUDE_PLUGIN_ROOT` to `COPILOT_PLUGIN_ROOT`. Event name `SessionStart` is the same in Copilot CLI.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${COPILOT_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ]
  }
}
```

### 10. `README.md` (MODIFIED)

Add Copilot CLI installation section:

```
### GitHub Copilot CLI

/plugin install superpowers-copilot@<marketplace>
```

Plus a brief description of the Copilot CLI-specific features (`/fleet` integration, request-efficient design).

---

## Data Flow

```
Session start
  → AGENTS.md loaded → agent invokes using-superpowers skill
  → using-superpowers loads copilot-cli-tools.md + copilot-workflow.md
  → Agent knows all tool mappings and Copilot CLI patterns

User asks to build something
  → brainstorming skill invoked
  → optional: /research pre-flight
  → clarifying questions (request-efficient: batch up front)
  → design presented and approved
  → writing-plans skill

User says "go"
  → subagent-driven-development skill
  → task(implementer) → task(spec-reviewer) → task(quality-reviewer) → done
  → if independent tasks: dispatching-parallel-agents → /fleet
```

---

## Out of Scope

- Changes to the 12 skills not named above
- New skills (this is a port, not a feature addition)
- Copilot CLI marketplace listing setup (post-implementation step)
- Windows `run-hook.cmd` changes (hook command path update only)

---

## Success Criteria

1. `plugin.json` valid and installable via `/plugin install`
2. All 15 skills accessible via `skill` tool in a Copilot CLI session
3. `dispatching-parallel-agents` skill explicitly guides the agent to use `/fleet`
4. `subagent-driven-development` handoff loop works end-to-end with Copilot CLI `task` tool
5. Session-start hook fires and invokes `using-superpowers` correctly
6. `AGENTS.md` triggers `using-superpowers` in programmatic sessions
