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
