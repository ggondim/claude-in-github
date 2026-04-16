# claude-in-github

> A template repo for autonomous agentic development workflows powered by [Claude Code](https://github.com/anthropics/claude-code-action).

Drop this template into any GitHub repository and let Claude agents implement a multi-issue plan from start to finish — no human intervention required after the kickstart.

---

## What you get

Four composable GitHub Actions workflows plus tooling:

| File | Purpose |
|---|---|
| `.github/workflows/claude-plan.yml` | **Planner** — `@plan-agent` turns a feature request into a ready-to-kickstart meta issue (plus task issues) |
| `.github/workflows/claude-meta.yml` | **Orchestrator** — deterministic bash script that tracks progress and dispatches tasks |
| `.github/workflows/claude-task.yml` | **Worker** — implements a single task, creates and auto-merges its PR |
| `.github/workflows/claude-fix.yml` | **Recovery** — `@claude fix` picks up failed tasks, resolves conflicts |
| `.github/prompts/` | Agent system prompts as editable `.md` files — loaded into workflows at runtime |
| `.github/scripts/meta-orchestrate.sh` | Bash script that powers the meta orchestrator (zero LLM) |
| `.github/ISSUE_TEMPLATE/meta-issue.yml` | Structured form for creating meta issues |
| `.github/ISSUE_TEMPLATE/task-issue.yml` | Structured form for creating task issues |
| `scripts/setup.sh` | Validates prerequisites and creates labels |
| `scripts/smoke-test.sh` | End-to-end validator — creates 3 trivial tasks and runs the loop |

## Architectural highlights

- **Meta orchestrator is 100% deterministic** — it's a bash script, not an LLM. Parses a YAML plan block from the meta issue, checks merged PRs, updates checkboxes, dispatches the next wave. Runs in seconds, costs $0.
- **LLMs are only used where they must be** — planning (plan-agent), implementing (task worker), and fixing failures (fix agent). Everything else is bash.
- **Plan agent is optional but end-to-end** — mention `@plan-agent` on a feature request and you get back a full meta issue with task issues created and linked. Model and reasoning level are configurable via directive.
- **Loop closure via `workflow_dispatch`** — the only way to cascade workflow runs from `GITHUB_TOKEN`-authenticated steps, since PR merges and bot comments by `GITHUB_TOKEN` don't trigger workflows.
- **Branch per meta issue** — `meta/<N>` acts as an integration branch for the plan. Task PRs target it. When all tasks are done, a final PR merges `meta/<N>` → `main`.
- **Prompts live in `.github/prompts/`** — edit the `.md` files to tune agent behavior without touching workflow YAML.

---

## How it works

```
   (optional — skip if you write the plan manually)

 Human files a feature request and
 comments @plan-agent [model] [reasoning]
        │
        ▼
   ┌──────────────────────────────────────────┐
   │         claude-plan.yml                   │
   │  1. Parse directive (model, reasoning)    │
   │  2. Plan agent (opus): explore repo,      │
   │     draft meta-issue plan with            │
   │     placeholder task IDs                  │
   │  3. Splitter agent (sonnet): extract      │
   │     task specs into JSONL                 │
   │  4. Bash: create task issues,             │
   │     substitute placeholders with real     │
   │     issue numbers, update issue body,     │
   │     label `meta`+`draft`, assign author   │
   └──────────────┬───────────────────────────┘
                  │  (issue now looks like a manually-authored meta)
                  ▼
 Human reviews the plan, comments @claude to kickstart
        │
        ▼
   ┌──────────────────────────────────────────┐
   │         claude-meta.yml (bash)            │
   │  1. Parse YAML plan from issue body       │
   │  2. Ensure meta/<N> branch exists         │
   │  3. Scan merged PRs → update checkboxes   │
   │  4. Find next ready wave                  │
   │  5. gh workflow run claude-task.yml ───── │──┐
   └──────────────┬───────────────────────────┘  │
                  │                              │
                  ▼                              │
   ┌──────────────────────────────────────────┐  │
   │         claude-task.yml (Claude)          │◄─┘
   │  1. Read issue body (gh issue view)       │
   │  2. Implement tasks                       │
   │  3. Commit + push branch                  │
   │  4. Post-step: auto-create + merge PR     │
   │  5. gh workflow run claude-meta.yml ───── │──┐
   └──────────────────────────────────────────┘  │
                                                 │
                  ┌──────────────────────────────┘
                  ▼
              loops back to claude-meta.yml
                  │
          (when all waves done)
                  ▼
   meta script opens final PR: meta/<N> → main
```

### Branch model

```
main
 ├── meta/17                          ← meta issue #17's integration branch
 │    ├── claude/17-issue-1-xxxxx     ← task #1 branch
 │    ├── claude/17-issue-3-xxxxx     ← task #3 branch
 │    └── claude/17-issue-7-xxxxx     ← task #7 branch
 │
 └── meta/42                          ← another meta issue's branch
      └── claude/42-issue-1-xxxxx
```

- Each **meta issue** gets its own integration branch: `meta/<issue_number>`
- Each **task** branches from the meta branch: `claude/<meta>-issue-<task>-<timestamp>`
- Task PRs target the meta branch (not `main`)
- When all tasks are done, a final PR merges `meta/<N>` → `main`
- Multiple meta issues (multiple implementation plans) can run in parallel

### Meta issue format

The meta issue body can use **one of two formats**:

**1. YAML (preferred — explicit):**

````markdown
## Plan

```yaml
waves:
  - name: Foundation
    tasks: [1]
  - name: Contracts
    tasks: [2]
  - name: Core
    tasks: [3, 5, 6, 7]
```

## Progress

- [ ] #1 Project Bootstrap `P0`
- [ ] #2 Data Model `P0`
- [ ] #3 Router `P0`
...
````

**2. Markdown (fallback — natural headings):**

```markdown
## Wave 1 — Foundation
- [ ] #1 Project Bootstrap `P0`

## Wave 2: Contracts
- [ ] #2 Data Model `P0`

### Wave 3 (Core)
- [ ] #3 Router `P0`
- [ ] #5 Token Management `P0`
```

The orchestrator tries YAML first (parsed with `yq`, pre-installed on runners). If no valid YAML plan block is found, it falls back to markdown parsing (awk-based, lenient). Markdown parsing rules:

- **Wave header:** line starts with `#`, `*`, or the word "Wave"; contains "Wave <N>"; is not a checkbox
- **Task:** checkbox line `- [ ] #N` or `* [ ] #N` (or `[x]` for done), assigned to the most recent wave
- **Wave name:** whatever text follows "Wave <N>", after stripping separators (`:`, `—`, `-`, `(`, etc.)

State (which tasks are done) is always derived from merged PRs — the checkboxes are updated by the orchestrator.

---

## Quick start

### 1. Create your repo from this template

Click **Use this template** at the top of this page → create a new repo.

### 2. Run the setup script

```bash
cd your-new-repo
./scripts/setup.sh
```

The script will:
- Create required labels (`meta`, `smoke-test`, `priority:P0` … `P3`)
- Check that the `CLAUDE_CODE_OAUTH_TOKEN` secret is set
- Check that Actions have the right workflow permissions
- Report anything that needs manual setup

### 3. Manual prerequisites (one-time per repo)

#### a) Install the Claude Code GitHub App

Visit https://github.com/apps/claude and install it on your repo.

#### b) Add the OAuth token secret

Get a token from https://claude.com/oauth/code and add it:

```bash
gh secret set CLAUDE_CODE_OAUTH_TOKEN
```

#### c) Enable write permissions + PR creation for Actions

```bash
gh api repos/OWNER/REPO/actions/permissions/workflow \
  -X PUT \
  -f default_workflow_permissions=write \
  -F can_approve_pull_request_reviews=true
```

If your org blocks this, enable it at `https://github.com/organizations/<ORG>/settings/actions`:
- **Workflow permissions:** Read and write permissions
- ☑️ Allow GitHub Actions to create and approve pull requests

### 4. Validate with a smoke test

```bash
./scripts/smoke-test.sh --cleanup
```

This creates 3 trivial tasks in 2 waves, kickstarts the loop, waits for the final PR, then cleans up all artifacts.

If the smoke test passes, your setup is ready for real work.

---

## Usage

### Option A — Generate the plan with `@plan-agent` (easiest)

1. **File a feature/problem issue** — just describe what you want, no template required. The issue must NOT already have the `meta` label or a `priority:P*` label (those mark existing meta and task issues, respectively).

2. **Comment `@plan-agent`** to trigger the planner. You can pin the model and reasoning budget with a directive:

   ```
   @plan-agent                    → opus + high thinking (defaults)
   @plan-agent sonnet             → sonnet + high thinking
   @plan-agent opus max           → opus + ultrathink
   @plan-agent haiku off          → haiku, no extended thinking
   ```

   Directive tokens: **model** is `opus` | `sonnet` | `haiku`; **reasoning** is `off` | `low` | `medium` | `high` | `max`. Unknown tokens are ignored.

3. **Two agents run in sequence:**
   - **Plan agent** — explores the repo with Read/Glob/Grep and drafts the meta-issue body (Purpose, YAML Plan, Progress, Notes) using placeholder IDs (`T1`, `T2`, ...) for tasks
   - **Splitter agent** — extracts each task spec into structured JSONL

4. **A deterministic bash step** creates the real task issues, substitutes the placeholders with real issue numbers across the YAML and progress sections, updates the original issue body, labels it `meta` + `draft`, and assigns you as the author of the planning request.

5. **Review the plan.** Remove the `draft` label when you're happy (optional), then comment `@claude` on the issue to kickstart the orchestration loop (same as Option B from step 3).

### Option B — Write the plan manually

1. **Create task issues** using the "Task Issue" template. One issue per unit of work. Include acceptance criteria and explicit dependencies.

2. **Create a meta issue** using the "Meta Issue" template. Reference the task issues by number in the YAML plan block.

3. **Kickstart** by commenting on the meta issue:
   ```
   @claude
   ```
   Or assign the meta issue to someone (triggers the same workflow).

4. The orchestrator takes over — creates the branch, dispatches the first wave, tracks progress, advances waves, and opens the final PR automatically.

### Handling failures

When a task worker fails:

1. A notification is posted on both the **meta issue** and the **task issue**.
2. Retry by commenting on the failed task issue:
   ```
   @claude fix
   ```
3. The fix agent picks up the existing partial branch, reads the error context from previous comments, and attempts to complete the task.

### Retrying the orchestrator

If the meta orchestrator itself fails (rare), just comment `@claude` on the meta issue. The script is idempotent — it reads current state from GitHub and picks up where it left off.

### Manually dispatching the orchestrator

You can also trigger the meta workflow directly (no comment needed):

```bash
gh workflow run claude-meta.yml -f meta_issue=17
```

---

## Workflow details

### `claude-plan.yml` — Planner (two-agent)

| | |
|---|---|
| **Triggers** | `@plan-agent` comment on any issue that is NOT labeled `meta` and does NOT have a `priority:P*` label (PR comments ignored) |
| **Implementation** | Two Claude agents + deterministic bash finalization |
| **Agent 1 (Plan)** | Model configurable via directive (default `claude-opus-4-7`). Tools: `Read, Glob, Grep, Write`. Explores the repo and writes the meta-issue body to `/tmp/plan-body.md` |
| **Agent 2 (Splitter)** | `claude-sonnet-4-6`. Tools: `Read, Write`. Extracts task specs into `/tmp/tasks.jsonl` |
| **Finalization** | Bash creates task issues via `gh issue create`, substitutes placeholder IDs with real issue numbers, updates the meta issue body, adds `meta`+`draft` labels, assigns the commenter, replies on the issue |
| **Extended thinking** | Controlled by directive (`off`/`low`/`medium`/`high`/`max`) → injects magic phrase (`think hard` / `think very hard` / `ultrathink`) into the prompt |
| **Timeout** | 30 minutes |
| **Key permissions** | `contents: read`, `issues: write`, `id-token: write` |

### `claude-meta.yml` — Orchestrator (deterministic)

| | |
|---|---|
| **Triggers** | `@claude` comment on meta issue, assign on meta issue, PR merge into `meta/*` (human merges), `workflow_dispatch` (from task worker post-step) |
| **Implementation** | Pure bash script in `.github/scripts/meta-orchestrate.sh` |
| **LLM** | None |
| **Timeout** | 10 minutes |
| **Key permissions** | `contents: write`, `pull-requests: write`, `issues: write`, `actions: write` |

### `claude-task.yml` — Worker

| | |
|---|---|
| **Triggers** | `workflow_dispatch` (from meta script), `@claude` comment on non-meta issue (manual retry) |
| **Implementation** | Claude Code agent with explicit prompt |
| **Model** | `claude-sonnet-4-6` |
| **Timeout** | 60 minutes |
| **Key permissions** | `contents: write`, `pull-requests: write`, `issues: write`, `actions: write` |

### `claude-fix.yml` — Fix agent

| | |
|---|---|
| **Triggers** | `@claude fix` comment on non-meta issue |
| **Implementation** | Claude Code agent, picks up existing partial branch |
| **Model** | `claude-sonnet-4-6` |
| **Timeout** | 60 minutes |
| **Key permissions** | Same as task worker |

---

## Guardrails

| Guardrail | Description |
|---|---|
| **Deterministic meta** | The orchestrator is a bash script — no LLM drift, no probabilistic failures |
| **60-minute timeout** | Task/fix agents have a hard time limit to prevent runaway costs |
| **Failure notifications** | Failures are reported on both the meta and task issues |
| **Fix agent** | `@claude fix` provides semi-automated recovery |
| **Idempotent orchestrator** | Re-running the meta script is always safe — it reads current state from GitHub |
| **Conflict resolution** | Task worker auto-resolves merge conflicts via API merge |
| **Bot allowlist** | Only `claude[bot]` and `github-actions[bot]` can trigger workflows |
| **Loop closure via `workflow_dispatch`** | Reliable cross-workflow triggering from `GITHUB_TOKEN` |

### Known limitations

| Limitation | Workaround |
|---|---|
| `GITHUB_TOKEN`-generated events don't trigger other workflows (neither PR merges nor bot comments) | Use `gh workflow run` (workflow_dispatch IS allowed for `GITHUB_TOKEN`) |
| Workflow validation fails when workflow files change between trigger and execution | Re-trigger via `@claude` comment (uses the latest workflow from `main`) |
| `git` auth not available in post-steps | Post-steps use `gh api` and `gh` CLI instead of `git` commands |
| Git ref replication lag after branch creation | Meta script waits up to 5s; task worker pre-step polls up to 20s |
| Parallel tasks in the same wave may touch shared files | Post-step tries direct merge, falls back to API merge; unresolvable conflicts trigger failure notification |
| P1+ tasks need human review before merging (with branch protection) | The orchestrator comments on P0 tasks mentioning auto-merge; others wait for review |

---

## Configuration

The defaults should work out of the box, but you can customize:

### Edit agent prompts

All agent system prompts live in `.github/prompts/` as `.md` files:

| File | Used by |
|---|---|
| `plan-agent.md` | `claude-plan.yml` — Plan agent (supports `{{THINK_PHRASE}}` placeholder) |
| `splitter-agent.md` | `claude-plan.yml` — Splitter agent |
| `task-worker.md` | `claude-task.yml` — Task worker |
| `fix-agent.md` | `claude-fix.yml` — Fix agent |

Each workflow has a "Load prompt" step that reads the relevant file into `$GITHUB_ENV` at runtime and feeds it into the Claude Code action. Edit the `.md` file and the next workflow run picks it up — no YAML changes needed.

### Change the model

For the **task/fix agents**, edit the `claude_args` line in `claude-task.yml` or `claude-fix.yml`:

```yaml
claude_args: "--model claude-opus-4-5 --allowedTools Bash,Read,Glob,Grep,Write,Edit"
```

**Important:** the model ID must be the full version, not a short alias. `claude-sonnet` does not work — use `claude-sonnet-4-6`.

For the **plan agent**, no YAML edit is needed — pass the directive in the comment:

```
@plan-agent sonnet medium
@plan-agent opus max
@plan-agent haiku off
```

### Change the timeout

Edit `timeout-minutes` at the job level. Max is 360 minutes on GitHub-hosted runners.

### Restrict who can trigger

By default, any `@claude` / `@plan-agent` comment from a repo collaborator triggers workflows. To restrict further, add `allowed_non_write_users` or modify the `if` conditions.

---

## Design decisions

Non-obvious choices documented for future maintainers:

- **Why is the meta orchestrator pure bash instead of an LLM?** The meta does state tracking and dispatching — not creative work. LLMs were probabilistic about parsing wave structure and updating checkboxes. A deterministic script is faster (seconds vs minutes), cheaper ($0 vs ~$0.10 per run), and never drifts.

- **Why YAML for the plan structure?** It's trivially parseable with `yq` (pre-installed on GitHub runners), human-editable, and naturally expresses waves. ASCII dependency graphs were visual but required LLM interpretation.

- **Why `workflow_dispatch` for loop closure?** GitHub Actions deliberately blocks `GITHUB_TOKEN`-generated events from triggering other workflows (to prevent infinite loops). The exception is `workflow_dispatch` and `repository_dispatch`. Bot comments by `GITHUB_TOKEN` are also silent, so we can't use comments for cross-workflow communication.

- **Why a separate meta branch per plan?** Isolation. Multiple plans can run in parallel without stepping on each other. It also makes it trivial to abandon a plan — just delete the branch.

- **Why explicit git commands in the task worker prompt?** Earlier versions said "the workflow handles the PR" and the agent interpreted this as "I don't need to commit/push either." Being explicit eliminates the ambiguity.

- **Why does the task worker poll for branch visibility?** GitHub has eventual consistency on git refs. When the meta script creates `meta/<N>` and immediately dispatches a task worker, the ref may not be visible to `actions/checkout`. Polling handles this.

- **Why `gh api` instead of `git ls-remote` in post-steps?** Git credentials are revoked after the `claude-code-action` step completes, but `GITHUB_TOKEN` still works with the `gh` CLI.

---

## Contributing

Issues and PRs welcome. If you find a new failure mode, please document the root cause in the "Known limitations" table.

## License

MIT
