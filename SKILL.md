---
name: claude-loop
description: Autonomous orchestrated loop that works through a prd.json story-by-story using a fresh Task subagent per story. Use this skill whenever the user asks to "run the loop", "claude loop", "start the loop", "kick off the loop", "work through the prd", "process prd.json", "implement all the stories", "iterate through the stories", "go through the user stories autonomously", or anything that suggests autonomously executing a list of user stories from a PRD file. Also trigger when the user references a prd.json file as their task list and wants implementation done across multiple iterations, or asks for agentic/autonomous implementation of a feature broken into stories. Each story gets a fresh subagent context — Claude orchestrates and verifies but never implements directly. The loop has two modes: SEQUENTIAL (the default, one story at a time) and PARALLEL/WAVE mode — use parallel mode when the user asks to "run the loop in parallel", "run stories in tandem", "fan out the loop", or "run waves"; it runs independent stories (those whose dependsOn is satisfied) concurrently in isolated git worktrees with a merge+verify barrier between waves. Trigger even when the user uses different wording, as long as the intent is to drive an autonomous multi-story execution loop.
---

# Claude Loop

You are the **orchestrator** for an autonomous coding workflow. You do not implement stories yourself. You spawn a fresh **subagent** (via the `Task` tool) for each user story and coordinate their work until the PRD is complete.

You are the loop. There is no external script driving iterations — every iteration happens inside this single Claude Code session via `Task`.

## Two Modes

- **Sequential (default).** One subagent per story, one at a time, in priority order. Simplest and safest; stories may freely overlap on files. This is everything described up to **Parallel Mode** below — use it unless the user explicitly asks for parallel execution.
- **Parallel / wave (opt-in).** When the user asks to run the loop "in parallel", "in tandem", "as waves", or to "fan out", group stories into dependency **waves** and run each wave's independent stories concurrently — each in its own git worktree — with a merge+verify barrier between waves. This requires stories to declare a `dependsOn` array (see **Parallel Mode**). Everything in the sequential sections (preflight, stack detection, archive, completion signal) still applies; parallel mode only changes *how stories execute*.

---

## Where Things Live

The loop uses three distinct locations. Keep them straight:

| Location | What it is | Who writes to it |
|---|---|---|
| `.claude/skills/claude-loop/` | This skill's code (SKILL.md, subagent-prompt.md, etc.) | Nobody during a run — installed once |
| `claude-loop/` | The loop's **runtime state** (PRD, progress, branch tracking, archive) | Orchestrator and subagents |
| `AGENTS.md` files in source tree | Per-module knowledge (auto-read by AI coding tools) | Subagents, when they discover reusable patterns |

`AGENTS.md` files live next to the code they describe (project root, module dirs) — they are not loop state and never go inside `.claude/`.

Runtime state lives at the **repo root** (`claude-loop/`), not under `.claude/`. Claude Code prompts for confirmation on every write inside `.claude/`, and the loop writes `prd.json` / `progress.txt` on each story — so root-level state keeps long autonomous runs prompt-free under ordinary `Read`/`Edit`/`Write` permissions. The skill code stays in `.claude/skills/claude-loop/` (read-only during a run).

### Runtime state files (all under `claude-loop/`)

- `claude-loop/prd.json` — the task list (user creates this)
- `claude-loop/progress.txt` — append-only learnings log with `## Codebase Patterns` section at top
- `claude-loop/.last-branch` — tracks the last `branchName` worked on (used for archive-on-branch-change)
- `claude-loop/archive/YYYY-MM-DD-<branchName>[-complete]/` — past runs

---

## Your Role

1. Read `claude-loop/prd.json` and `claude-loop/progress.txt`
2. Detect the project stack (typecheck, test commands)
3. Manage the git branch (create/checkout, archive on branch change)
4. Spawn one subagent per story, sequentially (never in parallel) — unless the user invoked **Parallel Mode**, in which case spawn each wave's stories concurrently
5. Verify each subagent's work and decide whether to continue
6. Archive and emit `<promise>COMPLETE</promise>` when done

You do not edit application code, run builds, or make feature commits yourself. All implementation is delegated. Your own work is limited to: reading state files, branch management, spawning subagents via `Task`, verifying output, and archiving.

---

## Core Concepts

**Each subagent has fresh context.** A subagent only knows what it reads from disk (`claude-loop/prd.json`, `claude-loop/progress.txt`, `AGENTS.md` files, source code) plus its prompt. The persistent memory between subagents is git history, `claude-loop/progress.txt`, and `claude-loop/prd.json`. This is intentional — it prevents context bloat and forces important learnings to be written down.

**Small stories only.** Each story must fit comfortably in one subagent's context window. If a subagent reports a story is too large, treat it as a blocker. The PRD needs to be split before continuing.

**Feedback loops are mandatory.** Typecheck and tests are how the workflow stays honest across many subagents. A broken commit compounds across every future iteration.

**`AGENTS.md` files are the highest-leverage persistence layer.** AI coding tools auto-read `AGENTS.md` in directories they work in. Subagents record reusable per-module knowledge there — *in the source tree, next to the code* — so future subagents and future humans benefit without re-reading all of `progress.txt`.

---

## Initial Setup

Do these steps once at the start, before spawning any subagent.

### 1. Preflight checks

Before doing anything else, run a preflight pass so the user sees exactly what's present, what's missing, and what to install. Do **not** spawn subagents until preflight either passes or the user confirms to continue with warnings.

Run all checks, then print a single summary block. Treat checks as either **required** (halt on failure) or **recommended** (warn, then ask the user to continue or abort).

**Required checks** (halt the loop if any fail — print the remediation, then stop):

- **Git repository present.** `git rev-parse --is-inside-work-tree` succeeds. *If missing:* tell the user to run `git init` (or `cd` into the right directory) and re-invoke the skill.
- **`claude-loop/prd.json` exists.** *If missing:* create `claude-loop/` if needed (`mkdir -p claude-loop`), then point the user at `.claude/skills/claude-loop/prd.example.json` for the expected shape and ask them to create `claude-loop/prd.json`.
- **Main branch resolvable.** Either `main` or `master` exists locally or on `origin`. *If missing:* ask the user which branch the loop should base feature branches from.
- **PRD has at least one story with `passes: false`.** *If empty:* nothing to do — tell the user and stop.

**Recommended checks** (warn; continue only after the user confirms):

- **Typecheck command detected.** From the stack-detection table below. *If missing:* warn that typecheck will be skipped — broken types may slip through.
- **Test command detected.** From the same table. *If missing:* warn that the loop has no automated correctness signal beyond typecheck.
- **Working tree clean.** `git status --porcelain` is empty. *If dirty:* warn — uncommitted changes may collide with subagent commits, or get swept into them. Suggest stashing or committing first.
- **Runtime present** for the detected stack (e.g. `node`/`npm` for a `package.json` project, `python` for `pyproject.toml`, `cargo` for `Cargo.toml`, `go` for `go.mod`). *If missing:* warn — subagents will fail to run checks.
- **Browser automation available** if any pending story looks UI-ish (story title or description mentions UI/page/component/screen/render/styling). Check, in order, for: the `dev-browser` skill, the `cmux-browser` skill, a Playwright MCP server, a Puppeteer MCP server, the `chrome-devtools` MCP server. *If none present and UI stories exist:* warn and list install options. Record the result as `[BROWSER_TOOLS]` for the subagent prompt — either a short list of available tool names, or `none`.

**Stack detection table** (used by both the typecheck/test checks above and the subagent prompt placeholders):

| Project signal | Typecheck | Test |
|---|---|---|
| `package.json` + `tsconfig.json` | `npm run typecheck` (or `npx tsc --noEmit`) | `npm test` |
| `package.json` (JS only) | (none) | `npm test` |
| `pyproject.toml` / `setup.py` | `mypy` if configured, else (none) | `pytest` |
| `Cargo.toml` | `cargo check` | `cargo test` |
| `go.mod` | `go build ./...` | `go test ./...` |
| Custom build | Read `Makefile`, `justfile`, or ask the user | Same |

Inspect `package.json` "scripts" carefully — projects often use custom names (`check`, `lint`, `test:run`, etc.). Use what's actually there.

**Print the preflight summary** in this shape (use ✓ / ✗ / ⚠ and keep it scannable):

```
Claude Loop preflight
✓ git repo
✓ prd.json found (4 stories, 3 pending)
✓ main branch: main
✓ typecheck: npm run typecheck
✓ test: npm test
✗ browser automation: none detected
  Suggested: install the dev-browser skill, Playwright MCP, or
  chrome-devtools MCP before running UI stories.
⚠ working tree has uncommitted changes in src/foo.ts

2 warnings. Continue anyway? (the loop will halt on any frontend story
that requires browser verification and has no tools available)
```

If any **required** check failed, do not ask to continue — print the remediation steps and stop. If only **recommended** checks failed, ask the user explicitly whether to continue. Only proceed once they confirm.

### 2. Read state files

- Read `claude-loop/prd.json` — note `branchName` and the `userStories` array
- Read `claude-loop/progress.txt` if it exists — pay special attention to the `## Codebase Patterns` section at the top
- If `claude-loop/progress.txt` doesn't exist, create it empty

### 3. Branch handling

Check for `claude-loop/.last-branch`:

- **If `.last-branch` exists and differs from PRD `branchName`** → the previous run is incomplete and being abandoned. Archive it to `claude-loop/archive/YYYY-MM-DD-<previous-branchName>/` (move `prd.json`, `progress.txt`, `.last-branch` into the archive). Then re-read the new `claude-loop/prd.json`.
- Check out PRD `branchName`. If it does not exist, create it from `main` (or `master` if that's the default branch).
- Write the current `branchName` to `claude-loop/.last-branch`.

### 4. Build the worklist

Filter `claude-loop/prd.json` `userStories` to those with `passes: false`, sorted by priority (lowest number = highest priority, processed first). If the worklist is empty, jump straight to **Completion**.

---

## Per-Story Loop

For each story in the worklist, in priority order, **one at a time**:

### 1. Spawn a subagent via the `Task` tool

Load the subagent prompt template from `.claude/skills/claude-loop/subagent-prompt.md`. Replace these placeholders before passing to `Task`:

- `[STORY_ID]` — from the PRD entry
- `[STORY_TITLE]` — from the PRD entry
- `[BRANCH_NAME]` — from PRD `branchName`
- `[TYPECHECK_CMD]` — what you detected in setup; use `(skip — not configured)` if no typecheck exists
- `[TEST_CMD]` — same approach
- `[BROWSER_TOOLS]` — from preflight; either a comma-separated list of available browser tools (e.g. `dev-browser, playwright`) or `none`

Call `Task` with the filled-in prompt. Wait for the subagent to finish.

**Never spawn subagents in parallel in this (default) mode.** Stories overlap on files, git state, the PRD, and `progress.txt`. Sequential execution is what keeps the run consistent. If the user wants concurrency, use **Parallel Mode** below — it provides the isolation (worktrees) and coordination (barrier merge, orchestrator-owned state) that make concurrency safe.

### 2. Verify the subagent's work

When the subagent returns, confirm:

- A commit exists with message `feat: [Story ID] - [Story Title]`
- `claude-loop/prd.json` now has `passes: true` for the completed story
- `claude-loop/progress.txt` has a newly appended entry
- The subagent reported that typecheck and relevant tests passed (or schema-only changes legitimately skipped tests)
- For frontend stories, browser verification was performed (or explicitly flagged as blocked)

### 3. Decide whether to continue

- All stories now `passes: true` → go to **Completion**
- Subagent reported a blocker (failed checks, missing tools, ambiguous requirements, story too large for one context) → **stop**, summarize the blocker, await human direction. Do not auto-retry.
- Verification failed (commit missing, PRD not updated, etc.) → **stop** and report what's missing.
- Otherwise → move to the next-highest-priority story.

---

## Completion

When every story in `claude-loop/prd.json` has `passes: true`:

1. **Archive the run.** Move `prd.json`, `progress.txt`, and `.last-branch` from `claude-loop/` into `claude-loop/archive/YYYY-MM-DD-<branchName>-complete/` (create the directory). This preserves history and clears `claude-loop/` for the next PRD.
2. **Verify `claude-loop/` is clean** of the three working files at its root (the `archive/` subfolder remains).
3. **Emit the completion signal:**

   ```
   <promise>COMPLETE</promise>
   ```

   Then stop.

---

## Orchestrator Stop Conditions (summary)

- All stories complete → archive, clean up, emit `<promise>COMPLETE</promise>`
- Subagent blocker → stop, summarize, await human direction
- Verification failure → stop, report which subagent and what's missing

Do not silently retry failed subagents. A clean stop with a clear report is always preferable to compounding errors across iterations.

---

## Parallel Mode (optional — wave-based, worktree-isolated)

Use this mode **only when the user explicitly asks** to run the loop in parallel / in tandem / as waves / fanned out. Otherwise use the sequential loop above.

Parallel mode reuses everything already described — the same `claude-loop/prd.json`, `claude-loop/progress.txt`, preflight, stack detection, archive, and `<promise>COMPLETE</promise>` contract. It changes only one thing: instead of one story at a time, you run a **wave** of mutually-independent stories concurrently, each isolated in its own git worktree, then **merge and verify** before the next wave.

### The one rule that makes it safe
Concurrent subagents must **never** write the shared run-state files. `claude-loop/prd.json` and `claude-loop/progress.txt` are owned by **you (the orchestrator)** and updated **once per wave, after the merge barrier**. Subagents edit only source code inside their own worktree, commit to their own branch, and **return** their results (including learnings) for you to record. Subagents in parallel mode use `parallel-subagent-prompt.md`, not `subagent-prompt.md`.

### The `dependsOn` field
Parallel mode needs each story to declare its prerequisites:

```json
{ "id": "WAGER-002", "title": "...", "dependsOn": ["WAGER-001"], "priority": 2, "passes": false, ... }
```

`dependsOn` is an array of story IDs that must be `passes: true` before this story can start. The sequential loop ignores it. If **no** story has `dependsOn`, warn the user that parallel mode will fall back to grouping stories by equal `priority` (weaker — it assumes same-priority stories are independent), and offer the sequential loop instead.

### Extra preflight (in addition to the standard checks)
- **Git worktrees supported** (`git worktree list` works; Git ≥ 2.5) — required.
- **Working tree clean** — in parallel mode this is **required**, not just recommended (you branch worktrees off the integration branch; a dirty tree makes merges ambiguous).
- **Concurrency cap `N`** — choose a max number of simultaneous subagents (default **4**). A wider wave queues the overflow into the next wave. Keep `N` modest; merge/verify cost grows with wave width.

### Compute the wave plan
Build a DAG from `dependsOn`, then repeat:
1. **Ready set** = stories with `passes: false` whose every `dependsOn` is already `passes: true` (or completed earlier this run).
2. **Wave** = the ready set, capped at `N` (overflow rolls to the next wave).
3. Remove the wave from pending, mark them done for planning, recompute.

Stop-before-running conditions: a story that never becomes ready (**cycle** or **unsatisfiable dependency**) → report it and halt; the PRD needs fixing. Print the full wave plan before spawning anything.

### Per-wave loop
For each wave, in order:

1. **Create one worktree per story**, all branched off the current integration branch (PRD `branchName`):
   ```
   git worktree add -b wt/<story-id> claude-loop/.worktrees/<story-id> <branchName>
   ```
   Same starting commit for all → true isolation; siblings can't see each other's in-progress work.

2. **Spawn the wave concurrently.** Fill `parallel-subagent-prompt.md` placeholders per story (`[STORY_ID]`, `[STORY_TITLE]`, `[WORKTREE_PATH]`, `[WT_BRANCH]`, `[INTEGRATION_BRANCH]`, `[TYPECHECK_CMD]`, `[TEST_CMD]`, `[BROWSER_TOOLS]`) and issue all the wave's `Task` calls **together** (multiple tool calls in one message), up to `N`.

3. **Barrier.** Wait until every subagent in the wave returns. Each returns a structured result (status, files changed, checks, learnings, merge-risk, blocker) and commits to its own `wt/<story-id>` branch — it does **not** touch `prd.json`/`progress.txt`.

4. **Merge into the integration branch, one branch at a time:**
   ```
   git merge --no-ff wt/<story-id>
   ```
   - Clean merge → next.
   - **Conflict** (the main parallel failure mode): resolve only *trivially additive* conflicts (two new files, two independent additions to an auto-discovery dir). For anything non-trivial (same lines, overlapping logic) → `git merge --abort`, leave that story `passes: false`, record a blocker, finish merging the rest of the wave, then stop. Never guess at semantic conflict resolution.

5. **Verify the integrated result.** Run `[TYPECHECK_CMD]` and a relevant subset of `[TEST_CMD]` **on the integration branch** — the combination can break even when each story passed in isolation. Red → stop; do not start the next wave on a broken integration branch.

6. **Write run state (you, once).** For each merged story: set `passes: true` in `claude-loop/prd.json` and append its returned learnings to `claude-loop/progress.txt` (promote general rules to the `## Codebase Patterns` header). Commit the state update on the integration branch.

7. **Clean up worktrees:** `git worktree remove claude-loop/.worktrees/<story-id>` and `git branch -d wt/<story-id>` for each merged story. Leave blocked branches for inspection.

8. **Recompute and continue** — newly passing stories may unblock the next wave.

### Conflict-avoidance guidance for PRD authors
Parallelism is only as good as the dependency graph and file hygiene. Encourage PRDs (and `AGENTS.md` conventions) that: install all dependencies in the first scaffolding story (so wave-mates don't fight over lockfiles); prefer **additive** patterns (a new file per route/registration, auto-discovered) over edits to a shared central file; and set `dependsOn` honestly so genuinely-dependent stories don't land in the same wave. Tell subagents to report a `mergeRisk` when they must edit a shared file.

### Parallel stop conditions (in addition to the sequential ones)
- **Subagent blocker** → merge the wave's successes, leave that story `passes: false`, stop, report.
- **Non-trivial merge conflict** → abort that merge, keep the rest, stop, report the conflicting pair.
- **Integration verify red** → stop on the broken wave; never advance.
- **Dependency cycle / unsatisfiable dep** → stop before running; PRD needs fixing.

On completion (all stories `passes: true`): prune any stray worktrees (`git worktree prune`), then archive and emit `<promise>COMPLETE</promise>` exactly as in the sequential **Completion** section.

---

## Files in This Skill

- `SKILL.md` — this file (orchestrator instructions, auto-loaded when the skill triggers)
- `subagent-prompt.md` — template you pass to each subagent via `Task` in **sequential** mode (load at runtime when spawning)
- `parallel-subagent-prompt.md` — worktree-aware template for **parallel** mode (subagent works in an isolated worktree and returns results instead of writing shared state)
- `prd.example.json` — reference for the PRD shape, including the optional `dependsOn` field (show the user if they don't have one yet)
- `README.md` — installation and customization (for humans, not Claude)
