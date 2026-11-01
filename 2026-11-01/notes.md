# Today's study notes: Running Local LLMs (Chapter 3)

1 November 2026 · Based on Chapter 3: Running Local LLMs of the Local & Private AI Engineering Handbook (Part 1).

## 1. Ollama and its REST API (Listing 3.3)

- Ollama runs a model locally and exposes an API.
- A Python script can send a chat request to `http://localhost:11434/api/chat`.
- We looked at settings like `stream=False`, `temperature=0.2`, and `num_ctx=4096`.
- We also saw how the response's token counts and durations can be used to calculate generation speed.

## 2. Comparing local inference tools (Section 3.5)

We compared the tools and their main purposes:

| Tool      | Main use                                  |
| --------- | ----------------------------------------- |
| Ollama    | Easy local model running and management   |
| llama.cpp | More control and flexible model execution |
| LM Studio | GUI-based experimentation                 |
| vLLM      | High-throughput GPU serving               |

We also discussed how an OpenAI-compatible API can make switching between backends easier.

## 3. Measuring speed (Listing 3.5)

Inference has two distinct speed measurements:

- **Prompt processing (prefill):** processing the input prompt.
- **Token generation (decode):** generating the response one token at a time.

In the example, the results were 250 prompt tokens/sec and 20 generated tokens/sec. Those values came from the listing's sample response, not a real benchmark.
