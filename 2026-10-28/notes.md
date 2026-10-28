# 2026-10-28

## What I worked on

- **CPU Inference Lab:** finished and published the local LLM deployment/benchmark project (M0 baseline). The full project lives in [`cpu-inference-lab/`](cpu-inference-lab/).

## What the project is

A small, reproducible lab that answers one question: **what actually limits LLM speed on a plain desktop CPU?**
The method is *predict first, measure second, explain the gap*:

1. Write predictions (with the expected mechanism) before running anything.
2. Run the benchmark with a pinned llama.cpp build and a pinned, hashed model.
3. Keep every raw output and the environment snapshot.
4. Recompute the statistics from raw samples and explain any miss separately.

> Note on naming: the folder in the earlier GitHub repo is called `ollama_lab`, but the benchmark itself uses **llama.cpp's `llama-bench` directly**, not Ollama. Ollama is built on llama.cpp, so the numbers are relevant to it, but Ollama was not what was measured here.

**Setup:** Intel Core i5-12400F (6C/12T), 16 GB DDR4-2667 in a single stick (single-channel), Windows 11, Balanced power plan.
**Model:** Qwen2.5-0.5B-Instruct, Q4_K_M quantization (491 MB GGUF). **Engine:** llama.cpp build b11871, CPU only.

## Tasks (one commit/push each)

1. Project overview and notes (this file)
2. Environment recording script + snapshots
3. Benchmark runner + raw results
4. Analysis script + tests
5. Pre-registered predictions + artifact hashes
6. Results explained, and the interesting miss
7. Update the main log

---

## Task 2: Environment recording (`scripts/record_env.py`)

**Why:** a benchmark number is meaningless without the machine it came from. Before any run, this script writes a JSON snapshot of the environment, stored beside the results.

**What it captures:** OS and Python version, CPU model/cores/threads (via PowerShell `Win32_Processor`), RAM size plus each memory stick's capacity and speed (`Win32_PhysicalMemory`), the active Windows power plan (`powercfg`), the llama.cpp `--version` output, and the git commit of the repo.

**Design choices worth remembering:**
- Every probe is best-effort. If one fails, its field stores the error instead of crashing, and a `warnings` list is written at the end.
- It never assumes things: memory type codes are mapped with a lookup that falls back to "unknown", and stick count is explicitly **not** treated as channel count (one stick means single-channel, but two sticks still need checking against the slot labels).
- Output files are timestamped and never overwritten: `experiments/env/env_<timestamp>_<label>.json`.

**Result:** 4 snapshots committed (`baseline`, `baseline-llama`, `baseline-llama-clean`, `m0-bench`). The `m0-bench` one belongs to the final benchmark session.

---

## Task 3: Benchmark runner and raw results (`scripts/m0_bench.py`)

**The command it runs:**

```
llama-bench -m qwen2.5-0.5b-instruct-q4_k_m.gguf -p 512 -n 128 -t 6 -ngl 0 -r 5 -o json
```

| Flag | Meaning |
|---|---|
| `-p 512` | **Prefill** test: process a 512-token prompt (`pp512`) |
| `-n 128` | **Decode** test: generate 128 tokens one at a time (`tg128`) |
| `-t 6` | 6 threads, one per physical core |
| `-ngl 0` | 0 layers on GPU, so this is pure CPU |
| `-r 5` | 5 repetitions inside each process |
| `-o json` | machine-readable output, with every per-repetition sample |

**Why three separate processes?** Repetitions inside one process share a warm cache and a loaded model. Starting llama-bench three times shows whether results move between fresh starts. Total: 3 processes x 5 reps = 15 samples per test.

**What the runner does:**
1. `--dry-run` previews the commands and checks that the exe and model exist.
2. Runs `record_env.py` first.
3. Before each process, records free RAM and CPU load (so a noisy run can be spotted afterwards).
4. Saves the **untouched** stdout (JSON) and stderr per run, plus a `_meta.json` with timestamps, wall time and return codes.
5. Refuses to overwrite any existing file (`open(..., "x")`).

**Result:** session `m0_20261025_123707` - 3 JSON files, 3 stderr logs, 1 meta file. All return codes 0; pre-run CPU load was 3.8-16.4%, so the machine was mostly idle. The stderr logs confirm the CPU backend loaded was `ggml-cpu-alderlake.dll`.

---

## Task 4: Analysis script and tests (`scripts/analyze_m0.py`, `tests/`)

**What it does:** reads the raw llama-bench JSON and recomputes statistics from the per-repetition `samples_ts` (tokens/s), rather than trusting the tool's summary.

- **Mean, sample standard deviation (n-1), CV%** (CV = stdev / mean, a unit-free measure of run-to-run noise), min and max.
- Reported **per process** and **pooled** over all 15 samples, plus the spread of the three process means.
- **Consistency check:** warns if threads, batch size, KV-cache type, model file, build, etc. differ between rows that are about to be pooled.
- **Cross-check:** compares its recomputed mean/stdev with llama-bench's own `avg_ts` / `stddev_ts` and warns on a mismatch.
- For decode only, prints an *assumption-dependent* implied bandwidth (`tok/s x model bytes`), clearly labelled as not a measurement.
- It deliberately does **not** compare against the predictions; that is the postmortem's job.

**Tests (`pytest`, 5 passing):** use tiny synthetic numbers that can be verified by hand, e.g. samples `1, 2, 3` give mean 2, stdev 1, CV 50%. They also check test naming (`pp512`, `tg128@d2048`), numeric run ordering (`run10` after `run2`), grouping by test, and that a deliberate mismatch is flagged.

**Output on the real session:**

```
== pp512 ==  pooled n=15  mean 344.45  stdev 15.66  CV 4.55%  range 326.64 - 380.03
== tg128 ==  pooled n=15  mean  45.42  stdev  1.47  CV 3.23%  range  42.76 -  46.94
```

---

## Task 5: Pre-registered predictions and pinned artifacts (`docs/`)

**Predictions were written on 2026-10-25, before any benchmark run, and the file is never edited afterwards.** Corrections go in a separate `postmortem.md`. This stops me from quietly "predicting" what I already saw.

**The arithmetic behind the main prediction:**
- Single-channel DDR4-2667 peak = 2667 x 10^6 transfers/s x 8 bytes = **21.3 GB/s**.
- Decoding one token needs (roughly) one pass over all model weights, so ceiling = 21.3 GB/s / 0.4914 GB = **about 43 tok/s**.
- Real bandwidth is below peak, so I predicted **25-38 tok/s**.

| # | Prediction | Mechanism expected |
|---|---|---|
| 1 | Decode 25-38 tok/s | Weight reads through single-channel DRAM |
| 2 | Prefill 300-1,000 tok/s | Weights reused across 512 tokens, compute-bound |
| 3 | Decode CV below 5% | Only OS/background noise |
| 4 | Decode stops scaling near 4 threads | Memory bandwidth saturates (later experiment) |
| 5 | Q4_K_M / Q8_0 decode ratio 1.0-1.375 | Fewer bytes, but dequantization costs (later experiment) |
| 6 | Biggest risk: real DRAM traffic per token differs from file size | Weight repacking, cache reuse |

**Pinned artifacts (`artifact_hashes.txt`):** size, SHA-256 and source URL of the llama.cpp release zip (24,216,651 B) and the model GGUF (491,400,032 B), so anyone can verify they are benchmarking the same bytes. `llama-version_b11871.txt` and `llama-bench-help_b11871.txt` capture the exact tool version and its options.

Large files (`*.gguf`, `models/`, `tools/`, `.venv/`) are excluded by `.gitignore`.
