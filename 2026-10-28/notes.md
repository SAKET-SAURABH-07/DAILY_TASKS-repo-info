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
