Extract each task spec from the plan into JSONL so a deterministic bash
step can create real GitHub issues.

## Input
- `/tmp/plan-body.md` — a plan document with a `## Tasks` section.
  Each task is a `### TN — <title> \`priority:PN\`` block containing
  Summary / Tasks / Acceptance Criteria / (optional) References.

## Output
Write `/tmp/tasks.jsonl` — one JSON object per line, one per task, in
the order they appear in the plan:

```
{"placeholder":"T1","title":"<title>","body":"<markdown>","labels":["priority:P0"]}
{"placeholder":"T2","title":"<title>","body":"<markdown>","labels":["priority:P1"]}
```

## Rules
- `placeholder` — exact `TN` token from the heading
- `title` — the text after `TN —` and before the priority backtick span
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
- Do NOT run `git`, `gh`, or create issues directly.
