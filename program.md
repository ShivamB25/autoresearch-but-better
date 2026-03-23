# autoresearch

This is an experiment to have the LLM do its own research.

## Setup

To set up a new experiment, work with the user to:

1. **Stay on the current branch unless the human says otherwise.** Do not create a new branch by default in this repo.
2. **Read the in-scope files**: The repo is small. Read these files for full context:
   - `README.md` — repository context.
   - `prepare.py` — fixed constants, data prep, tokenizer, dataloader, evaluation. Do not modify.
   - `train.py` — the file you modify. Model architecture, optimizer, training loop.
3. **Verify data exists**: Check that `~/.cache/autoresearch/` contains data shards and a tokenizer. If not, tell the human to run `uv run prepare.py`.
4. **Initialize results.tsv**: Create `results.tsv` with just the header row. The baseline will be recorded after the first run.
5. **Initialize lab_journal.jsonl**: Create an empty `lab_journal.jsonl` file. Record one JSON line per experiment.
6. **Confirm and go**: Confirm setup looks good.

Once you get confirmation, kick off the experimentation.

## Experimentation

Each experiment runs on a single GPU. The training script runs for a **fixed time budget of 5 minutes** (wall clock training time, excluding startup/compilation). You launch it simply as: `uv run train.py`.

**What you CAN do:**
- Modify `train.py` — this is the only file you edit. Everything is fair game: model architecture, optimizer, hyperparameters, training loop, batch size, model size, etc.

**What you CANNOT do:**
- Modify `prepare.py`. It is read-only. It contains the fixed evaluation, data loading, tokenizer, and training constants (time budget, sequence length, etc).
- Install new packages or add dependencies. You can only use what's already in `pyproject.toml`.
- Modify the evaluation harness. The `evaluate_bpb` function in `prepare.py` is the ground truth metric.

**The goal is simple: get the lowest val_bpb.** Since the time budget is fixed, you don't need to worry about training time — it's always 5 minutes.

**VRAM** is a soft constraint. Some increase is acceptable for meaningful val_bpb gains, but it should not blow up dramatically.

**Simplicity criterion**: All else being equal, simpler is better. Keep small wins that simplify the code. Avoid tiny gains that add ugly complexity.

**The first run**: Your very first run should always be to establish the baseline, so you will run the training script as is.

**Baseline discipline**: Do not start by importing a bundle of upstream "wins". First measure the local baseline and repeat it enough times to understand the machine's noise floor.

## Output format

Once the script finishes it prints a summary like this:

```
---
val_bpb:          0.997900
training_seconds: 300.1
total_seconds:    325.9
peak_vram_mb:     45060.2
mfu_percent:      39.80
total_tokens_M:   499.6
num_steps:        953
num_params_M:     50.3
depth:            8
```

You can extract the key metrics from the log file:

```
grep "^val_bpb:\|^peak_vram_mb:" run.log
```

## Logging results

When an experiment is done, log it to `results.tsv` (tab-separated, NOT comma-separated — commas break descriptions).

The TSV has a header row and 5 columns:

```
commit	val_bpb	memory_gb	status	description
```

1. git commit hash (short, 7 chars)
2. val_bpb achieved (e.g. 1.234567) — use 0.000000 for crashes
3. peak memory in GB, round to .1f (e.g. 12.3 — divide peak_vram_mb by 1024) — use 0.0 for crashes
4. status: `keep`, `discard`, or `crash`
5. short text description of what this experiment tried

Example:

```
commit	val_bpb	memory_gb	status	description
a1b2c3d	0.997900	44.0	keep	baseline
b2c3d4e	0.993200	44.2	keep	increase LR to 0.04
c3d4e5f	1.005000	44.0	discard	switch to GeLU activation
d4e5f6g	0.000000	0.0	crash	double model width (OOM)
```

## Persistent memory

After every experiment, append one JSON line to `lab_journal.jsonl`.

Example:

```json
{"experiment": 1, "hypothesis": "halve batch size for more steps", "change": "TOTAL_BATCH_SIZE 2**19 -> 2**18", "val_bpb": 0.9860, "baseline_bpb": 0.9979, "delta": -0.0119, "status": "keep", "reasoning": "more optimizer steps in fixed time budget improves convergence"}
```

Before proposing a new experiment, read `lab_journal.jsonl` to avoid repeating the same idea.

In particular, use it to track:
- the local noise floor from repeated baseline runs
- which changes were throughput wins vs true quality wins
- which ideas looked good at 5 minutes but failed under longer confirmation runs

## Known dead ends (do not retry early)

These have been reported as repeated failures in upstream/community exploration. Do not spend early experiments on them unless you have a very specific reason:

- Weight tying (shared embed/unembed)
- Parallel attention + MLP
- `n_kv_head = 1`
- `x0_lambda` init at `0.0`
- Depth 10-11 with larger dim in the same 5-minute budget
- Replacing ReLU-squared with GeLU
- Very high learning rates (about 2x current baseline)
- Removing value embeddings

Treat this as a time-saving hint, not as law. Local hardware can differ.

## Search priorities

Search in this order unless the evidence strongly says otherwise:

1. **Throughput / step-count levers first**
   - `TOTAL_BATCH_SIZE`
   - `DEVICE_BATCH_SIZE`
2. **Capacity frontier second**
   - `DEPTH`
   - `ASPECT_RATIO`
3. **Attention cost structure third**
   - `WINDOW_PATTERN`
4. **Optimizer and schedule micro-tuning last**
   - `EMBEDDING_LR`
   - `UNEMBEDDING_LR`
   - `WARMUP_RATIO`
   - `WARMDOWN_RATIO`
   - `FINAL_LR_FRAC`
   - weight decay and init-scale details

The reason is simple: this repo is a fixed wall-clock optimization problem. The most important question is how many useful optimizer steps you get in 5 minutes on the current hardware.

## The experiment loop

The experiment runs on the current branch unless the human explicitly asks for a separate branch.

LOOP FOREVER:

1. Look at the git state: the current branch/commit you're on.
2. Read `lab_journal.jsonl` and form one specific hypothesis.
3. Tune `train.py` with that experimental idea. Prefer one axis at a time unless two changes are tightly coupled.
4. git commit.
5. Run the experiment: `uv run train.py > run.log 2>&1`.
6. Read out the results: `grep "^val_bpb:\|^peak_vram_mb:" run.log`.
7. If the grep output is empty, the run crashed. Run `tail -n 50 run.log` to read the Python stack trace and attempt a fix.
8. If it still fails after about 3 attempts, give up on that idea.
9. Record the result in `results.tsv` and `lab_journal.jsonl`.
10. If val_bpb improved clearly, keep the commit. If it is equal or worse, git reset back to where you started.

The idea is that you are a completely autonomous researcher trying things out. If they work, keep. If they don't, discard.

**Noise floor**: Treat tiny deltas with skepticism.
- `> 0.005`: probably real
- `0.001–0.005`: repeat before trusting
- `< 0.001`: likely noise unless repeated or clearly simpler

Repeat every promising config at least once before trusting it.

## Promotion ladder for longer confirmation runs

Use the default 5-minute run as the cheap search loop.

Only promote a config beyond that if it has already shown a real gain at 5 minutes.

Recommended ladder:
1. **Search run** — default 5-minute run
2. **Repeat run** — another 5-minute run with the same config
3. **Longer confirmation run** — 15 minutes
4. **Final confirmation run** — 30 minutes if still promising

Longer runs are for truth, not for broad search. Do not long-run everything.

If longer confirmation requires a local-only change to the time budget or execution environment, do it deliberately and do not confuse that with the default search configuration.

**Timeout**: Each experiment should take about 5 minutes total (+ startup and eval overhead). If a run exceeds 10 minutes, kill it and treat it as a failure.

**Crashes**: If a run crashes because of something dumb and easy to fix, fix it and re-run. If the idea itself is broken, log `crash`, revert, and move on.

**NEVER STOP**: Once the experiment loop has begun, do not pause to ask the human if you should continue. The loop runs until the human interrupts you.
