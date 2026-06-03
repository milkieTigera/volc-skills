---
name: benchmark-eval
description: Use when planning, launching, monitoring, or summarizing high-throughput benchmark evaluation for ML models, robot policies, simulation workloads, dataset tasks, or other repeated experiments. Use when the request involves multi-GPU task sharding, many workers, smoke tests, headless evaluation, CPU/GPU utilization tuning, resumable launch sessions, model-agnostic eval wrappers, expected-count verification, log parsing, or benchmark result summaries.
---

# Benchmark Eval

## Overview

Run benchmark evaluation as a scheduling and verification problem, not as a model-specific script. Keep model loading or system-under-test behavior behind a small adapter, then shard benchmark units across devices and summarize results from structured logs.

## Evaluation Boundary

Separate these layers before optimizing throughput:

- **System adapter**: checkpoint or artifact path, tokenizer/processor, data preprocessing, action or prediction API, result schema, seeds, and any model-specific environment variables.
- **Benchmark scheduler**: benchmark names, suites, task ranges, repetitions, seeds, worker count, device assignment, CPU thread limits, logging, monitoring, and summary parsing.

The scheduler should only require an eval command that accepts a shard definition and output path. Prefer one of these interfaces:

```text
--benchmark <name>
--task-start <inclusive_index>
--task-end <exclusive_index>
--num-trials <n>
--seed <seed>
--log-dir <log_dir>
--device <logical_device>
```

or:

```text
--benchmark <name>
--shard-id <i>
--num-shards <n>
--num-trials <n>
--seed <seed>
--log-dir <log_dir>
--device <logical_device>
```

If an eval script does not expose a shardable interface, add a thin wrapper or minimal patch before full fan-out. Do not bake checkpoint names, model families, benchmark suite names, or action horizons into the scheduler.

## Preflight

Before launching a full run:

1. Confirm benchmark assets, datasets, simulators, and config files are local to the target host or reachable through an approved storage path.
2. Confirm the Python environment can import the benchmark package, model package, eval entrypoint, and required runtime libraries.
3. Confirm output directories live on durable high-capacity storage, not `/`, `/root`, or a small home directory.
4. Use headless or noninteractive rendering for simulation benchmarks unless explicitly debugging visuals.
5. Record the run id, code commit, model artifact id, benchmark version, data version, seed policy, and expected result schema.
6. Run a one-shard smoke test before full fan-out.

Smoke test skeleton:

```bash
CUDA_VISIBLE_DEVICES=0 python path/to/eval.py \
  --benchmark <benchmark_name> \
  --task-start 0 --task-end 1 \
  --num-trials 1 \
  --seed 0 \
  --device 0 \
  --log-dir <log_root>/smoke
```

The smoke must load the system under test, execute at least one benchmark unit, write the expected result fields, and exit successfully.

## Workload Accounting

Write down the expected work before launching:

```text
BENCHMARKS=<benchmark_or_suite_names>
TASKS=<task_count_per_benchmark>
TRIALS=<trials_or_samples_per_task>
SEEDS=<seed_count>
EXPECTED_UNITS=benchmarks * tasks * trials * seeds
EXPECTED_WORKERS=<worker_count>
EXPECTED_LOGS=<worker_count or benchmark-defined count>
```

Prefer one task or one small shard per worker when memory permits. This keeps devices busy and isolates long-tail tasks. If startup overhead is high, group adjacent tasks into larger shards.

## Worker Plan

Choose concurrency from resource measurements, not guesswork:

```text
workers_per_gpu * peak_vram_per_worker < 0.85 * gpu_vram
workers_per_host * cpu_threads_per_worker <= available_cpu_threads
```

Start with conservative defaults, then increase only after smoke and pilot runs:

```text
TASKS_PER_WORKER=1
WORKERS_PER_GPU=1..5
TRIALS_PER_TASK=<benchmark_default_or_requested_value>
OMP_NUM_THREADS=1..4
MKL_NUM_THREADS=1..4
OPENBLAS_NUM_THREADS=1..4
NUMEXPR_NUM_THREADS=1..4
```

Always set `CUDA_VISIBLE_DEVICES=<physical_gpu>` per worker and pass a single logical device such as `--device 0` inside that worker. Increase `TASKS_PER_WORKER` or reduce `WORKERS_PER_GPU` when memory, CPU, or simulator startup overhead becomes the bottleneck.

## Launch Pattern

Use `tmux`, `screen`, a job scheduler, or another resumable execution environment for full runs. Stagger worker launches by 1-3 seconds to avoid checkpoint, dataset, or filesystem read storms.

Keep benchmark-specific information inside `eval_cmd`, not in the sharding loop:

```bash
launch_worker() {
  local gpu="$1" benchmark="$2" start="$3" end="$4" name="$5" out="$6"
  (
    export CUDA_VISIBLE_DEVICES="$gpu"
    eval_cmd "$benchmark" "$start" "$end" "$name"
  ) >"$out" 2>&1 &
}

eval_cmd() {
  local benchmark="$1" start="$2" end="$3" name="$4"
  python path/to/eval.py \
    --benchmark "$benchmark" \
    --task-start "$start" \
    --task-end "$end" \
    --num-trials "${TRIALS_PER_TASK:-1}" \
    --seed "${BASE_SEED:-0}" \
    --log-dir "$LOG_ROOT/$benchmark" \
    --device 0 \
    --run-id "$RUN_ID-$name"
}
```

Round-robin shards across GPUs:

```bash
gpu=0
for benchmark in $BENCHMARKS; do
  for start in $(seq 0 $((TASK_COUNT - 1))); do
    end=$((start + 1))
    name="${benchmark}_tasks${start}_${end}_gpu${gpu}"
    launch_worker "$gpu" "$benchmark" "$start" "$end" "$name" "$LOG_ROOT/$name.out"
    gpu=$(((gpu + 1) % GPU_COUNT))
    sleep "${LAUNCH_STAGGER_SECONDS:-1}"
  done
done
```

## Monitoring

Track three surfaces:

```bash
nvidia-smi --query-gpu=index,utilization.gpu,memory.used --format=csv,noheader,nounits
ps -eo pid,ppid,pcpu,pmem,cmd | grep -E '<eval_entrypoint_or_run_id>' | grep -v grep
grep -R -n -E 'CUDA out of memory|Killed|RuntimeError|Traceback|failed .*status=|Error executing job' "$LOG_ROOT"
```

Progress should come from worker logs or structured result files:

```bash
grep -R -n -E 'completed|progress|Final results|Overall|success|accuracy|score' "$LOG_ROOT"
```

Expect utilization to fluctuate for CPU-bound simulators, dataloaders, or evaluation loops with intermittent model inference. In the tail phase, utilization naturally drops as short shards finish. Do not launch duplicate tail workers unless the user explicitly wants repeated seeds.

## Result Summary

Prefer structured outputs such as JSONL, CSV, or benchmark-defined result files. If the benchmark only prints text logs, write or use a small independent parser before full runs so summary logic is not tied to the model implementation.

The summary should report:

- benchmark or suite name
- task or shard id
- seed
- episodes, examples, trials, or samples evaluated
- success count, accuracy, score, reward, latency, or benchmark-specific primary metric
- failed or missing shards
- aggregate metric and per-benchmark metrics

Verify aggregate counts against `EXPECTED_UNITS` before reporting a score.

## Completion

Consider a run complete only when all are true:

- No expected eval worker processes remain.
- Detached session or job scheduler reports completion.
- Expected worker logs, pid files, and result files are present.
- Summary counts equal expected benchmark units.
- No real error lines remain: OOM, `Killed`, `RuntimeError`, `Traceback`, launcher failure, or job execution errors.
- The independent summary parser reproduces the benchmark's reported aggregate metrics.

Simulation or rendering libraries may print cleanup warnings during interpreter shutdown. Treat them as teardown noise only when the worker exited successfully, wrote final results, and the expected counts include that worker.

## Common Mistakes

- Running one worker per GPU when each worker is mostly CPU-bound or simulator-bound.
- Increasing concurrency after workers finish, which duplicates shards and corrupts reproducibility unless extra seeds are intentional.
- Saving videos, traces, or heavyweight artifacts during throughput runs unless required by the benchmark.
- Passing physical GPU ids inside a worker after setting `CUDA_VISIBLE_DEVICES`; use the logical device inside the worker.
- Mixing model-specific flags into the launcher loop instead of isolating them in an adapter function or config.
- Reporting the benchmark metric before checking missing shards, failed workers, and expected counts.
- Keeping logs, caches, checkpoints, or result artifacts under `/`, `/root`, or small home directories.
