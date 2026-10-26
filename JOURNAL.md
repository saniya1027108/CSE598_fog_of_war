# Project journal

One entry per experiment or decision, newest at the bottom. Write it as you go, not at the end.
This is where the evidence for your grade lives (see docs/GRADING.md).

## Template

### YYYY-MM-DD: short title
- **Question / hypothesis:** what you expect and why.
- **Commit:** the commit hash the experiment ran at, and what you changed.
- **Setup:** data, model size, training compute (GPU-hours), games played for evaluation.
- **Result:** numbers with intervals (for example `fow.arena` ratings), plus anything you saw in games.
- **Decision:** what you will do next, and why.
- **AI help:** what an assistant did for this entry, if anything.

---

### 2026-10-25: reproduced the baseline

- **Question / hypothesis:** the shipped baseline should land near its published local rating of 1217 when played against `bot:greedy` and `bot:guess-sf-4x500`.
- **Commit:** `64c4157` (template checkout). No agent code or weights were changed.
- **Setup:** `python -m fow.arena agents/baseline bot:greedy bot:guess-sf-4x500 --games 100 --workers 4 --seed 0` on CPU. Stockfish 19 for the guess bot. 0 GPU-hours. The network is the shipped 314,057-parameter policy.
- **Result:** 77/100 against `bot:greedy`, 68/100 against `bot:guess-sf-4x500`. Every game ended by king capture. Fitted rating **1200 [1144, 1267]**. The published 1217 is inside that interval. Reference ratings stayed pinned (greedy 1000, guess-sf-4x500 1060).
- **Decision:** first training run is more imitation data with the same network (PLAN.md idea 1). Do not change the architecture until that comparison is done.
