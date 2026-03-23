# CLAUDE.md

Repository instructions for Claude working in `autoresearch`.

## What this repo is

This repo is a small, fixed-budget LLM training setup for autonomous experimentation. The main job is to improve `val_bpb` under the default 5-minute training budget without making the codebase harder to reason about.

## Files that matter

- `train.py` — main editable file. Model, optimizer, hyperparameters, and training loop live here.
- `prepare.py` — data prep, tokenizer, dataloader, and evaluation harness. Treat as read-only unless a human explicitly asks otherwise.
- `program.md` — experiment workflow and operating rules.
- `experiment-plan.md` — concrete first-wave CUDA experiment plan.
- `README.md` — repo context and setup.
- `pyproject.toml` / `uv.lock` — dependency source of truth.

## Environment

- Python project, not Node.
- Package manager: `uv`.
- Python requirement: `>=3.10`.
- Expected hardware: one NVIDIA GPU with CUDA.
- This local machine may not be able to run CUDA training. Do not pretend local macOS results are equivalent.

## Working rules

- Stay on the current branch unless the human explicitly asks for a different branch.
- Default file to change: `train.py`.
- Do not modify `prepare.py` casually.
- Do not add dependencies unless the human explicitly asks for them.
- Keep `results.tsv` untracked.
- Keep long training logs in `run.log`, not in chat.

## Canonical commands

### Install dependencies
```bash
uv sync
```

### One-time data and tokenizer setup
```bash
uv run prepare.py
```

### Smaller setup validation
```bash
uv run prepare.py --num-shards 8
```

### Run one experiment
```bash
uv run train.py > run.log 2>&1
```

### Extract key metrics
```bash
grep "^val_bpb:\|^peak_vram_mb:\|^num_steps:\|^mfu_percent:\|^training_seconds:" run.log
```

### Inspect a failed run
```bash
tail -n 50 run.log
```

## Experiment discipline

- Use 5-minute runs for broad search.
- Establish the local baseline first.
- Repeat promising configs before trusting small deltas.
- Promote only shortlist candidates to longer runs.
- Record every run in `results.tsv` and `lab_journal.jsonl`.

Record these fields for each run:

- `val_bpb`
- `num_steps`
- `mfu_percent`
- `peak_vram_mb`
- `training_seconds`
- exact knob change
- short hypothesis

Keep / discard guidance:

- `Δ > 0.005` vs the relevant baseline or comparator: keep candidate
- `0.001–0.005`: repeat before trusting
- `< 0.001`: discard unless repeated or clearly simpler
- crash / OOM: log it and revert

## Search order for tuning

Unless evidence says otherwise, search in this order:

1. Throughput / step-count levers
   - `TOTAL_BATCH_SIZE`
   - `DEVICE_BATCH_SIZE`
2. Capacity frontier
   - `DEPTH`
   - `ASPECT_RATIO`
3. Attention cost structure
   - `WINDOW_PATTERN`
4. Optimizer and schedule details
   - `EMBEDDING_LR`
   - `UNEMBEDDING_LR`
   - `WARMUP_RATIO`
   - `WARMDOWN_RATIO`
   - `FINAL_LR_FRAC`
   - weight decay and init-scale details

This repo is a fixed wall-clock optimization problem. First learn what the target CUDA machine rewards. Do not start by importing a large bundle of upstream changes.

## Known constraints

- No formal test suite, linter, formatter, or CI is defined here.
- Validation is practical, not ceremonial:
  1. the code imports and starts
  2. the experiment completes
  3. `val_bpb` appears in the summary
  4. `peak_vram_mb` stays reasonable
  5. the diff remains understandable

## Style and implementation guidance

- Follow the existing style in `train.py` and `prepare.py`.
- Keep section-banner comments in place.
- Prefer small diffs.
- Prefer simple changes over clever frameworks.
- Do not refactor unrelated code while testing an experiment idea.
- Preserve machine-readable summary lines such as `val_bpb` and `peak_vram_mb`.

## Practical workflow

Before meaningful edits:

1. Read `README.md`, `prepare.py`, and `train.py`.
2. Verify cached data exists under `~/.cache/autoresearch/`.
3. Read `program.md` for the operating loop.
4. Read `experiment-plan.md` for the first experiment wave.

When running experiments:

1. Form one specific hypothesis.
2. Change one axis at a time unless two knobs are tightly coupled.
3. Run the experiment.
4. Extract metrics.
5. Log the result.
6. Keep or revert based on evidence.

## Definition of done

A solid change in this repo usually means:

- `uv run train.py` still works on the target CUDA machine
- the output is still easy to grep
- the change is small enough to understand quickly
- the complexity cost is justified by the result

When unsure, optimize for clarity, reproducibility, and minimal surface area.
