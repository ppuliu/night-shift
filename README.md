# Night Shift

> Step away for the night.
>
> Claude Code keeps grinding.
>
> Wake up to well planned, committed, tested, reviewed work.

Night Shift is a Claude Code skill that turns a normal Claude Code session into an autonomous overnight development agent. You approve one thing — an **objective** — then walk away. The agent plans, implements, and commits work against a feature branch for up to 8 hours.

When OpenAI Codex is available, every commit is gated by an independent review from a second, separately-prompted LLM. When it isn't, the agent falls back to a clearly-marked self-review and keeps working — you lose the dual-LLM guarantee but not the loop.

```
═══════════════════════════════════════════════════════════════
  🌙  NIGHT SHIFT ENGAGED  🌙
═══════════════════════════════════════════════════════════════

  You can step away now. I'll take it from here.

  Run ID:      2026-04-19-2318
  Branch:      night-shift/harden-api-errors (from d9aad96)
  Objective:   Harden error handling across the API layer —
               standardize error responses, add retries to
               outbound HTTP, and cover the gaps with tests.
  Max runtime: 8 hours (or until Codex and I agree we're done)
  Handoff:     .night-shift/runs/2026-04-19-2318/handoff.md

  Sleep well. 🌙
═══════════════════════════════════════════════════════════════
```

<a href="https://www.loom.com/share/6bcfdd2579c74de5bdad595c686fa547" target="_blank" rel="noopener noreferrer"><img src="https://github.com/user-attachments/assets/1a95844b-51ca-4944-8918-2a49c3f3e83a" alt="Watch the Night Shift demo on Loom"></a>


## Why

Long-horizon autonomous coding agents fail in predictable ways: they hallucinate progress, paper over broken tests, escalate scope into unrelated refactors, sometimes push to `main`. Most "run it overnight" setups are one LLM grading its own homework.

Night Shift makes three structural bets:

1. **Two independent LLMs review each other, never the same one grading itself.** Claude writes; Codex (when available) adversarially reviews. A task cannot be committed until a real review artifact exists on disk — a compacted, forgetful agent cannot quietly skip the gate.
2. **Small, reversible units.** Every task is one commit. A failing review or test reverts only that task — not the whole session. Every commit on the branch passes tests.
3. **The agent can't decide it's done.** Ending on "we're done" requires both the agent proposing it *and* Codex agreeing. Without Codex, the shift runs to the 8-hour hard cap.

## How it works

Each task runs: Claude plans → Claude implements → Codex reviews the code (loops until clean, hard-stops and reverts at 10 rounds) → tests must pass → file-gated commit. The pre-commit gate refuses to stage a task whose `code-review.txt` is missing, empty, or doesn't match the verdict recorded in state.

Work is organized as:

- **Objective** — the one thing you approve.
- **Key Results** — concrete deliverables the agent proposes iteratively, each gated by Codex.
- **Tasks** — independently committable units under each key result.

The agent ends the shift only when it explicitly proposes "we're done" and Codex agrees on a re-review of the full objective and key-result history. Repeated Codex rejections of new proposals are a signal to invoke that path — never a substitute for it.

## Requirements

- **Claude Code** launched with `--dangerously-skip-permissions` in a directory you trust. Without it, the autonomous loop stalls on permission prompts.
- **Git** (recommended). Outside a git repo, Night Shift runs in a graceful-degrade mode with no commits, no rollback, no drift protection.
- **OpenAI [Codex CLI](https://github.com/openai/codex)** (recommended). When present, you get the dual-LLM review loop and the ability to end on consensus. When absent, Claude self-reviews with explicit `CODEX UNAVAILABLE` markers on every artifact, the handoff flags those tasks prominently, and the shift runs to the 8h cap.

## Install

From inside Claude Code:

```
/plugin marketplace add ppuliu/night-shift
/plugin install night-shift@night-shift
```

Or clone directly into your skills directory:

```bash
git clone https://github.com/ppuliu/night-shift.git ~/.claude/skills/night-shift
```

Restart Claude Code (or start a new session). The skill auto-registers.

## Quickstart

In the project you want worked on, launch Claude Code with `--dangerously-skip-permissions`. Make sure you're on a feature branch (Night Shift offers to create one if you're on `main`) and your working tree is clean. Then:

```
/night-shift
```

The skill asks you, one question at a time: bypass-permissions confirmation, branch choice, uncommitted-changes handling, and an **objective** (or type `propose` to have the agent suggest one from session context). After you confirm, the `NIGHT SHIFT ENGAGED` banner fires and the shift clock starts — the 8-hour cap measures from the banner, not from when you typed `/night-shift`.

To stop early: say `stop night shift`. To retrigger during an active shift, you get a Stop/Resume/Abandon prompt.

## In the morning

The authoritative artifact is:

```
.night-shift/runs/<RUN_ID>/handoff.md
```

It summarizes what shipped per key result, per-task commit hashes, Codex review rounds, decisions the agent made without you, and items needing human attention. Review the branch like any other PR.

Night Shift never runs `git push` and never opens PRs. You decide what ships.

## Run folder

```
.night-shift/runs/<RUN_ID>/
├── state.json           Source of truth for the run
├── handoff.md           Human-facing summary
└── key-results/<KR>/
    ├── codex-approval.txt          Was this KR worth doing?
    ├── decomp-adversarial.txt      Are these the right tasks?
    └── tasks/<T>/
        └── code-review.txt         Is the code clean?
```

Every verdict is a real file on disk. Previous runs are never modified.

## Safety model


| Failure mode                           | How Night Shift prevents it                                                  |
| -------------------------------------- | ---------------------------------------------------------------------------- |
| Agent commits broken code              | Full test suite runs before every commit; failures trigger scoped rollback   |
| Agent skips code review to "save time" | Pre-commit gate verifies review file exists and matches recorded verdict     |
| Agent grades its own homework          | Codex is a separate process; self-review (when needed) is explicitly flagged |
| Agent decides it's done prematurely    | Dual consensus required — Codex must agree, else 8h cap fires                |
| Agent runs indefinitely                | 8-hour hard cap, finished atomically on the current task                     |
| Your own work gets clobbered           | Drift check before every write; external HEAD change stops the run           |
| Failed task leaves repo dirty          | Per-task scoped rollback to the task's `start_commit`                        |
| Agent pushes to remote                 | The skill never calls `git push`                                             |
| Scope creeps beyond objective          | Every key result gated on "does this serve the objective?"                   |


The full ruleset is in `[INVARIANTS.md](INVARIANTS.md)`; the full execution spec is in `[SKILL.md](SKILL.md)`.

## Commands


| Trigger                                                          | Action                                              |
| ---------------------------------------------------------------- | --------------------------------------------------- |
| `/night-shift`                                                   | Start a shift, or Stop/Resume/Abandon an active one |
| `start night shift`, `going to sleep`, `take over for the night` | Same as `/night-shift`                              |
| `stop night shift`, `end night shift`, `wrap up`                 | Run the end procedure on the active shift           |


## Limitations

- **Wall clock, not compute time.** The 8h cap measures elapsed wall clock. If your machine suspends, that counts. `caffeinate` or disable sleep-on-AC if this matters.
- **No user questions mid-shift.** Ambiguities are resolved by the agent and logged in `state.json.decisions_made` — review in the handoff.
- **Single session per repo.** External commits to the feature branch trigger drift protection and stop the run cleanly.
- **Without Codex, no early exit.** The shift runs to the 8h cap and every task's review is explicitly self-reviewed.

