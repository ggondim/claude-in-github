Extract each task spec from the plan into JSONL so a deterministic bash
step can create (or update / close) real GitHub issues.

## Input
- `/tmp/plan-body.md` — a plan document with a `## Tasks` section. Each
  task is a `### <ref> — <title> \`priority:PN\`` block containing Summary /
  Tasks / Acceptance Criteria / (optional) References.
- `/tmp/existing-tasks.json` — **present only on revisions**. An array of
  `{"number": <int>, "title": "…", "body": "…"}` for every task issue
  currently in the plan. Use this as reference when emitting bodies for
  preserved tasks (tasks whose heading uses a real integer number) so the
  refreshed body is accurate.

## Output
Write `/tmp/tasks.jsonl` — one JSON object per line, one per task, in the
order they appear in the plan:

```
{"ref":"T1","title":"<title>","body":"<markdown>","labels":["priority:P0"]}    ← new task
{"ref":15,"title":"<title>","body":"<markdown>","labels":["priority:P0"]}       ← preserved
```

## Rules
- `ref` — copy **exactly** what's in the `### <ref> —` heading of each
  task in `/tmp/plan-body.md`. If the heading uses a `Tn` placeholder,
  emit `ref` as the string `"T1"` / `"T2"` / etc. If the heading uses an
  integer (e.g. `### 15 —`), emit `ref` as an unquoted integer `15`. Do
  **not** invent or translate — the plan-agent has already chosen identity;
  you are mechanically extracting.
- `title` — the text after `<ref> —` and before the priority backtick span.
- `body` — markdown matching the task-issue template:

  ```markdown
  ## Summary

  <summary text>

  ## Tasks

  - [ ] ...

  ## Acceptance Criteria

  - [ ] ...

  ## References

  <text or omit this section>
  ```

- `labels` — extract from `` `priority:PN` `` in the heading → `["priority:PN"]`.
  If no priority backtick is present, use `[]`.
- Each line must be a single valid JSON object. Escape newlines inside
  string values as `\n`. Do not split one object across multiple lines.
- Write ONLY the JSONL content to `/tmp/tasks.jsonl`. No preamble, no
  trailing prose, no code fences.
- You are NOT asked to diff, match, or decide which tasks correspond to
  which existing issues — the plan-agent's choice of integer vs `Tn` in the
  heading is the sole identity signal. Pass it through faithfully.
- Do NOT run `git`, `gh`, or create issues directly.
