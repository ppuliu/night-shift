---
name: night-shift
description: >
  Autonomous agent that runs while the human is away. Trigger with "/night-shift"
  or phrases like "start night shift", "going to sleep", "take over for the night",
  "keep working while I'm away". The skill routes based on context: if no shift
  is active, it starts one; if a shift is already active, it prompts Stop / Resume
  / Abandon. Stop phrases like "stop night shift" or "end night shift" run the
  end procedure directly. The user approves ONE thing — an objective. Key Results
  and their tasks are decided iteratively during execution, each gated by a
  Codex review. The shift ends either when Codex and the agent agree further
  work would be over-engineering, or at an 8-hour hard cap. Each shift is an
  independent run with its own folder under `.night-shift/runs/<RUN_ID>/`;
  previous runs are kept as history. Use this any time the user wants unattended
  autonomous work or overnight runs.
---
# Night Shift

You are entering autonomous mode. The human is stepping away — sleeping, taking a
break, or otherwise unavailable. Your job is to make meaningful progress on their
codebase without asking any questions after the initial goal confirmation.

This skill has a **single unified entrypoint**. On invocation, route based on
(a) whether an active night shift exists and (b) the user's phrasing.

**A note on OKR vocabulary.** This skill borrows three terms from the OKR
framework — **objective** (aspirational direction the user sets), **key
result** (a concrete shippable step that advances the objective, gated by
Codex), and **task** (an independently committable unit under a key result).
We borrow the vocabulary and hierarchy only; we do not adopt the rest of
corporate OKR ritual (scoring, quarterly cycles, stretch targets, etc.).
Key results here are typically deliverable-shaped ("migrate /users endpoints
to ApiError") rather than metric-shaped; Codex's gate is "does this advance
the objective?" not "is this measurable?".

## Routing

First, detect whether any active shift exists:

```bash
# Active = any .night-shift/runs/*/state.json with status == "running".
# No shell variables named `status` — zsh makes `$status` read-only.
for f in .night-shift/runs/*/state.json; do
  [ -e "$f" ] || continue
  [ "$(jq -r .status "$f" 2>/dev/null)" = "running" ] && echo "$f"
done
```

If there are multiple active runs (shouldn't happen normally), pick the most
recent by `started_at` and warn the user.

> **Shell note:** the `Bash` tool may run under zsh, where `$status` is a
> read-only built-in. Never use `status` as a shell variable name in any
> snippet — use `run_status`, `st`, or run the logic inside `sh -c '...'`
> (as above) so the assignment executes in sh, not zsh.

Then route:


| Phrasing                                                                      | Active shift? | Action                                                    |
| ----------------------------------------------------------------------------- | ------------- | --------------------------------------------------------- |
| start/go trigger (e.g. "start night shift", "/night-shift", "going to sleep") | No            | Begin fresh shift → §Pre-flight                           |
| start/go trigger                                                              | Yes           | Prompt **Stop / Resume / Abandon** → §Stop-Resume-Abandon |
| stop trigger (e.g. "stop night shift", "end night shift", "wrap up")          | Yes           | Run end procedure → §Ending a Shift                       |
| stop trigger                                                                  | No            | Reply: "No active night shift to stop." Stop.             |


## Pre-flight (new shift)

Run these checks in order. Failures stop the shift before any work begins.

**Ask pre-flight questions ONE AT A TIME.** Do not batch confirmations into a
single message. For each check that requires user input (bypass-permissions
confirmation, non-git mode confirmation, branch creation on main/master,
uncommitted changes, objective, goal approval), send a single question,
wait for the reply, then proceed to the next check. This keeps the pre-flight
conversational and avoids overwhelming the user with a wall of decisions.

### 1. Bypass-permissions confirmation

Night shift is fully autonomous — it runs many tool calls and edits without human
input. If Claude Code is NOT launched with `--dangerously-skip-permissions`, the
loop will stall on permission prompts while the user is away.

There is **no reliable way** for the agent to introspect the current permission
mode from inside a session. So ask the user to confirm once, as the first pre-flight
question:

```
Before I start: is this session running in bypass-permissions mode
(i.e., you launched Claude Code with `--dangerously-skip-permissions`)?

Without it, the autonomous loop will stall on permission prompts while you're away.

Reply "yes" to proceed, or "no" to abort — in which case, relaunch Claude Code
with the flag and trigger night shift again.
```

If the user says no → abort with the above instructions. If yes → continue.

### 2. `jq` dependency

Every subsequent step reads/writes `state.json` with `jq`. If it is not on
PATH, install it via the platform's package manager before proceeding:

```bash
if ! command -v jq >/dev/null 2>&1; then
  if command -v brew >/dev/null 2>&1; then
    brew install jq
  elif command -v apt-get >/dev/null 2>&1; then
    sudo apt-get update && sudo apt-get install -y jq
  elif command -v dnf >/dev/null 2>&1; then
    sudo dnf install -y jq
  elif command -v yum >/dev/null 2>&1; then
    sudo yum install -y jq
  elif command -v pacman >/dev/null 2>&1; then
    sudo pacman -S --noconfirm jq
  else
    echo "Could not auto-install jq. Please install it and retry." >&2
    exit 1
  fi
fi
```

On Linux this may prompt for a sudo password. Pre-flight is interactive (the
user is still present), so a password prompt is acceptable here — it would
not be acceptable once the autonomous loop starts. After installing, re-check
`command -v jq` and abort with a clear message if it still isn't available.

### 3. Git repo detection

Run `git rev-parse --is-inside-work-tree` to determine if the cwd is inside a
git repo.

- **Git repo → full mode.** All commit-per-goal, drift, rollback, handoff-commit
machinery runs.
- **No git repo → graceful-degrade mode.** See §Non-Git Degrade Mode for the
limitations. Warn the user once and ask for confirmation before proceeding:
  ```
  This directory is not a git repo. Night shift will run in degrade mode:
  no commits, no rollback, no drift protection. Failed goals may leave partial
  changes in your working tree. Proceed?
  ```
  If yes → continue in degrade mode. If no → abort.

### 4. Gitignore entry (git mode only)

Ensure `.night-shift/` is ignored. Read `.gitignore` (create if missing) and
append `.night-shift/` on its own line if not already present. This is a one-line
diff the user will see alongside their other changes — expected and explicit.

```bash
if ! grep -qxF '.night-shift/' .gitignore 2>/dev/null; then
  printf '\n.night-shift/\n' >> .gitignore
fi
```

Do NOT use `.git/info/exclude` — we prefer the tracked `.gitignore` for
collaborator consistency and clarity.

### 5. Branch check (git mode only)

Run `git branch --show-current`. Must not be `main` or `master`.

- If on `main` or `master`: **propose creating a new branch** rather than
waiting for the user. Suggest a descriptive name based on session context
(e.g., `night-shift/YYYY-MM-DD` or `feat/<topic>` inferred from the
conversation). Ask for confirmation or a different name. On confirmation,
run `git checkout -b <name>` and proceed.
- On any other branch: proceed on it.

Record the branch as `BRANCH` and the current commit as `BASE_COMMIT`.

### 6. Clean working tree (git mode only)

Run `git status --short`. If there are uncommitted changes (other than the
`.gitignore` line we just added), ask the user to commit or stash them. This is
the only other permitted question besides goal confirmation.

If the `.gitignore` change is the only dirty file, commit it automatically
with message `chore: ignore .night-shift/` before proceeding.

### 7. Active-shift guard

This should have been handled in §Routing, but guard against races: re-scan
`.night-shift/runs/*/state.json` and verify no `status: "running"` entry exists.
If one appeared between routing and here, loop back to §Stop-Resume-Abandon.

### 8. Initialize this run

**All timestamps in this skill are LOCAL time, not UTC.** Run folders and
handoffs are read by the human on their wall clock; UTC makes "when did this
run start" mental math harder. Use `date` (no `-u`) everywhere. For
machine-readable timestamps in state.json, include the local offset — never
append `Z`.

```bash
RUN_ID=$(date +%Y-%m-%d-%H%M)                              # local time
RUN_DIR=".night-shift/runs/${RUN_ID}"
mkdir -p "$RUN_DIR/key results"
```

Write initial state to `$RUN_DIR/state.json`. Note `started_at` is **null** at
this point — pre-flight (bypass-permissions Q, branch Q, objective Q, etc.)
is interactive, not autonomous work. The shift clock starts at the handoff
banner (see §Objective Confirmation), so elapsed-time and the 8h hard cap
measure only the autonomous portion.

```json
{
  "run_id": "2026-04-19-0847",
  "status": "running",
  "started_at": null,
  "mode": "git",
  "cwd": "/abs/path/to/project",
  "skill_dir": "/Users/.../.claude/skills/night-shift",
  "branch": "feat/xyz",
  "base_commit": "abc1234",
  "expected_head": "abc1234",
  "objective": null,
  "hard_cap_hours": 8,
  "average_task_duration_minutes": null,
  "end_consensus_file": null,
  "key_results": [],
  "test_results": null
}
```

(In degrade mode: `"mode": "no-git"`, omit `branch`, `base_commit`, `expected_head`.)

### 9. Locate INVARIANTS.md in the skill directory

The non-negotiable rules live in `INVARIANTS.md`, a sibling of this SKILL.md
in the skill's install directory. The agent does NOT copy it into the run
folder — it reads directly from the skill dir every time it needs a refresh.

Record the skill directory path in `state.json` as `skill_dir` so later
tasks (which run after context compaction) can find the invariants file
without re-deriving the path:

```json
{
  "skill_dir": "/Users/.../.claude/skills/night-shift",
  ...
}
```

The agent can locate the skill dir at pre-flight time because it was loaded
from there. If the path is ever unknown at runtime, fall back to
`$HOME/.claude/skills/night-shift/INVARIANTS.md` (the default user-level
install location).

## Objective Question

Before scanning anything, ask the user one question:

```
Do you have a high-level objective for tonight — something like
"improve the checkout flow" or "harden error handling in the API layer" —
or should I propose goals based on session context?

Reply with a goal, or "propose" to let me choose.
```

- **If the user gives an objective:** record it in `state.json.objective`.
Reconnaissance is scoped toward that goal — session history, specs, and
code related to the objective area are the priority.
- **If "propose":** leave `objective: null` and run full reconnaissance
across all sources.

This question is cheap and dramatically improves goal quality — it prevents
the skill from proposing low-value micro-goals when the user has a clear
larger direction in mind.

## Reconnaissance (2–3 minutes)

Scan these sources in priority order. Conversation history is the most
important signal — it tells you what the user actually cares about right now.

1. **Chat/session history (PRIMARY)** — Read the full conversation carefully.

  What has the user been working on? What did they ask for that isn't done
   yet? What problems did they mention? Extract concrete unfinished tasks,
   stated intentions, and open threads. If an objective was given, focus on
   conversation threads related to it.
2. **Unfinished changes** — Git state: recent commits, uncommitted work.

  Cross-reference with session history. (Skip in degrade mode.)
3. **Unimplemented specs and designs** — Check `docs/` for specs discussed

  in the session but lacking implementation.
4. **Codebase TODOs** — `grep -r "TODO\|FIXME\|HACK\|XXX"` across the project.
5. **CLAUDE.md / project docs** — Conventions and architecture.

## Objective Confirmation (the only approval gate)

The user approves only ONE thing: the objective. Key Results and tasks are
decided iteratively during execution, each with its own Codex review.
Reviewing a long goal list at the start is a tax the user should not have to
pay.

**If the user gave an objective in the objective question:** echo it back
verbatim and ask for confirmation (single yes/no). Example:

```
Objective for tonight: "Harden error handling in the API layer"

I'll decide key results iteratively as I go, with Codex reviewing each one.
The shift ends when Codex and I agree further work would be over-engineering,
or after 8 hours — whichever comes first.

Confirm? (yes / edit)
```

**If the user asked the skill to "propose":** use reconnaissance to propose a
single objective and ask for confirmation. Example:

```
Based on session context, I propose this objective:

  "Finish the paper-trading dashboard and make it user-friendly."

I'll decide key results iteratively as I go. Shift ends on Codex+me consensus
or 8 hours, whichever comes first.

Confirm, or give me a different objective?
```

Once confirmed, the objective is locked for the run. Record it in
`state.objective`. No further user approval is needed until the handoff.

Once confirmed, emit a **visible handover banner** as its own message. This is
the marker between "interactive setup" and "autonomous execution". The user
should see it, understand they can walk away, and close the laptop if they
want. Use emojis and dividers so it stands out from ordinary output:

```
═══════════════════════════════════════════════════════════════
  🌙  NIGHT SHIFT ENGAGED  🌙
═══════════════════════════════════════════════════════════════

  You can step away now. I'll take it from here.

  Run ID:      RUN_ID
  Branch:      BRANCH (from BASE_COMMIT)          [git mode only]
  Objective:  <one-line summary>
  Max runtime: 8 hours (or until Codex and I agree we're done)
  Handoff:     .night-shift/runs/RUN_ID/handoff.md

  Say "end night shift" any time to stop early.
  Type /remote-control to monitor from your phone or browser.

  Sleep well. 🌙
═══════════════════════════════════════════════════════════════
```

This banner is the ONLY place where emojis / ASCII decoration are expected —
it's intentional, to give the user a clear visual handoff. After this
message, the agent enters autonomous mode. No more questions until handoff.

**Immediately after emitting the banner, set `started_at` in state.json.**
This is the shift's true start — all elapsed-time computations and the 8-hour
hard cap measure from here, not from pre-flight:

```bash
STARTED_AT=$(date +%Y-%m-%dT%H:%M:%S%z)
jq --arg t "$STARTED_AT" '.started_at = $t' "$RUN_DIR/state.json" \
  > "$RUN_DIR/state.json.tmp" && mv "$RUN_DIR/state.json.tmp" "$RUN_DIR/state.json"
```

## Run State Persistence

The shift may run for hours across many goals. To survive context pressure and
produce accurate handoffs, persist run state to `$RUN_DIR/state.json` after
every phase transition. **This file is the source of truth — not your memory.**

```json
{
  "run_id": "2026-04-19-0847",
  "status": "running",
  "mode": "git",
  "started_at": "2026-04-19T08:47:00-0700",
  "cwd": "/abs/path",
  "branch": "feat/xyz",
  "base_commit": "abc1234",
  "expected_head": "def5678",
  "objective": "Harden error handling in the API layer",
  "hard_cap_hours": 8,
  "average_task_duration_minutes": 10,
  "end_consensus_file": null,
  "key_results": [
    {
      "id": 1,
      "title": "Standardize error responses across API endpoints",
      "why": "Objective needs every endpoint to return consistent error shape",
      "status": "completed",
      "codex_approval_file": "key-results/1/codex-approval.txt",
      "decomp_adversarial_file": "key-results/1/decomp-adversarial.txt",
      "started_at": "2026-04-19T08:50:00-0700",
      "ended_at": "2026-04-19T09:20:00-0700",
      "duration_minutes": 30,
      "tasks": [
        {
          "id": 1,
          "title": "Introduce ApiError type with code/message/details",
          "status": "completed",
          "commit": "def5678",
          "start_commit": "abc1234",
          "plan_adversarial_file": "key-results/1/tasks/1/plan-adversarial.txt",
          "code_review_file": "key-results/1/tasks/1/code-review.txt",
          "code_review_rounds": 1,
          "code_review_status": "clean",
          "code_review_evidence": "No P1 or P2 findings.",
          "started_at": "2026-04-19T08:50:00-0700",
          "ended_at": "2026-04-19T09:02:00-0700",
          "duration_minutes": 12,
          "decisions_made": [],
          "issues_noted": []
        }
      ]
    },
    {
      "id": 2,
      "title": "Add retry + circuit-breaker to outbound HTTP clients",
      "status": "in_progress",
      "codex_approval_file": "key-results/2/codex-approval.txt",
      "started_at": "2026-04-19T09:20:00-0700",
      "tasks": []
    }
  ],
  "test_results": { "passed": 23, "failed": 2, "total": 25 }
}
```

Valid run `status` values: `running`, `completed`, `interrupted`,
`abandoned`, `drift-stopped`, `hard-capped`.

Valid key result `status` values: `proposed`, `codex-rejected`, `in_progress`,
`completed`, `blocked`, `reverted`, `reverted-on-resume`.

Valid task `status` values: `pending`, `in_progress`, `completed`,
`blocked`, `reverted`, `reverted-on-resume`.

Update state.json after:

- Any status change (key result or task)
- Each Codex review round (approval, decomp adversarial, plan adversarial, code review)
- Start / end of each task and key result (with timestamps — used for
dynamic duration estimation)
- Each test run
- Any decision made without user input
- When the end-of-shift consensus file is written

Before starting the next task, **re-read state.json AND
`$SKILL_DIR/INVARIANTS.md`** (path stored as `skill_dir` in state.json) to
refresh context after any compaction. See §Per-Task Context Refresh.

## Drift Check (git mode only)

Since night shift works directly on the user's feature branch, the branch or
HEAD could change underneath the run (user makes a commit, rebases, switches
branches).

The state file tracks `expected_head` — the exact commit the agent expects HEAD
to be at. It is updated only when the agent itself makes a commit (Step 6).
Initially equals `BASE_COMMIT`.

Before any step that writes, commits, or rolls back:

```bash
CURRENT_BRANCH=$(git branch --show-current)
CURRENT_HEAD=$(git rev-parse HEAD)
UNEXPECTED_CHANGES=$(git status --porcelain=v1 -z | tr '\0' '\n' | grep -v '.night-shift/')
```

Verify:

1. `CURRENT_BRANCH` still matches `BRANCH` from the state file
2. `CURRENT_HEAD` equals `expected_head`
3. Any dirty files in `UNEXPECTED_CHANGES` must be files the current task

  is actively working on (listed in `current-plan.md` or already known from
   this task's execution). Untouched dirty files = external modification.

**If any check fails, the run is over:**

1. Set `status: "drift-stopped"` in state.json.
2. Make NO other repo writes. Do not commit, do not write a handoff file, do

  not rollback.
3. Print a terminal-only summary of what was accomplished before drift.
4. Stop. The human must investigate.

Run the drift check before:

- Inner 3 (Execute — before writing code)
- Inner 4 (Code Review — before first review round)
- Inner 6 (Commit — before staging)
- Any rollback operation
- Writing the handoff note (before both file write and commit)

## The Execution Loop

The loop has two levels. The **outer loop** iterates once per key result —
proposing, getting Codex approval, decomposing, and executing. The **inner
loop** runs once per task and is where the actual Claude↔Codex safety
dance happens.

```
┌──────────────────────────────────────────────────────────────────┐
│                   OUTER LOOP — per KEY RESULT                       │
│                                                                   │
│  0. Check end conditions (§End Conditions). If met → Handoff.     │
│  A. Propose next key result → write $RUN_DIR/proposed-key result.md   │
│  B. Codex key result approval review (gate)                         │
│        Approved → continue. Rejected → record, go back to A.      │
│  C. Write decomposition plan → $RUN_DIR/current-decomp.md         │
│  D. Codex adversarial-review the decomposition                    │
│  E. For each task in order → run INNER LOOP                   │
│  F. Record key result duration, update running avg, go to Outer 0.  │
│                                                                   │
│    ┌──────────────────────────────────────────────────┐           │
│    │          INNER LOOP — per TASK               │           │
│    │                                                   │           │
│    │  0. Context refresh (read INVARIANTS.md + state)  │           │
│    │  1. Write task plan to current-plan.md        │           │
│    │  2. Codex adversarial-review the plan             │           │
│    │  3. Execute                                       │           │
│    │  4. Codex code review (loop until clean, file gate)│          │
│    │  5. Validate (tests, deliverables, UI screenshot) │           │
│    │  6. Commit (git mode) / Record (degrade mode)     │           │
│    │  7. Update state → next task                  │           │
│    └──────────────────────────────────────────────────┘           │
└──────────────────────────────────────────────────────────────────┘
```

**Key Results are NOT pre-planned.** The agent decides each one iteratively,
using what's already been shipped and what the objective still needs. Codex
gates each key result (Outer B) and the shift ends either on dual consensus or
at the 8-hour hard cap (§End Conditions).

### Outer 0: End Condition Check

Before proposing a new key result, evaluate the end conditions (§End
Conditions). If either fires, skip to Handoff. Otherwise continue to A.

### Outer A: Propose the Next Key Result

Write a proposal to `$RUN_DIR/proposed-key result.md`. Overwritten per
iteration. Include:

- Proposed key result title
- **Why:** how it serves the objective, citing completed key results if
relevant (what's been done vs what remains)
- Rough scope (a few bullets) — NOT a full decomposition yet
- Alternatives you considered, and why this one first

Base the proposal on: the objective, `state.key_results[]` (what's shipped,
what's blocked), session history, git state, and specs/docs. Every proposal
must make progress toward the objective; if nothing remains that would, go
to §End Conditions (consensus path) instead.

### Outer B: Codex Key Result Approval

Codex gates each key result. Run an adversarial review on the proposal.

**Pass the full `state.json` to Codex verbatim.** Do NOT paraphrase the
objective or summarize completed key results — paraphrasing loses the exact
wording of the user-approved objective and the shape of what's already shipped,
which is exactly the information Codex needs to judge whether the proposal
is worth doing or would over-engineer. Include both `state.json` and
`proposed-key-result.md` in the review context (e.g. quote their contents
inline in the prompt, or reference both file paths explicitly).

```bash
# $PROMPT = state.json contents (verbatim) + proposed-key-result.md, bundled
#           as a single adversarial-review prompt describing what to evaluate.
# $OUT    = $RUN_DIR/key-results/<G>/codex-approval.txt
echo "$PROMPT" | codex exec - -s read-only -c 'model_reasoning_effort="high"' 2>&1 \
  | tee "$OUT" >/dev/null
```

Invoke via the Bash tool with `timeout: 600000` (10 min). `codex exec` has a
5-minute internal default and high-effort reviews occasionally need longer.

Save the full output to `$RUN_DIR/key-results/<G>/codex-approval.txt` and
record the path in `state.key_results[G].codex_approval_file`. The review must
answer, at minimum: **is this key result worth implementing toward the
objective, or would it over-engineer / over-optimize?**

- **Approved** (no P1/P2 findings contesting the key result's value): mark
`status: "in_progress"`, record `started_at`, proceed to C.
- **Rejected** (Codex contests the key result): mark `status: "codex-rejected"`, record Codex's reasoning in state. Loop back to A with
a different proposal. Rejections never auto-terminate the shift — if you find
yourself unable to propose a key result Codex will accept, that's a signal
to invoke §End Conditions (Condition 2) yourself: write `end-consensus-draft.md`,
include the rejected proposals and Codex's reasoning as evidence, and let
Codex's re-review on the full draft decide whether the shift ends or you
adopt Codex's counter-proposal. Until then, keep iterating or wait for
the 8h cap.

If Codex is unavailable, the agent MUST still write a rigorous
self-adversarial review to the same file, clearly marked
`CODEX UNAVAILABLE — SELF-ADVERSARIAL REVIEW`, challenging the key result's
value before approving. Agent self-approval without genuine adversarial
reasoning is a protocol violation.

### Outer C: Decomposition Plan

With the key result approved, write a concrete decomposition to
`$RUN_DIR/current-decomp.md`. Overwritten per key result. Include:

- Key Result title and the objective it serves
- Numbered list of **tasks**. Each task must be:
  - Independently meaningful (would make sense as its own commit)
  - Independently testable (has clear acceptance criteria)
- Dependencies between tasks if any
- Risks and open decisions

Use measured durations (`state.average_task_duration_minutes`) to size
the split — don't guess. Keep tasks in the same ballpark as the running
average so estimates remain useful.

### Outer D: Decomposition Adversarial Review

```bash
# $PROMPT = state.json (verbatim) + current-decomp.md.
# $OUT    = $RUN_DIR/key-results/<G>/decomp-adversarial.txt
echo "$PROMPT" | codex exec - -s read-only -c 'model_reasoning_effort="high"' 2>&1 \
  | tee "$OUT" >/dev/null
```

Save the full output to `$RUN_DIR/key-results/<G>/decomp-adversarial.txt` and
record the path in `state.key_results[G].decomp_adversarial_file`.

If Codex flags real issues (missing task, wrong order, task too
large, dependency cycle), revise `current-decomp.md` and re-run. Max 2
revision rounds; if still contested, record disagreement in state and
proceed.

Then copy the confirmed task list into `state.key_results[G].tasks[]`.

If Codex is unavailable, do a rigorous self-adversarial review and write
`CODEX UNAVAILABLE — SELF-ADVERSARIAL REVIEW` to the file.

### Outer E: Iterate Tasks

For each task in order, run the **Inner Loop** (below). Before each
task starts: record `started_at`. When it ends: record `ended_at` and
compute `duration_seconds`.

### Outer F: Key Result Complete

When all tasks have a terminal status (`completed`, `blocked`, or
`reverted`):

- Mark `state.key_results[G].status = "completed"` (even if some tasks
were blocked — the key result is "as done as it will get").
- Record `ended_at` and compute key result `duration_seconds`.
- Update `state.average_task_duration_minutes` = running mean over all
completed tasks across all key results. This informs future
decomposition and end-condition estimates.
- `rm "$RUN_DIR/current-decomp.md"` and `rm "$RUN_DIR/proposed-key result.md"`.
- Return to **Outer 0**.

---

### Inner 0: Per-Task Context Refresh

**Before planning any task, always do these three reads:**

```bash
SKILL_DIR=$(jq -r .skill_dir "$RUN_DIR/state.json")
cat "$SKILL_DIR/INVARIANTS.md"     # Refresh non-negotiable rules
cat "$RUN_DIR/state.json"          # Where are we exactly?
cat "$RUN_DIR/current-decomp.md"   # What's the parent goal's plan?
```

This is the single most important defense against context compaction. After
hours of work and multiple compactions, your memory of the skill rules can be
wrong. The files on disk are the source of truth. Always re-read them.

### Inner 1: Task Plan

Write a detailed implementation plan for THIS task to
`$RUN_DIR/current-plan.md`. Overwritten per task. Include:

- Task title and reference to parent goal
- Files to create or modify (specific paths)
- Approach and key design decisions
- Test strategy
- Risks or assumptions

### Inner 2: Task Plan Adversarial Review

```bash
# $PROMPT = state.json (verbatim) + current-decomp.md + current-plan.md.
# $OUT    = $RUN_DIR/key-results/<G>/tasks/<S>/plan-adversarial.txt
echo "$PROMPT" | codex exec - -s read-only -c 'model_reasoning_effort="high"' 2>&1 \
  | tee "$OUT" >/dev/null
```

Save the full output to `$RUN_DIR/key-results/<G>/tasks/<S>/plan-adversarial.txt`
and record the path in the task's `plan_adversarial_file`.

If Codex raises valid concerns, revise `current-plan.md` and re-run. Max 2
revision rounds; if still contested, note disagreement in state + handoff and
proceed.

**Inner 2 and Inner 4 are NOT interchangeable.** Inner 2 reviews the *plan*.
Inner 4 reviews the *actual code*. Running Inner 2 does NOT satisfy Inner 4.

### Inner 3: Execute

**Run the drift check** (git mode).

**Git mode:** record a snapshot for scoped rollback:

```bash
SUBTASK_START_COMMIT=$(git rev-parse HEAD)
{ git ls-files -z; git ls-files -z --others --exclude-standard; } \
  | sort -zu > "$RUN_DIR/key-results/<G>/tasks/<S>/pre-files.txt"
```

Save `start_commit` in the task's state entry.

**Degrade mode:** no rollback available. A failed task will leave partial
changes in the working tree.

Implement the plan. Follow existing project patterns. Write tests alongside
the implementation, not after.

Rules during execution:

- **No questions to the user.** Make a reasonable decision and record it in
the task's `decisions_made`.
- **Stay in scope of the task.** Don't silently expand into the next
task's work.
- **Write tests.**
- **Subagent delegation is allowed for Inner 3 ONLY.** Prompt MUST include:
  - "Do NOT run `git add` or `git commit`"
  - "Only create/modify files listed in the plan"
  - "Report all files changed when done"

  The night shift agent MUST then independently run Inner 4–7 itself. A
  subagent's "done" is NOT evidence of quality. The Codex review is.

  **Do NOT delegate Inner 0, 1, 2, 4, 5, 6, or 7.**

### Inner 4: Code Review Loop (Codex) — with file gate

**Run the drift check** before the first review round.

You MUST run the actual `codex review` CLI. Checking output yourself and
deciding "it's fine" is NOT the same as Codex saying it's clean.

```
REVIEW_ROUND      = 0
CODEX_OUTPUT_FILE = "$RUN_DIR/key-results/<G>/tasks/<S>/code-review.txt"
BASE_SHA          = <the task's start_commit from state.json>

while REVIEW_ROUND < 3:
    REVIEW_ROUND += 1
    # Invoke via the Bash tool with timeout: 600000 (10 min):
    codex review --base $BASE_SHA -c 'model_reasoning_effort="high"' 2>&1 \
      | tee $CODEX_OUTPUT_FILE
    # The pre-commit gate will verify CODEX_OUTPUT_FILE exists and contains a verdict.

    Parse the output:
    - NO P1 or P2 findings:
        → code_review_status = "clean"
        → code_review_rounds = REVIEW_ROUND
        → code_review_file = CODEX_OUTPUT_FILE
        → code_review_evidence = <the verdict line, e.g. "No P1 or P2 findings">
        → BREAK
    - P1 or P2 findings:
        → Fix each finding
        → Update state with round + findings
        → CONTINUE (re-run review on the fixes, overwriting CODEX_OUTPUT_FILE)

If REVIEW_ROUND == 3 and still P1/P2:
    → HARD STOP: revert task via scoped rollback, mark blocked
```

**Every fix MUST be re-reviewed.** The loop only exits when Codex reports no
significant issues, or after 3 failed rounds (triggers revert). No "findings
don't apply" escape — fix them or revert.

**"Clean"** = no `[P1]` or `[P2]` markers in the Codex output.

**Does NOT count as running Inner 4:**

- Citing a prior Inner 2 adversarial review
- "Too simple to need review"
- "Tests pass so it's fine"
- Running the review on a different task
- Writing a plausible-looking review to the file without actually running Codex

### Inner 5: Validate

Before marking the task complete:

1. Run the full test suite — all tests must pass.
2. Verify the task's deliverables are actually delivered.
3. **For UI tasks** (templates, pages, CSS, frontend components):

  verification MUST include opening the page in a browser (playwright,
   browse, or preview) and screenshotting as evidence. "Tests pass" is NOT
   sufficient for UI work.
4. Check no unintended files were modified.

**Validation failure is a hard stop.** If tests fail:

- Attempt to fix (1 attempt)
- Re-run `codex review --base $BASE_SHA -c 'model_reasoning_effort="high"' 2>&1 | tee $CODEX_OUTPUT_FILE`
on the fixes
- Re-run tests
- If tests still fail:
  - **Git mode:** scoped rollback of this task, mark `blocked`, record
  failures in `issues_noted`, move to next task.
  - **Degrade mode:** mark task `blocked`, record failures. Partial
  changes remain in the working tree — flag in handoff.

**Principle: never commit code that doesn't pass tests.** The human wakes up
to a branch where every commit is green, even if fewer tasks completed.

### Inner 6: Commit (git mode) / Record (degrade mode)

**Git mode — Run the drift check first.**

**Pre-commit structural gate.** Before staging anything, verify ALL of:

1. `state.key_results[G].tasks[S].code_review_status == "clean"`
2. `state.key_results[G].tasks[S].code_review_file` is set AND the file

  exists AND is non-empty
3. The file contains a Codex verdict line (the `code_review_evidence` stored

  in state matches a line in the file)

```bash
REVIEW_FILE="$RUN_DIR/key-results/<G>/tasks/<S>/code-review.txt"
[ -s "$REVIEW_FILE" ] || { echo "FATAL: code-review.txt missing or empty"; exit 1; }
grep -qF "$CODE_REVIEW_EVIDENCE" "$REVIEW_FILE" || { echo "FATAL: verdict line not in file"; exit 1; }
```

If any of this fails, the task CANNOT be committed — revert it. Hard
gate. No exceptions. This exists specifically to prevent the agent from
rationalizing "I'll skip Codex for efficiency" after a context compaction:
the file either exists with real Codex output, or the task dies here.

Stage only this task's deliverables with targeted `git add`:

```bash
git add src/errors/ApiError.ts tests/errors/ApiError.test.ts
# NEVER: git add . or git add -A
```

Commit:

```
[[ORCA_RAW_HTML_INLINE:%3Ctype%3E]]: [task title]

Night shift key result [[ORCA_RAW_HTML_INLINE:%3CG%3E]] ("[[ORCA_RAW_HTML_INLINE:%3Ckey%20result%20title%3E]]"), task [[ORCA_RAW_HTML_INLINE:%3CS%3E]]/[[ORCA_RAW_HTML_INLINE:%3Ctotal%3E]]:
- [key change 1]
- [key change 2]
```

Update state.json:

- Task `status` → `completed`
- Record commit hash
- **Update `expected_head`** to new HEAD: `git rev-parse HEAD`
- Confirm `code_review_rounds`, `code_review_status`, `code_review_evidence`,
`code_review_file` are recorded
- Update test results

**Degrade mode:** no commit. Update state.json same way but with
`commit: null`. Changes remain in the working tree.

### Inner 7: Update State and Next Task

Delete `$RUN_DIR/current-plan.md`: `rm "$RUN_DIR/current-plan.md"`.

Re-read state.json, move to the next pending task in the current
key result.

If all tasks in the current key result are done → return to **Outer F**.

## Scoped Rollback (git mode only)

Rollback operates at the **task** level. When a task fails review or
validation, rollback only that task's changes — not the goal, not the
working tree. Previously-completed tasks in the same goal stay committed.

**Run the drift check first.** If it fails, stop the run entirely instead of
rolling back.

```bash
# Locate the in-progress task's start_commit and pre-files.txt
SUBTASK_START=$(python3 -c "
import json
state = json.load(open('$RUN_DIR/state.json'))
for g in state['key_results']:
    for s in g.get('tasks', []):
        if s['status'] == 'in_progress':
            print(s['start_commit'])
            break
")
SUB_DIR="$RUN_DIR/key-results/<G>/tasks/<S>"   # substitute actual IDs

# Capture post-subtask file inventory
{ git ls-files -z; git ls-files -z --others --exclude-standard; } \
  | sort -zu > "$SUB_DIR/post-files.txt"

# Identify NEW files (created by this task)
python3 -c "
import sys
pre = set(open('$SUB_DIR/pre-files.txt','rb').read().split(b'\x00'))
post = set(open('$SUB_DIR/post-files.txt','rb').read().split(b'\x00'))
new_files = post - pre - {b''}
sys.stdout.buffer.write(b'\x00'.join(new_files))
" > "$SUB_DIR/new-files.txt"

# Remove new files FIRST (git checkout errors on files that didn't exist at SUBTASK_START)
xargs -0 rm -f < "$SUB_DIR/new-files.txt"
xargs -0 git rm --cached --ignore-unmatch -- < "$SUB_DIR/new-files.txt" 2>/dev/null

# Identify MODIFIED files (pre-existing)
git diff -z --name-only "$SUBTASK_START" > "$SUB_DIR/modified.txt"
git diff -z --name-only >> "$SUB_DIR/modified.txt"
sort -zu -o "$SUB_DIR/modified.txt" "$SUB_DIR/modified.txt"

# Restore only pre-existing modified files to pre-task state
xargs -0 git checkout "$SUBTASK_START" -- < "$SUB_DIR/modified.txt" 2>/dev/null

# VERIFY — hard gate
git diff --quiet "$SUBTASK_START"
ROLLBACK_OK=$?
```

**Rollback verification is a hard stop.** If `ROLLBACK_OK` != 0:

1. Do NOT continue to the next task.
2. Set run `status: "drift-stopped"` in state.json.
3. Print terminal-only summary — working tree may contain partial changes.
4. The human must investigate.

**Safety check before rollback:** If `git status --porcelain=v1` shows
modifications to files NOT in `modified.txt` or `new-files.txt` for this
task, something unexpected happened. Stop immediately (same hard-stop
path).

## End Conditions

The shift ends when **one** of these two conditions fires (checked at Outer 0
before proposing the next key result):

### Condition 1: Hard Cap — 8 hours elapsed

Compute elapsed time: `now - state.started_at`. If elapsed ≥
`state.hard_cap_hours` (default 8):

- Finish the CURRENT task if one is in progress (run it through the
inner loop including commit). Do NOT start a new task.
- Set run `status: "hard-capped"`.
- Proceed to Handoff.

This is a wall-clock cap, not a "target" — 8h is the ceiling, not a goal.

### Condition 2: Dual Consensus — the objective is done

The agent proposes "we're done" → Codex must agree before handoff fires.
This prevents the agent from ending early on its own.

**Agent-initiated path:** At Outer 0, if the agent judges that any further
key result would over-engineer or over-optimize the objective:

1. Write a "done reasoning" file to `$RUN_DIR/end-consensus-draft.md`:
  - The verbatim objective from `state.objective` (do NOT paraphrase)
  - Completed key results as bullet points copied from `state.key_results[]`
  (titles and outcomes as recorded, not summarized in new words)
  - Why any plausible next key result would be over-engineering or
  over-optimizing (be specific — enumerate the ideas you considered)
  - Recommended remaining work, if any, for a future shift
2. Run Codex adversarial review on the draft. **Pass the full `state.json`
   to Codex verbatim** alongside the draft — the "done vs keep going"
   decision is only valid if Codex sees the literal objective and the complete
   key-result history. Paraphrasing is forbidden here; it was the specific
   failure mode that caused Codex to approve a premature end in an earlier run.
  ```bash
  # $PROMPT = state.json (verbatim) + end-consensus-draft.md.
  # $OUT    = $RUN_DIR/end-consensus.txt
  echo "$PROMPT" | codex exec - -s read-only -c 'model_reasoning_effort="high"' 2>&1 \
    | tee "$OUT" >/dev/null
  ```
3. Save Codex's response to `$RUN_DIR/end-consensus.txt`. Update

  `state.end_consensus_file`.
4. **Parse Codex's verdict:**
  - **Codex agrees** (no P1/P2 findings contesting "done"): the run
   terminates. Proceed to Handoff.
  - **Codex disagrees** (Codex proposes additional valuable work):
  extract Codex's suggested key result. The agent MUST start a new
  iteration at Outer A using that key result as the starting proposal
  (the agent may refine it, but cannot discard it without a
  counter-argument Codex re-reviews). This is how Codex blocks early
  exit.

There is no Codex-initiated end path. Repeated Outer B rejections are a
signal *to the agent* that it should write `end-consensus-draft.md` and run
the agent-initiated path above (citing the rejections as evidence) — but
they never replace the re-review. Codex's "done" verdict only counts when
issued against a full draft that sees the verbatim objective and complete
key-result history.

**If Codex is unavailable** during Condition 2: the agent CANNOT end on
consensus alone. It must either (a) keep proposing key results and running
them through the self-adversarial loop, or (b) wait for the 8h cap.
Self-consensus is not valid consensus.

### Neither condition = keep going

If neither fires, return to Outer A and propose the next key result.

## Time Tracking

Durations drive realistic estimates for task sizing and end-condition
proximity. Track them on every task and key result:

- `started_at` — recorded when the task / key result moves to `in_progress`
- `ended_at` — recorded when it reaches a terminal status
- `duration_minutes` = `(ended_at - started_at)` rounded to minutes

At Outer F (key result completion), update the running mean:

```
state.average_task_duration_minutes =
  mean(duration_minutes for all tasks with status in
       {completed, blocked, reverted} across all key results)
```

Use the running mean when sizing new tasks in Outer C. If a proposed
task looks 3× the running mean, split it. If it looks 3× smaller, merge
with an adjacent one. This keeps durations honest — no static "small/medium/
large" heuristic table.

For end-condition proximity, the agent can estimate whether another key result
will fit in the remaining wall-clock before the 8h cap:

```
remaining_minutes = hard_cap_hours * 60 - minutes_since(started_at)
expected_key_result_minutes = average_key_result_duration_minutes (mean of
                            duration_minutes across completed key results)
```

If `remaining_minutes < expected_key_result_minutes`, the agent may choose to
go to Condition 2 early (propose "done"), but this is a suggestion, not a
rule — Codex still has to agree.

## Codex Unavailability

**"Unavailable" = Codex returned a technical error** (command not found,
timeout, runtime crash). It does NOT mean you decided to skip it. Choosing to
skip Codex is a protocol violation, not an unavailability event. Skipping
Outer B, Inner 2, or Inner 4 by choice means the task MUST be reverted.

If Codex is genuinely unavailable:

- **Adversarial review (Outer B / Inner 2):** Proceed without it — adversarial
review is valuable but not the primary quality gate. Write a short note to
the corresponding adversarial file (e.g. `plan-adversarial.txt`) containing
`"UNAVAILABLE: <error message>"` so the artifact still exists. Record the
skip in state.json.
- **Code review (Inner 4):** Primary quality gate. If unavailable:
  - Perform a self-review: re-read all changed files with fresh eyes.
  - Run the test suite as the minimum quality bar.
  - Write the self-review output to the task's `code-review.txt` with
  a clearly-marked header: `"CODEX UNAVAILABLE — SELF-REVIEW: <error>"`
  followed by your notes.
  - Record `code_review_status: "self-reviewed-unavailable"` in state.json
  (a distinct value from `"clean"`).
  - The pre-commit structural gate still requires the file to exist and
  be non-empty — the self-review artifact satisfies that.
  - The handoff MUST prominently flag that these tasks were not Codex-reviewed.

No other reason justifies skipping Codex — not "straightforward changes", not
"UI-only work", not "tests pass". Run it.

## Handoff (End of Successful Shift)

When the shift ends (consensus or 8-hour cap):

1. **Run the drift check** (git mode). If it fails, emit terminal-only summary

  and stop — do NOT set status to "completed" (leave as "drift-stopped").
2. Read `$RUN_DIR/state.json` as the source of truth.
3. Write the handoff to `$RUN_DIR/handoff.md`. Same location in both git and

  degrade mode — no new folder to create, no commit to make. The handoff
   lives alongside the run's state and artifacts so the whole run (plans,
   Codex outputs, handoff) is in one place.
4. Set run `status: "completed"` and `completed_at` in state.json.
5. **Do NOT delete the run folder.** Previous runs are kept as history under

  `.night-shift/runs/`. Each run's `handoff.md` is the authoritative
   user-facing record for that run.

If the user wants the handoff committed to the repo, they can do that
manually after reviewing. The skill intentionally does not commit — the
handoff is private notes, not a repo artifact.

Handoff structure:

```markdown
# Night Shift Handoff — RUN_ID

## Summary
[2-3 sentences: what was accomplished overall, in terms of the large goals]

**Run ID:** RUN_ID
**Mode:** git / degrade
**Branch:** BRANCH              [git mode only]
**Started from:** BASE_COMMIT   [git mode only]
**Objective:** [text or "propose"]
**Commits:** N                  [git mode only]

## Goals

### Goal 1: [Large goal title] — Complete / Blocked / Partial
**What shipped:** [1-2 sentences at goal level]
**Tasks:** M completed / K total (origin: user-approved × J, autonomous × L)

#### Task 1.1: [Title] — Complete
- **Commit:** [short hash + message]   [git mode only]
- **Changes:** [key bullet]
- **Codex review:** Clean after N rounds

#### Task 1.2: [Title] — Blocked
- **Why:** [what Codex flagged or test failed]
- **State:** reverted (git) / partial-in-tree (degrade)

#### Task 1.3 [autonomous]: [Title] — Complete
- (same fields — the [autonomous] tag marks additions the user did not explicitly approve)

**Decisions made (goal-level):**
- [judgment calls made without the user]

### Goal 2: ...

## Test Results
- Passing: X/Y
- New tests added: N

## Items Needing Human Attention
- [Anything you were unsure about]
- [Decisions that should be validated]
- [Tasks Codex flagged that you disagreed with]
- [Tasks that were blocked/reverted and why]
- [Autonomous additions — user should evaluate separately]
- [Degrade-mode: any tasks that left partial changes in working tree]
- [Codex-unavailable tasks, if any]

## How to Review                                  [git mode only]
```bash
# All night shift commits (one per task, grouped under goals in this handoff)
git log BASE_COMMIT..HEAD --oneline

# Full diff
git diff BASE_COMMIT

# Revert a specific task's commit
git revert <commit-hash>

# Or undo all night shift work
git reset --hard BASE_COMMIT
```

## Recommendations for Next Session

- [What to work on next]
- [Any tech debt introduced]

```

Terminal summary after writing the handoff:

```

Night shift complete! Handoff: .night-shift/runs/RUN_ID/handoff.md
Run folder kept at: .night-shift/runs/RUN_ID/

Branch: BRANCH (N new commits since BASE_COMMIT)
Run `git log BASE_COMMIT..HEAD --oneline` to review.

Completed: N/total goals

- Goal 1: [title]
- Goal 2: [title]
- Goal 3: [title] (blocked — see handoff)

```

## Stop / Resume / Abandon (active-shift re-trigger)

When a start/go trigger fires AND an active shift already exists, prompt the
user:

```

Found an active night shift:
  Run ID: RUN_ID
  Branch: BRANCH
  Started: STARTED_AT
  Progress: X/Y key results complete, key result Z in_progress (phase: PHASE)

What do you want to do?
  [1] Stop     — run full end procedure on the active shift
                 (handoff + archive). Then you can start a new one.
  [2] Resume   — revert the in-progress task to its task boundary,
                 continue iterative key result decisions from where we left off.
  [3] Abandon  — leave the repo state as-is, mark this run "abandoned",
                 start a new shift. No handoff. Old folder kept as history.

```

### Stop

Identical to §Ending a Shift Early below — the shared end procedure.

### Resume (task boundary)

1. Run drift check against the old run's `expected_head` and `branch`. If drift
   detected → switch to Stop path (safer than trying to resume into unknown
   state). Warn the user.
2. Read state.json. Find the task with `status: "in_progress"` (there is
   at most one across the whole run, since tasks run sequentially).
3. If an in-progress task exists:
   - **Git mode:** scoped rollback of just that task to its
     `start_commit`. Set its status to `reverted-on-resume`. Update
     `expected_head` to the new HEAD. Completed tasks in the same goal
     stay committed — Resume does NOT revert past work.
   - **Degrade mode:** scoped rollback unavailable — warn the user that the
     in-progress task's partial changes remain in the working tree. Set
     its status to `reverted-on-resume` (conceptually; the user must clean
     up manually).
4. Continue the execution loop from the next `pending` task in the same
   goal (or the first task of the next goal if the current goal is
   exhausted). Do NOT re-confirm goals or task lists — they were already
   approved / decomposed.
5. Record a `resumed_at` timestamp in state.json so the eventual handoff can
   note the discontinuity.

### Abandon

1. Warn the user: "Abandoning an active shift leaves the in-progress goal's
   changes in the working tree — you may want to review or stash them before
   starting a new shift."
2. Set `status: "abandoned"` and `abandoned_at` in the old run's state.json.
3. Do NOT write a handoff, do NOT revert commits, do NOT touch branch state.
4. Proceed to §Pre-flight as if starting fresh.

## Ending a Shift Early (Stop)

Triggered by "stop night shift", "end night shift", etc., when an active shift
exists. Also the Stop path from §Stop/Resume/Abandon.

1. Finish the current atomic operation — don't leave broken code mid-edit.
2. For the in-progress task:
   - If it has **already passed both Inner 4 (Codex review clean) and
     Inner 5 (tests pass)** and all structural-gate requirements are met,
     commit it normally via Inner 6.
   - Otherwise, **revert it** (git mode) or mark blocked and leave partial
     changes (degrade mode). The Codex review requirement and file gate are
     NOT waived by early stop.
3. Mark the containing goal's status appropriately (`completed` if all its
   tasks are terminal, else `blocked`).
4. Write the handoff note with current progress, clearly marking any
   interrupted task as "interrupted — reverted" or "interrupted —
   partial changes in tree".
5. Run the standard Handoff procedure (commit handoff in git mode, leave
   file at cwd root in degrade mode).
6. Set run `status: "interrupted"` (not `completed`) and `ended_at` in
   state.json.
7. Print terminal summary as in Handoff.

## Non-Git Degrade Mode

When night shift runs outside a git repo, it degrades gracefully. The execution
loop and Codex reviews still run, but these guarantees are gone:

| Feature             | Git mode | Degrade mode |
|---------------------|----------|--------------|
| Commit per task | Yes      | No — tasks just update state.json |
| Scoped rollback     | Yes      | No — failed tasks leave partial changes |
| Drift check         | Yes      | No — can't detect external branch/HEAD changes |
| Every-commit-green  | Yes      | N/A — nothing is committed |
| Structural file gate | Yes     | Yes — `code-review.txt` still required |
| Handoff location    | `.night-shift/runs/RUN_ID/handoff.md` (file only) | `.night-shift/runs/RUN_ID/handoff.md` (file only) |

Prominently flag the limitations in the handoff. If a task fails
validation in degrade mode, its partial changes stay in the working tree —
the user must review and clean them up manually.

## Error Recovery

- **Build breaks:** Fix before moving on. If you can't fix in 2 attempts, revert
  the task (git mode) or mark it blocked (degrade mode) and move on.
- **Merge conflicts:** Should not happen since no one else is committing to
  this branch. If they do, treat as drift — stop the run and write the handoff.
- **Codex unavailable:** See §Codex Unavailability.

## Principles

1. **Every commit must be green.** (Git mode.) Never commit code that doesn't
   pass tests. Revert rather than leave broken code.
2. **Leave no broken windows.** Never leave the codebase in a worse state than
   you found it. Use scoped rollback to undo only the failed task's changes.
3. **Document your judgment calls.** Every decision made without the human
   goes in state.json and the handoff.
4. **Trust the review loop.** Claude↔Codex review cycle is the quality gate.
   If the gate can't run, flag it loudly. The structural file gate exists so
   that a compacted, forgetful agent cannot quietly skip it.
5. **Refresh the rules.** Re-read `INVARIANTS.md` and `state.json` at the top
   of every task. Your memory of this skill will be wrong after a few
   hours; the disk is the source of truth.
6. **Stay in your lane.** Only work within approved goals. Task expansion
   must stay in the goal's subject matter; autonomous additions must be flagged.
7. **Never push.** Never push to remote. The human decides when to push.
8. **State file is truth.** Always update and re-read state.json. Don't trust
   your memory for multi-hour runs.
9. **Runs are independent.** Each shift gets its own folder under
   `.night-shift/runs/RUN_ID/`. Previous runs are history, not to be modified
   or appended to.
10. **End only on consensus or cap.** The shift ends either when Codex and
    the agent agree further work would be over-engineering, or when 8
    hours have elapsed. The agent cannot end on its own. See §End Conditions.
```

