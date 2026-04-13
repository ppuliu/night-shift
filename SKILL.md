---
name: night-shift
description: >
  Autonomous agent that runs 24/7 while the human is away. Activate with "/night-shift start"
  or phrases like "start night shift", "going to sleep", "take over for the night", "keep working
  while I'm away". The agent scans the session, codebase, and specs, proposes goals, gets
  confirmation, then enters an autonomous loop: plan → adversarial review → execute → code review →
  iterate until clean. Writes a handoff note when done. Use "/night-shift end" or "end night shift"
  to stop early. Use this skill any time the user wants unattended autonomous work, overnight runs,
  or asks you to "keep going" without them.
---

# Night Shift

You are entering autonomous mode. The human is stepping away — sleeping, taking a break, or
otherwise unavailable. Your job is to make meaningful progress on their codebase without asking
any questions after the initial goal confirmation.

This skill has two modes:
- **Start** (`/night-shift start` or trigger phrases): Begin autonomous work
- **End** (`/night-shift end` or "end night shift"): Stop work and write handoff

## Starting the Night Shift

### Phase 0: Pre-flight Checks

**Must not be on `main` or `master`. Any other branch is fine.**

1. **Check the current branch:**
   - Run: `git branch --show-current`
   - If on `main` or `master`, **stop** and tell the user:
     "You're on `main`. Please switch to a feature branch first
     (e.g., `git checkout -b feat/my-feature`) so night shift work stays isolated."
   - If on any other branch (feature branch, existing night-shift branch, etc.),
     proceed — the agent will continue working on this branch.

2. **Check for stale state from a previous run:**
   - If `.night-shift/` directory exists in the repo root, a previous night shift
     either crashed or wasn't cleaned up properly.
   - First, kill any stale caffeinate process:
     ```bash
     OLD_PID=$(cat .night-shift/caffeinate.pid 2>/dev/null)
     if [ -n "$OLD_PID" ] && ps -p "$OLD_PID" -o comm= 2>/dev/null | grep -q caffeinate; then
       kill "$OLD_PID" 2>/dev/null
     fi
     ```
   - Tell the user: "Found leftover state from a previous night shift (and stopped
     its caffeinate process). Want me to clean it up and start fresh?"
   - On confirmation, `rm -rf .night-shift/` and continue.
   - If the user says no, stop — they may want to investigate.

3. **Exclude `.night-shift/` from git** before creating any files there. This uses the
   repo-local exclude file (does NOT modify any tracked files):
   ```bash
   grep -qxF '.night-shift/' .git/info/exclude 2>/dev/null || echo '.night-shift/' >> .git/info/exclude
   ```
   This MUST happen before `mkdir -p .night-shift` or any PID file write, otherwise
   `git status` will report `.night-shift/` as untracked and the clean-tree check will fail.

4. **Prevent the computer from sleeping (macOS):**

   Use a **bounded-timeout caffeinate** that auto-expires if the agent dies. This is
   critical — an unbounded caffeinate left behind keeps the Mac awake forever.

   On non-macOS systems (no `caffeinate` command), skip this step entirely.

   **Starting caffeinate** (stale state is already cleaned up, so `.night-shift/` is
   guaranteed to not exist at this point):
   ```bash
   mkdir -p .night-shift
   caffeinate -i -t 2700 &
   echo $! > .night-shift/caffeinate.pid
   test -s .night-shift/caffeinate.pid || echo "WARNING: failed to save caffeinate PID"
   ```

   Tell the user: "Starting `caffeinate` to prevent your Mac from sleeping during
   the night shift. It auto-expires every 45 minutes and gets renewed throughout the
   run. If I crash or get disconnected, your Mac will resume normal sleep within 45
   minutes at most."

   **Renewal — the `caffeinate-renew` pattern:**
   Run this before every potentially long phase: Step 1 (Plan), Step 3 (Execute),
   Step 4 (each Code Review round), and Step 5 (Validate). This means caffeinate
   is refreshed multiple times per goal, not just at goal boundaries.

   ```bash
   OLD_PID=$(cat .night-shift/caffeinate.pid 2>/dev/null)
   if [ -n "$OLD_PID" ] && ps -p "$OLD_PID" -o comm= 2>/dev/null | grep -q caffeinate; then
     kill "$OLD_PID" 2>/dev/null
   fi
   caffeinate -i -t 2700 &
   echo $! > .night-shift/caffeinate.pid
   ```

   The `ps -p ... -o comm=` check ensures we only kill a process that is actually
   `caffeinate` — not an unrelated process that reused the PID after the old lease
   expired.

   **Cleanup — the `caffeinate-stop` pattern:**
   Run on any controlled exit (normal completion, early stop, drift failure):
   ```bash
   OLD_PID=$(cat .night-shift/caffeinate.pid 2>/dev/null)
   if [ -n "$OLD_PID" ] && ps -p "$OLD_PID" -o comm= 2>/dev/null | grep -q caffeinate; then
     kill "$OLD_PID" 2>/dev/null
   fi
   rm -f .night-shift/caffeinate.pid
   ```

   **On uncontrolled exit** (crash, OOM, context exhaustion, user kills agent):
   caffeinate auto-expires after at most 45 minutes. No manual intervention needed.

5. Record the current branch as `BRANCH`: `git branch --show-current`
6. Record the current commit as `BASE_COMMIT`: `git rev-parse HEAD`
7. Ensure the working tree is clean: `git status --short`
   - If there are uncommitted changes, ask the user to commit or stash them before starting.
     This is the only other question allowed besides goal confirmation.
8. Create the state directory: `mkdir -p .night-shift/`
9. Initialize the run state file (see "Run State Persistence" below)

**State files live in `.night-shift/`** in the repo root. This means:
- The user can open `.night-shift/state.json` or `.night-shift/plan.md` in their
  editor at any time to see what the agent is doing.
- The `.git/info/exclude` entry prevents these files from appearing in `git status`
  or being staged. This avoids modifying any tracked file like `.gitignore`.
- Codex can see `.night-shift/plan.md` via `--scope working-tree` for reviews.

All night shift work happens on the current branch. The human can review the commits,
revert individual ones, or reset the branch.

### Phase 1: Reconnaissance (2-3 minutes)

Scan these sources to understand what needs doing, **in strict priority order**.
The conversation history is the most important signal — it tells you what the user
actually cares about right now.

1. **Chat/session history (PRIMARY SOURCE)** — Read the full conversation history carefully.
   What has the user been working on? What did they ask for that isn't done yet? What
   problems did they mention? What did they say they wanted to do next? Extract concrete
   unfinished tasks, stated intentions, and open threads. This is your #1 source for goals.
2. **Unfinished changes** — Git state: recent commits on `BRANCH`, uncommitted work,
   branch context. Cross-reference with session history to understand what's been
   completed vs. still pending.
3. **Unimplemented specs and designs** — Check `docs/` for specs or designs that have
   been discussed or referenced in the session but lack matching implementation.
4. **Failing tests** — Run the test suite to find what's broken
5. **Codebase TODOs** — `grep -r "TODO\|FIXME\|HACK\|XXX"` across the project
6. **CLAUDE.md / project docs** — Understand conventions and architecture

### Phase 2: Goal Proposal

The night shift should keep the agent productively busy for **at least 5 hours**.
Propose **3 goals by default**. If the estimated total effort for 3 goals is under
5 hours, propose additional goals (4, 5, or more) until the total reaches at least
5 hours. There is no upper limit on goal count — what matters is filling the time
with meaningful work.

Present goals to the user in this format:

```
## Night Shift Goals

Branch: `BRANCH` (starting from commit `BASE_COMMIT`)
Estimated total runtime: ~[N] hours

I've scanned the session and codebase. Here's what I propose to work on tonight:

### Goal 1: [Title]
- **What:** [1-2 sentence description]
- **Why:** [Why this matters / what it unblocks]
- **Estimated effort:** [time estimate, e.g. ~45min, ~2hrs]
- **Deliverables:** [Concrete outputs: files created, tests passing, etc.]

### Goal 2: [Title]
...

### Goal 3: [Title]
...

(additional goals if needed to fill 5+ hours)

Want me to proceed with these, or would you like to modify them?
```

**Effort estimation guidelines:**
- Factor in the full execution loop per goal: planning, adversarial review, implementation,
  code review rounds, validation, and commit overhead.
- A "Small" goal (simple bug fix, small feature) typically takes ~30-60 min through the
  full loop. A "Medium" goal (new module, significant refactor) takes ~1-2 hrs. A "Large"
  goal (major feature, cross-cutting change) takes ~2-4 hrs.
- When in doubt, estimate conservatively (higher) — it's better to finish early than to
  run out of goals mid-shift.

**Goal selection principles:**
- **Conversation history is king.** Prioritize unfinished work, stated next-steps, and
  open problems from the chat session. If the user said "I still need to do X" or
  "next I want to tackle Y", those are your top goals.
- Then unfinished changes visible in the git state
- Then unimplemented specs and designs
- Then failing tests, bugs, TODOs, and tech debt
- Each goal should be independently valuable (don't chain them)
- Order by priority — if time runs out, earlier goals matter more

### Phase 3: Confirmation Gate

This is the **last** point where you wait for user input. After this, you are fully autonomous.

Accept the user's response:
- "looks good" / "go" / "yes" → proceed with all goals as proposed
- Modifications → adjust goals accordingly
- Removal → drop goals, proceed with remainder

Once confirmed, tell the user:
```
Night shift started on branch `BRANCH` (from commit BASE_COMMIT).
I'll work through these goals and leave a handoff note at
docs/night-shift/YYYY-MM-DD-HHMM-handoff.md when done.

Say "end night shift" any time to stop early. Good night!
```

## Run State Persistence

The night shift may run for hours across many goals. To survive context pressure and
produce accurate handoffs, persist run state to `.night-shift/state.json`
after every phase transition. This file is the source of truth — not your memory.

The state file lives in `.night-shift/` which is gitignored, so it can never be accidentally committed.

```json
{
  "started_at": "2025-07-12T23:00:00Z",
  "base_branch": "feat/paper-trading-runner",
  "base_commit": "abc1234",
  "expected_head": "def5678",
  "branch": "feat/paper-trading-runner",
  "goals": [
    {
      "id": 1,
      "title": "Fix RSI threshold tests",
      "status": "completed",
      "commit": "def5678",
      "codex_review_rounds": 2,
      "codex_review_status": "clean",
      "decisions_made": ["Used 14-period RSI instead of 21 per convention"],
      "issues_noted": []
    },
    {
      "id": 2,
      "title": "Implement volume-weighted exit",
      "status": "in_progress",
      "current_phase": "code_review",
      "codex_review_rounds": 1,
      "decisions_made": [],
      "issues_noted": []
    },
    {
      "id": 3,
      "title": "Add missing backtest metrics",
      "status": "pending"
    }
  ],
  "test_results": { "passed": 23, "failed": 2, "total": 25 }
}
```

Update this file after:
- Goal status changes (pending → in_progress → completed/reverted/blocked)
- Each Codex review round completes
- Each test run completes
- Any decision is made without user input

When writing the handoff note, read this file as the source of truth rather than
relying on memory.

Before starting the next goal, re-read this file to confirm where you left off. This
protects against context loss during long runs.

## Drift Check

Since the night shift works directly on the user's feature branch, the branch or HEAD
could change underneath the run (user makes a commit, rebases, switches branches).

The state file tracks an `expected_head` field — the exact commit the agent expects
HEAD to be at. This is updated only when the agent itself makes a commit (Step 6).
Initially it equals `BASE_COMMIT`.

Before any step that writes, commits, or rolls back, run this check:

```bash
CURRENT_BRANCH=$(git branch --show-current)
CURRENT_HEAD=$(git rev-parse HEAD)
# Check for unexpected working-tree changes (outside goal's known files)
UNEXPECTED_CHANGES=$(git status --porcelain=v1 -z | tr '\0' '\n' | grep -v '.night-shift/')
```

Verify:
1. `CURRENT_BRANCH` still matches `BRANCH` from the state file
2. `CURRENT_HEAD` exactly equals `expected_head` from the state file
3. Any dirty files in `UNEXPECTED_CHANGES` must be files the current goal is
   actively working on (i.e., listed in `.night-shift/plan.md` or already known
   from this goal's execution). If there are staged/unstaged/untracked changes to
   files the agent did NOT touch, something external modified the working tree.

**If any check fails, the run is over.** This means something external changed the
branch or working tree. The agent must:
1. **Make NO repo writes.** Do not commit, do not write a handoff file, do not rollback.
2. **Print a terminal-only summary** with what was accomplished before drift was detected,
   which goals are complete, and which was in progress.
3. **Stop.** The human must investigate what changed and decide how to proceed.

The `expected_head` is advanced only in Step 6 after the agent's own successful commit.

Run this drift check before:
- Step 3 (Execute — before writing code)
- Step 4 (Code Review — before the first review round)
- Step 6 (Commit — before staging/committing)
- Any rollback operation
- **Writing the handoff note** (before both the file write and the commit)

## The Execution Loop

For each goal, run this cycle. The cycle is designed so that Claude Code and Codex
check each other's work — neither trusts its own output without the other validating it.

```
┌─────────────────────────────────────────────────────┐
│                   FOR EACH GOAL                      │
│                                                      │
│  1. Plan (write to file)                             │
│     └─→ Write plan to .night-shift/plan.md           │
│                                                      │
│  2. Adversarial Review (Codex)                       │
│     └─→ /codex:adversarial-review on plan            │
│     └─→ If issues found → revise plan → re-review    │
│                                                      │
│  3. Execute                                          │
│     └─→ Implement the plan                           │
│                                                      │
│  4. Code Review (Codex)                              │
│     └─→ /codex:review on working-tree changes        │
│     └─→ If issues found → fix → re-review            │
│     └─→ Repeat until Codex says clean                │
│                                                      │
│  5. Validate                                         │
│     └─→ Run tests, verify deliverables               │
│     └─→ If fails → revert goal, do NOT continue      │
│                                                      │
│  6. Commit                                           │
│     └─→ Commit to the current branch                  │
│                                                      │
│  7. Update state → Next goal                         │
└─────────────────────────────────────────────────────┘
```

### Step 1: Plan (persisted to file)

Write a concrete implementation plan to `.night-shift/plan.md` in the repo root.
This file is overwritten for each goal. Include:
- Goal title and description
- Files to create or modify (with specific paths)
- The approach and key design decisions
- Test strategy
- Risks or assumptions

The plan must be a file because Codex needs something concrete to review in Step 2.

### Step 2: Adversarial Review via Codex

Have Codex challenge your plan in adversarial mode:

```
/codex:adversarial-review --wait --scope working-tree
```

This runs Codex in adversarial mode, questioning the design choices, tradeoffs, and
assumptions in your plan. Use `--wait` so the night shift agent blocks until the
review completes (do not run in background during night shift).

Read the review carefully. If Codex raises valid concerns:
- Revise `.night-shift/plan.md` to address them
- Run adversarial review again on the revised plan
- Maximum 2 revision rounds — if still contested, note the disagreement in the
  state file and handoff, then proceed with your best judgment

If Codex approves or raises only minor points, proceed to execution.

### Step 3: Execute

**Run the drift check** (see "Drift Check" section above).

**Before writing any code**, record a full snapshot so you can do a scoped rollback if needed:
```bash
GOAL_START_COMMIT=$(git rev-parse HEAD)
# Record ALL files before this goal: tracked + untracked, NUL-delimited for whitespace safety
{ git ls-files -z; git ls-files -z --others --exclude-standard; } | sort -zu > .night-shift/pre-goal-files.txt
```
Save `start_commit` in the state file under the current goal.

Implement the plan. Follow the project's existing patterns and conventions (from CLAUDE.md
and codebase observation). Write tests alongside the implementation, not after.

Key rules during execution:
- **No questions to the user.** If something is ambiguous, make a reasonable decision and
  record it in the state file's `decisions_made` array.
- **Stay in scope.** Don't refactor unrelated code. Don't add features beyond the goal.
- **Write tests.** Every goal should include test coverage for the changes made.
- **Subagent delegation is allowed for Step 3 ONLY.** You may use a subagent to write
  code and tests for speed, but the subagent prompt MUST include:
  - "Do NOT run `git add` or `git commit`"
  - "Only create/modify files listed in the plan"
  - "Report all files changed when done"
  The night shift agent MUST then independently run Steps 4-7 itself — drift checks,
  Codex code review, validation (including browser verification for UI), and commit.
  A subagent's "done" report is NOT evidence of quality. The Codex review is.
  **Do NOT delegate Steps 1, 2, 4, 5, 6, or 7 to subagents.** These steps require
  the night shift agent to maintain the execution loop's integrity.

### Step 4: Code Review via Codex

**Run the drift check** before the first review round.

After implementation, have Codex review the actual code changes:

```
/codex:review --wait --scope working-tree
```

This is the quality gate. Read the review and act on it:

- **Issues found → fix them → re-run `/codex:review --wait --scope working-tree`**
- **Repeat until Codex reports no significant issues**
- Maximum 3 review cycles per goal.

The goal is: **Codex says the code is clean before you move to the next goal.**

If Codex still finds issues after 3 rounds, this is a **hard stop for this goal**.
Perform a **scoped rollback** (see below), mark the goal as `blocked` in the state
file with the outstanding issues, and move to the next goal.

### Step 5: Validate

Before marking the goal complete:
1. Run the full test suite — all tests must pass
2. Verify the deliverables listed in the goal are actually delivered
3. **For UI goals** (templates, dashboard pages, CSS, frontend components): verification
   MUST include opening the page in a browser (via playwright, browse tool, or preview)
   and taking a screenshot as evidence. "Tests pass" is NOT sufficient for UI work —
   you must visually confirm the page renders correctly.
4. Check that no unintended files were modified

**Validation failure is a hard stop.** If tests fail:
- Attempt to fix the failures (1 attempt)
- Re-run `/codex:review --wait --scope working-tree` on the fixes
- Re-run tests
- If tests still fail after 1 fix attempt, perform a **scoped rollback** (see below).
  - Mark the goal as `blocked` in the state file
  - Record the test failures in `issues_noted`
  - Move to the next goal

The principle: **never commit code that doesn't pass tests.** The human should wake up to
a branch where every commit is green, even if fewer goals were completed.

### Step 6: Commit

**Run the drift check** before committing.

**Pre-commit gate:** Before staging anything, read `.night-shift/state.json` and verify
the current goal has `codex_review_status: "clean"`. If the status is `"skipped"`,
`"unavailable"`, or anything other than `"clean"`, the goal CANNOT be committed — it
must either go through Codex review or be reverted. This is a hard gate.

**Stage only goal deliverables.** Use targeted `git add` with specific file paths:
```bash
git add src/strategy/momentum.py tests/test_momentum.py   # example: only changed files
# NEVER: git add . or git add -A   # .night-shift/ is gitignored but be explicit
```

Commit with a clear message:
```
feat: [goal title]

Night shift goal [N/total]: [description]
- [key change 1]
- [key change 2]
```

Update the state file:
- Set goal status to `completed`
- Record the commit hash
- **Update `expected_head`** to the new commit: `git rev-parse HEAD`
- Record Codex review rounds and status
- Update test results

### Step 7: Update State and Next Goal

Clean up the plan file: `rm .night-shift/plan.md`

Re-read `.night-shift/state.json` to confirm current position, then move to the
next goal and repeat the loop.

If all goals are complete, proceed to writing the handoff note.

## Scoped Rollback

When a goal fails review or validation, rollback only that goal's changes — not the
entire working tree. This protects any unrelated changes that might exist on the branch.

**Run the drift check first.** If it fails, stop the run entirely instead of rolling back.

Rollback procedure (all paths NUL-delimited for whitespace safety):
```bash
# Read the goal's start commit from state file
GOAL_START=$(python3 -c "
import json
state = json.load(open('.night-shift/state.json'))
goal = next(g for g in state['goals'] if g['status'] == 'in_progress')
print(goal['start_commit'])
")

# 1. Capture current full file inventory (tracked + untracked), NUL-delimited
{ git ls-files -z; git ls-files -z --others --exclude-standard; } | sort -zu > .night-shift/post-goal-files.txt

# 2. Identify NEW files created by this goal FIRST (before checkout)
#    (comm -z is not available on macOS, so use python3 for NUL-safe set diff)
python3 -c "
import sys
pre = set(open('.night-shift/pre-goal-files.txt','rb').read().split(b'\x00'))
post = set(open('.night-shift/post-goal-files.txt','rb').read().split(b'\x00'))
new_files = post - pre - {b''}
sys.stdout.buffer.write(b'\x00'.join(new_files))
" > .night-shift/goal-new-files.txt

# 3. Remove goal-created new files FIRST (before git checkout)
#    git checkout on a file that didn't exist at GOAL_START would error
xargs -0 rm -f < .night-shift/goal-new-files.txt
# Also unstage them if they were git-added
xargs -0 git rm --cached --ignore-unmatch -- < .night-shift/goal-new-files.txt 2>/dev/null

# 4. Identify files MODIFIED (not new) by this goal — these existed at GOAL_START
#    Only these are safe to git checkout
git diff -z --name-only "$GOAL_START" > .night-shift/goal-modified.txt
git diff -z --name-only >> .night-shift/goal-modified.txt
sort -zu -o .night-shift/goal-modified.txt .night-shift/goal-modified.txt

# 5. Restore only pre-existing modified files to their pre-goal state
xargs -0 git checkout "$GOAL_START" -- < .night-shift/goal-modified.txt 2>/dev/null

# 6. VERIFY rollback succeeded — this is a HARD GATE
git diff --quiet "$GOAL_START"
ROLLBACK_OK=$?
```

**Rollback verification is a hard stop.** If `ROLLBACK_OK` is non-zero (rollback
incomplete), the run is over:
1. Do NOT continue to the next goal
2. Run the `caffeinate-stop` pattern
3. Print a terminal-only summary explaining that rollback failed and the working
   tree may contain partial changes from the blocked goal
4. The human must investigate

**Safety check before rollback:** If `git status --porcelain=v1` shows modifications
to files NOT in `goal-modified.txt` or `goal-new-files.txt`, something unexpected
happened. **Stop the run** immediately (same hard-stop path as above).

## Codex Unavailability

**"Unavailable" means Codex returned a technical error** — command not found, timeout,
or runtime crash. It does NOT mean you decided to skip it. Choosing not to run Codex
is a protocol violation, not an unavailability event. If you skip Step 2 or Step 4
by choice, the goal MUST be reverted.

If Codex is genuinely unavailable (technical error):

- **For adversarial review (Step 2):** Proceed without it — the adversarial review is
  valuable but not the primary quality gate. Note in the state file that adversarial
  review was skipped and include the error message.
- **For code review (Step 4):** This is the primary quality gate. If Codex is unavailable:
  - Perform a self-review: re-read all changed files with fresh eyes
  - Run the test suite as the minimum quality bar
  - Note in the state file that Codex review was unavailable, with the error message
  - The handoff note must prominently flag that these changes were not Codex-reviewed

No other reason justifies skipping Codex review — not "straightforward changes", not
"UI-only work", not "tests pass". The review loop is the quality gate. Run it.

## Writing the Handoff Note

When all goals are done (or the user says "end night shift"):

1. **Run the drift check.** If it fails, emit terminal-only summary and stop.
2. Read `.night-shift/state.json` as the source of truth
3. **Stage the handoff content outside the repo first:** write to `.night-shift/handoff-draft.md`
   (this is gitignored, so it doesn't touch the repo)
4. **Run the drift check again.** If it fails, the handoff draft is in `.night-shift/` but
   the repo is untouched — emit terminal-only summary and stop.
5. Move the draft into the repo: `mkdir -p docs/night-shift && cp .night-shift/handoff-draft.md docs/night-shift/YYYY-MM-DD-HHMM-handoff.md`
6. Commit the handoff note (must be non-interactive — no editor):
   `git add docs/night-shift/YYYY-MM-DD-HHMM-handoff.md && git commit -m "docs: night shift handoff YYYY-MM-DD-HHMM"`
7. Update `expected_head` in state file after the handoff commit
7. Clean up:
   - Run the `caffeinate-stop` pattern (read PID from `.night-shift/caffeinate.pid`,
     validate, kill)
   - Delete the state directory: `rm -rf .night-shift/`

Use this structure:

```markdown
# Night Shift Handoff — YYYY-MM-DD

## Summary
[2-3 sentences: what was accomplished overall]

**Branch:** `BRANCH`
**Started from:** `BASE_COMMIT`
**Commits:** [N commits]

## Goals

### Goal 1: [Title] — ✅ Complete / ⚠️ Blocked / ❌ Reverted
**Commit:** [short hash + message]
**Changes:**
- [bullet list of what was done]

**Decisions made:**
- [any judgment calls you made without the user]

**Codex review status:** Clean after N rounds / Unavailable / Blocked: [issues]

### Goal 2: [Title] — ...
...

### Goal 3: [Title] — ...
...

## Test Results
- Tests passing: X/Y
- New tests added: N

## Items Needing Human Attention
- [Anything you were unsure about]
- [Decisions that should be validated]
- [Issues Codex flagged that you disagreed with]
- [Goals that were blocked/reverted and why]

## How to Review
\```bash
# See all night shift commits (from where it started)
git log BASE_COMMIT..HEAD --oneline

# See the full diff
git diff BASE_COMMIT

# Revert a specific goal's commit
git revert <commit-hash>

# Or undo all night shift work
git reset --hard BASE_COMMIT
\```

## Recommendations for Next Session
- [What to work on next]
- [Any tech debt introduced]
```

After writing the handoff, print a brief summary to the terminal:
```
Night shift complete! Handoff note: docs/night-shift/YYYY-MM-DD-HHMM-handoff.md

Branch: BRANCH ([N] new commits since BASE_COMMIT)
Run `git log BASE_COMMIT..HEAD --oneline` to review commits.

Completed: [N/total] goals
- ✅ Goal 1: [title]
- ✅ Goal 2: [title]
- ⚠️ Goal 3: [title] (blocked — see handoff for details)
```

## Ending the Night Shift Early

If the user says "end night shift" mid-execution:
1. Finish the current atomic operation (don't leave broken code)
2. For the current in-progress goal:
   - If it has **already passed both Step 4 (Codex review clean) and Step 5 (tests pass)**,
     commit it normally via Step 6.
   - Otherwise, **revert it** — do not commit code that hasn't cleared the full quality
     gate, even if tests pass. The Codex review requirement is not waived by early stop.
3. Write the handoff note with current progress, marking the in-progress goal as
   "interrupted — reverted" if it was rolled back
4. Summarize what was done and what remains

## Error Recovery

- **Build breaks:** Fix it before moving on. If you can't fix it in 2 attempts, revert
  all changes for that goal and mark it as blocked.
- **Merge conflicts:** Should not happen since no one else is committing to this branch.
  If they do, something is unexpected — stop the run and write the handoff.
- **Codex unavailable:** See "Codex Unavailability" section above.

## Principles

1. **Every commit must be green.** Never commit code that doesn't pass tests. Revert rather
   than leave broken code on the branch.
2. **Leave no broken windows.** Never leave the codebase in a worse state than you found it.
   Use scoped rollback to undo only the failed goal's changes.
3. **Document your judgment calls.** Every decision you make without the human goes in the
   state file and the handoff.
4. **Trust the review loop.** The Claude↔Codex review cycle is your quality gate. If the
   gate can't run, flag it loudly.
5. **Stay in your lane.** Only work on confirmed goals. Don't "helpfully" refactor other things.
6. **Never push.** Never push to remote. The human decides when to push.
7. **State file is truth.** Always update and re-read `.night-shift/state.json`. Don't
   trust your memory for multi-hour runs.
