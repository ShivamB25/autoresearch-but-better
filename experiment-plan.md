# Experiment Plan

This is the concrete companion to `program.md`. Read `program.md` for the operating rules, then use this file for the first wave of experiments.

This repo is a fixed wall-clock optimization problem. The first job is not to import upstream wins blindly. The first job is to learn what works on the target CUDA machine.

Use this plan on the current branch unless the human says otherwise.

## Record every run

For every experiment, record:
- `val_bpb`
- `num_steps`
- `mfu_percent`
- `peak_vram_mb`
- `training_seconds`
- exact knob changes
- short hypothesis

## Keep / discard rules

- `Δ > 0.005` → probably real, keep candidate
- `0.001–0.005` → repeat before trusting
- `< 0.001` → likely noise unless repeated or clearly simpler
- crash / OOM → log and revert

## Promotion policy

1. **Search run**: default 5-minute run
2. **Repeat run**: same config once more
3. **Longer confirmation**: 15 minutes
4. **Final confirmation**: 30 minutes if still promising

Do not long-run everything. Only promote the strongest candidates.

## First 10 experiments

### Phase A — local truth

1. **baseline-1**
   - Change: none
   - Goal: establish first local anchor

2. **baseline-2**
   - Change: none
   - Goal: estimate repeatability

3. **baseline-3**
   - Change: none
   - Goal: finish rough noise estimate

### Phase B — batch frontier

4. **batch-down-1**
   - Change: `TOTAL_BATCH_SIZE = 2**18`
   - Hypothesis: more optimizer steps in the same wall-clock budget beats smoother gradients

5. **batch-up-1**
   - Change: `TOTAL_BATCH_SIZE = 2**20`
   - Hypothesis: larger effective batch may improve optimization enough to offset fewer steps

6. **batch-down-2**
   - Change: `TOTAL_BATCH_SIZE = 2**17`
   - Hypothesis: if the current regime is strongly step-starved, going smaller may help again; if too noisy, it will regress

### Phase C — device batch / accumulation frontier

Use the best `TOTAL_BATCH_SIZE` from Phase B.

7. **device-batch-low**
   - Change: `DEVICE_BATCH_SIZE = 64`
   - Hypothesis: more accumulation may improve stability, though likely at a throughput cost

8. **device-batch-high**
   - Change: `DEVICE_BATCH_SIZE = 256`
   - Hypothesis: fewer microsteps may improve throughput enough to help, if VRAM allows

### Phase D — capacity frontier

Use the best batch regime from Phases B and C.

9. **depth-down**
   - Change: `DEPTH = 6`
   - Hypothesis: the current model may be over-capacity for the available wall-clock budget

10. **depth-up**
    - Change: `DEPTH = 10`
    - Hypothesis: extra capacity may pay off if throughput loss is acceptable

## Next experiments after the first 10

Only after the first 10 are understood:

11. best batch regime + `ASPECT_RATIO = 48`
12. best batch regime + `ASPECT_RATIO = 80`
13. best compute regime + `WINDOW_PATTERN = "L"`
14. best compute regime + `EMBEDDING_LR = 0.8`
15. best compute regime + `WARMDOWN_RATIO = 0.7`

## What to avoid early

Do not start with:
- weight tying
- `n_kv_head = 1`
- GeLU replacement
- removing value embeddings
- large bundled imports of upstream wins
- multi-agent or population-search framework work

## Interpretation guide

- If smaller batch wins strongly, the machine is probably **step-starved**.
- If lower depth wins, the baseline is probably **too capacity-heavy** for the local budget.
- If higher depth wins, the machine has more room for capacity.
- If `WINDOW_PATTERN = "L"` beats the current pattern later, attention efficiency assumptions differ on this hardware.
- If most changes sit inside the noise floor, improve measurement quality before more tuning.
