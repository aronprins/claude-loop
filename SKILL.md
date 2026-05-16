---
name: claude-loop
description: Autonomous orchestrated loop that works through a prd.json story-by-story using a fresh Task subagent per story. Use this skill whenever the user asks to "run the loop", "claude loop", "start the loop", "kick off the loop", "work through the prd", "process prd.json", "implement all the stories", "iterate through the stories", "go through the user stories autonomously", or anything that suggests autonomously executing a list of user stories from a PRD file. Also trigger when the user references a prd.json file as their task list and wants implementation done across multiple iterations, or asks for agentic/autonomous implementation of a feature broken into stories. Each story gets a fresh subagent context — Claude orchestrates and verifies but never implements directly. Trigger even when the user uses different wording, as long as the intent is to drive an autonomous multi-story execution loop.
---

# Claude Loop

You are the **orchestrator** for an autonomous coding workflow. You do not implement stories yourself. You spawn a fresh **subagent** (via the `Task` tool) for each user story and coordinate their work until the PRD is complete.

You are the loop. There is no external script driving iterations — every iteration happens inside this single Claude Code session via `Task`.

---

## Where Things Live

The loop uses three distinct locations. Keep them straight:

| Location | What it is | Who writes to it |
|---|---|---|
| `.claude/skills/claude-loop/` | This skill's code (SKILL.md, subagent-prompt.md, etc.) | Nobody during a run — installed once |
| `.claude/claude-loop/` | The loop's **runtime state** (PRD, progress, branch tracking, archive) | Orchestrator and subagents |
| `AGENTS.md` files in source tree | Per-module knowledge (auto-read by AI coding tools) | Subagents, when they discover reusable patterns |

`AGENTS.md` files live next to the code they describe (project root, module dirs) — they are not loop state and never go inside `.claude/`.

### Runtime state files (all under `.claude/claude-loop/`)

- `.claude/claude-loop/prd.json` — the task list (user creates this)
- `.claude/claude-loop/progress.txt` — append-only learnings log with `## Codebase Patterns` section at top
- `.claude/claude-loop/.last-branch` — tracks the last `branchName` worked on (used for archive-on-branch-change)
- `.claude/claude-loop/archive/YYYY-MM-DD-<branchName>[-complete]/` — past runs

---

## Your Role

1. Read `.claude/claude-loop/prd.json` and `.claude/claude-loop/progress.txt`
2. Detect the project stack (typecheck, test commands)
3. Manage the git branch (create/checkout, archive on branch change)
4. Spawn one subagent per story, sequentially (never in parallel)
5. Verify each subagent's work and decide whether to continue
6. Archive and emit `<promise>COMPLETE</promise>` when done

You do not edit application code, run builds, or make feature commits yourself. All implementation is delegated. Your own work is limited to: reading state files, branch management, spawning subagents via `Task`, verifying output, and archiving.

---

## Core Concepts

**Each subagent has fresh context.** A subagent only knows what it reads from disk (`.claude/claude-loop/prd.json`, `.claude/claude-loop/progress.txt`, `AGENTS.md` files, source code) plus its prompt. The persistent memory between subagents is git history, `.claude/claude-loop/progress.txt`, and `.claude/claude-loop/prd.json`. This is intentional — it prevents context bloat and forces important learnings to be written down.

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
- **`.claude/claude-loop/prd.json` exists.** *If missing:* create `.claude/claude-loop/` if needed (`mkdir -p .claude/claude-loop`), then point the user at `.claude/skills/claude-loop/prd.example.json` for the expected shape and ask them to create `.claude/claude-loop/prd.json`.
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

- Read `.claude/claude-loop/prd.json` — note `branchName` and the `userStories` array
- Read `.claude/claude-loop/progress.txt` if it exists — pay special attention to the `## Codebase Patterns` section at the top
- If `.claude/claude-loop/progress.txt` doesn't exist, create it empty

### 3. Branch handling

Check for `.claude/claude-loop/.last-branch`:

- **If `.last-branch` exists and differs from PRD `branchName`** → the previous run is incomplete and being abandoned. Archive it to `.claude/claude-loop/archive/YYYY-MM-DD-<previous-branchName>/` (move `prd.json`, `progress.txt`, `.last-branch` into the archive). Then re-read the new `.claude/claude-loop/prd.json`.
- Check out PRD `branchName`. If it does not exist, create it from `main` (or `master` if that's the default branch).
- Write the current `branchName` to `.claude/claude-loop/.last-branch`.

### 4. Build the worklist

Filter `.claude/claude-loop/prd.json` `userStories` to those with `passes: false`, sorted by priority (lowest number = highest priority, processed first). If the worklist is empty, jump straight to **Completion**.

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

**Never spawn subagents in parallel.** Stories overlap on files, git state, the PRD, and `progress.txt`. Sequential execution is what keeps the run consistent.

### 2. Verify the subagent's work

When the subagent returns, confirm:

- A commit exists with message `feat: [Story ID] - [Story Title]`
- `.claude/claude-loop/prd.json` now has `passes: true` for the completed story
- `.claude/claude-loop/progress.txt` has a newly appended entry
- The subagent reported that typecheck and relevant tests passed (or schema-only changes legitimately skipped tests)
- For frontend stories, browser verification was performed (or explicitly flagged as blocked)

### 3. Decide whether to continue

- All stories now `passes: true` → go to **Completion**
- Subagent reported a blocker (failed checks, missing tools, ambiguous requirements, story too large for one context) → **stop**, summarize the blocker, await human direction. Do not auto-retry.
- Verification failed (commit missing, PRD not updated, etc.) → **stop** and report what's missing.
- Otherwise → move to the next-highest-priority story.

---

## Completion

When every story in `.claude/claude-loop/prd.json` has `passes: true`:

1. **Archive the run.** Move `prd.json`, `progress.txt`, and `.last-branch` from `.claude/claude-loop/` into `.claude/claude-loop/archive/YYYY-MM-DD-<branchName>-complete/` (create the directory). This preserves history and clears `.claude/claude-loop/` for the next PRD.
2. **Verify `.claude/claude-loop/` is clean** of the three working files at its root (the `archive/` subfolder remains).
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

## Files in This Skill

- `SKILL.md` — this file (orchestrator instructions, auto-loaded when the skill triggers)
- `subagent-prompt.md` — template you pass to each subagent via `Task` (load at runtime when spawning)
- `prd.example.json` — reference for the PRD shape (show the user if they don't have one yet)
- `README.md` — installation and customization (for humans, not Claude)
