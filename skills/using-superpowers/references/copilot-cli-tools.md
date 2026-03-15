# Copilot CLI Tool Name Mapping

Copilot CLI tool names are nearly identical to Claude Code. Three diverge:

| Skill/prompt references this | Copilot CLI tool to use |
|---|---|
| `Skill` | `skill` (same) |
| `Task` (subagent dispatch) | `task` (same) |
| `Read` / `view` / read a file | `view` (same) |
| `Write` / `create` a file | `create` (same) |
| `Edit` | `edit` (same) |
| `Bash` | `bash` (same) |
| `Grep` | `grep` (same) |
| `Glob` | `glob` (same) |
| `TodoWrite` | `sql` — use the session database (`database: "session"`) |
| `WebSearch` | `web_search` |
| `WebFetch` | `web_fetch` |

## Using `sql` instead of `TodoWrite`

`TodoWrite` in Claude Code writes to a managed todo list. In Copilot CLI, use the `sql` tool against the built-in `todos` table:

```sql
-- Create todos
INSERT INTO todos (id, title, description) VALUES
  ('task-1', 'Title here', 'Full description with context');

-- Mark complete
UPDATE todos SET status = 'done' WHERE id = 'task-1';

-- Query what is ready (no pending dependencies)
SELECT t.* FROM todos t
WHERE t.status = 'pending'
AND NOT EXISTS (
    SELECT 1 FROM todo_deps td
    JOIN todos dep ON td.depends_on = dep.id
    WHERE td.todo_id = t.id AND dep.status != 'done'
);
```

The `todos` table is pre-created. Columns: `id TEXT`, `title TEXT`, `description TEXT`, `status TEXT` (pending/in_progress/done/blocked), `created_at`, `updated_at`.
