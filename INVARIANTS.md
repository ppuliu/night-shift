# Night Shift Invariants (read before every task)

These rules are NON-NEGOTIABLE. Context compaction is not an excuse. "Time
efficiency" is not an excuse. "Simple change" is not an excuse.

1. **Drift check before every write** (git mode). If it fails, stop the run.

2. **Key-result proposal requires Codex approval** (Outer B). Every key
   result must pass an adversarial review asking "is this worth implementing
   toward the objective, or would it over-engineer / over-optimize?" before
   decomposition begins. Output MUST be saved to
   `key-results/<KR>/codex-approval.txt`.

3. **Decomposition requires Codex adversarial review** (Outer D). Saved to
   `key-results/<KR>/decomp-adversarial.txt`.

4. **Task code review is mandatory** (Inner 3). Loop until Codex reports no
   P1/P2. Saved to `key-results/<KR>/tasks/<T>/code-review.txt`.

5. **Pre-commit structural gate.** A task cannot be committed unless its
   `code-review.txt` exists, is non-empty, and contains a Codex verdict line
   that matches `code_review_evidence` in state.json.

6. **End of shift requires dual consensus** — agent proposes "done" AND
   Codex agrees (or the 8-hour hard cap fires). Agent self-consensus is
   not valid consensus. The ONLY valid "done" rationale is that any
   further KR would *actively over-engineer or over-optimize the
   literal objective*. NEVER end early on "diminishing returns",
   "deferred to ROADMAP / future shift", "natural stopping point",
   "foundation in place", or "not enough time left in the cap" —
   estimates are unreliable and the cap path handles its own case.
   If Codex is unavailable, wait for the 8h cap.

7. **If Codex is unavailable**, the agent MUST write a rigorous
   self-adversarial review to the same file with header
   `CODEX UNAVAILABLE — SELF-ADVERSARIAL REVIEW`, genuinely challenging
   the work. Agent self-approval without adversarial reasoning is a
   protocol violation. This does NOT apply to end-of-shift consensus
   (see #6).

8. **No questions to the user mid-shift.** Make judgment calls and record
   in `decisions_made` in state.json.

9. **Every commit must be green.** If tests fail, revert (git mode) or
   mark blocked (degrade mode). Never commit red code.

10. **Stay on the objective.** Every key result must serve it. Don't
    refactor unrelated code.

11. **Never push to remote.** The human decides.
