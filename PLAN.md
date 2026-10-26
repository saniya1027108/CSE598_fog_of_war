# Project plan (proposal)

> Fill this in and submit your repository's link on Canvas by **Fri Oct 23, 11:59 PM**.
> Keep it short: about one page. You may update the plan later; record changes in `JOURNAL.md`.

- **Name:** Saniya Mulla
- **GitHub username:** saniya1027108
- **Repository is private, with @3rdAT and @Slash0BZ as collaborators:** yes

## 1. The baseline, reproduced

Command (CPU, 4 workers, seed 0). Stockfish 19 supplied `bot:guess-sf-4x500`. The shipped weights were not retrained.

`python -m fow.arena agents/baseline bot:greedy bot:guess-sf-4x500 --games 100 --workers 4 --seed 0`

| Opponent | Score | Notes |
|---|---|---|
| bot:greedy | 77 / 100 | all 100 games ended by king capture; no draws |
| bot:guess-sf-4x500 | 68 / 100 | same; reference rating of this bot is pinned at 1060 |

Local rating of the baseline: **1200**, 95% interval **[1144, 1267]**. The published local rating (1217, from 600 games) sits inside that interval. Both opponents kept their fixed reference ratings (greedy 1000, guess-sf-4x500 1060).

## 2. What you will try first

The shipped agent is a 314K-parameter residual CNN that imitates full-board Stockfish and plays its top legal move. The first three experiments change one thing at a time and stay near that size, under the 1M parameter floor of the size-adjusted rating. The pre-tournament entry (Nov 17) will be whichever of these wins a controlled comparison. Self-play is reserved for after that, if these do not move the rating.

### Idea 1: more imitation data, same network
- **What:** regenerate labels with `baselines/imitation/generate_data.py --games 20000` (the course Sol script), then train the same `PolicyNet(channels=64, blocks=4)` for 8–10 epochs into a copy of `submission/`.
- **Why you expect it to help:** the shipped model saw about 83,000 positions and matched its teacher on 39.4% of a held-out set. If it is data-limited, the same recipe with more games should raise both teacher agreement and Elo. The architecture stays at 314K parameters, so it keeps the full small-model bonus.
- **How you will measure it:** the same arena command as section 1, with `--seed 1`, against `bot:greedy` and `bot:guess-sf-4x500`, 100 games each. Compare the 95% interval to [1144, 1267]. Also report validation top-1 agreement with the teacher. Adopt it only if a 200-game head-to-head against `agents/baseline` favors the new weights.

### Idea 2: labels from a fog-consistent teacher
- **What:** one change from idea 1. Replace the full-board Stockfish label with the move a guess-style teacher picks: sample a few boards consistent with the observation and the memory, run Stockfish on each, and take the majority legal move. Same network, same position budget.
- **Why you expect it to help:** the current teacher sees pieces the agent cannot see, so it teaches moves that only make sense with hidden information. The game has no check, and the win is capturing the king, so those labels are a biased teacher.
- **How you will measure it:** 200-game head-to-head against the idea 1 agent, plus the section 1 arena against the two reference bots. Keep idea 2 only if it wins the head-to-head or its reference-bot interval separates from idea 1.

### Idea 3: a value head and a few belief samples at play time
- **What:** add a value head on the same trunk, trained to predict which side eventually captured the king. In `act`, score each candidate with the policy plus the value of a handful of boards sampled from memory (on the order of 4 forwards). No engine calls at play time.
- **Why you expect it to help:** the baseline plays one forward pass (about 4×10⁷ FLOPs) and the limit is 10¹², so a few extra passes fit easily. Search is the gap the baseline README names, and it can reject a top policy move that walks the king into an unseen attack.
- **How you will measure it:** an ablation on the same weights, policy-only versus policy plus value, 100 games each against the two reference bots. Keep the search only if the intervals separate.

## 3. Compute plan

Training stays on the Sol class account (`-q class`). The account cap is 960 GPU-minutes (16 GPU-hours), with at most 2 jobs running. Data generation does not need a GPU.

| Step | Where | Expected cost |
|---|---|---|
| Idea 1 labels, 20,000 games | `scripts/sol/generate_data.sbatch` (32 CPUs, 4 h cap) | 0 GPU-hours |
| Idea 1 training, 8–10 epochs | `scripts/sol/train.sbatch`, 1 GPU | under 0.5 GPU-hour (the shipped run was 12 minutes on 2 CPU cores) |
| Idea 2 labels and retrain | one more CPU job, then one GPU job | under 0.5 GPU-hour |
| Idea 3 value head | one GPU job on data already generated | under 0.5 GPU-hour |
| Evaluation | local or a CPU Sol job, `fow.arena` | 0 GPU-hours |

That is about 1.5 GPU-hours before Nov 17, leaving most of the cap for a later self-play run if the rating does not move. `reproduce.sh` will stay this same pipeline with fixed seeds, on the order of a few hours on Sol, not days. Each run’s GPU-hours go in `JOURNAL.md`.

## 4. Risks and questions

- More data may only copy the full-information teacher more accurately. If idea 1’s Elo does not move, idea 2 is the next entry rather than a bigger network.
- Fog labels may be noisier and lower both teacher agreement and Elo. The pre-tournament entry then stays the idea 1 weights. A weak valid entry still scores the Nov 17 points.
- At 100 games the interval here was about ±60 Elo. A gap smaller than that is not a result. Head-to-heads of at least 200 games decide what replaces `submission/`.
- Do not grow past a few million parameters until the size-adjustment coefficient β is announced. Models under 1M share the same bonus.
- This reproduction used Stockfish 19. The shipped labels used Stockfish 16. The training write-up will pin the binary.

