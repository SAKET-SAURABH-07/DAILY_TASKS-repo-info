# Today's study notes: Local LLM inference — complete revision

25 October 2026 · Based on our discussion of Chapter 2 of the Local & Private AI Engineering Handbook, plus our follow-up discussion about Q4, Q5 and Q8.

Today, we built the foundation for understanding how large language models run locally on your own PC, how much memory they need, and how to choose a model that fits your hardware.

I'll organize everything into concepts, formulas, examples, and practical rules so you can use this as your revision sheet.

## 1. What is a local LLM?

An LLM (Large Language Model) is a neural network trained to process and generate text. A local LLM runs on your own computer instead of relying entirely on a remote model server.

For example, you could run a model using Ollama on your Windows PC.

The main things that determine whether it runs well are:

- The model's parameter count and weight precision.
- Available system RAM and GPU VRAM.
- The context length and number of simultaneous users.
- The inference software and model format.
- The computer's processing power and memory bandwidth.

Running a model locally doesn't automatically mean it will be fast. A model might fit into your memory but still generate text slowly.

## 2. Tokens and the context window

### What is a token?

A token is a unit of text processed by an LLM. It might be a whole word, part of a word, punctuation, or another text fragment.

For example, the sentence:

> Local models can run on my PC.

is broken into tokens by the model's tokenizer. The exact token count depends on that tokenizer.

A token is not necessarily one word or one character.

### What is a context window?

The context window is the amount of tokenized information the model can handle in its active interaction.

It can include:

- Your current prompt.
- Earlier messages in the conversation.
- Retrieved documents or other supplied context.
- The model's generated response.

If the context limit is 8,192 tokens, the prompt and generated content must fit within the applicable context budget.

Important: A larger context window isn't free. Longer contexts generally require more memory, particularly for the KV cache.

## 3. Parameters: what they are and why they matter

Parameters are the learned numerical values inside a neural network. The model adjusts these values during training so it can learn patterns from data.

The most important parameter type we discussed is weights. Neural networks also use other parameters, such as biases.

Think of the weights as learned numerical settings that influence how the model processes information and produces its next-token prediction.

### What does 7B mean?

- `B` means billion.
- A `7B` model has approximately 7 billion parameters.
- An `8B` model has approximately 8 billion parameters.

The parameter count is not the number of words the model knows, nor is it the context window size.

A larger parameter count usually means more memory is needed to store the weights at the same numerical precision. However, parameter count alone does not determine model quality.

## 4. Parameters versus memory

The model's weights must be stored in memory when the model runs.

The basic estimate is:

\\[ \text{Weight memory} \approx \text{Parameter count} \times \text{Bytes per parameter} \\]

Different numerical formats use different amounts of memory per parameter.

| Numerical format | Bits per parameter | Bytes per parameter | Approx. weight memory for 7B |
| ---------------- | ------------------ | ------------------- | ---------------------------- |
| FP32             | 32 bits            | 4 bytes             | 28 GB                        |
| FP16             | 16 bits            | 2 bytes             | 14 GB                        |
| BF16             | 16 bits            | 2 bytes             | 14 GB                        |
| INT8             | 8 bits             | 1 byte              | 7 GB                         |
| 4-bit            | 4 bits             | 0.5 byte            | 3.5 GB                       |

These are rough, decimal-GB estimates for the weights alone. Real model files can differ due to metadata, scales, and quantization details.

### Why this matters

A 7B model in FP32 needs about 28 GB for its weights alone, while a nominal 4-bit version needs roughly 3.5 GB.

That difference is why quantization is so useful for local inference.

Remember: These figures are not the total RAM or VRAM required to run the model. You also need memory for the KV cache, runtime, and other working buffers.

## 5. RAM versus VRAM

These are two different types of memory used by a computer.

- RAM (Random Access Memory): System memory used by your CPU and operating system.
- VRAM (Video Random Access Memory): Memory on a dedicated GPU, used for graphics and GPU workloads, including LLM inference.

A local LLM can run using system RAM, GPU VRAM, or a combination of both if the runtime supports splitting the model across CPU and GPU.

For example, if a model is too large to fit entirely in your GPU's VRAM, a compatible runtime might keep some layers on the CPU and others on the GPU. This can allow the model to run, but it may be slower.

The handbook also recommends keeping a practical margin of roughly 10–20% of memory free beyond the weights and KV cache. This is a rule of thumb, not a guarantee.

## 6. The KV cache

KV stands for Key-Value.

During text generation, the transformer model calculates key and value representations for tokens. Rather than repeatedly calculating all of the earlier tokens' key-value information, inference software can store it in a cache.

This stored information is called the KV cache.

### Why does the KV cache matter?

Its memory usage generally increases with:

- The number of tokens in the active context.
- The number of concurrent sequences or users.
- The model's architecture, including its layers and number of KV heads.
- The numerical precision used to store the cache.

That means a model that runs comfortably with a short prompt may need considerably more memory when processing a long conversation.

### The handbook's KV-cache formula

For a conventional cache with keys and values stored at the same precision:

\\[ \begin{aligned} \text{KV bytes} \approx {}& 2 \times L \times H\_{KV} \times D\\\ &\times T \times B \times N \end{aligned} \\]

Where:

- \\(L\\) = number of transformer layers.
- \\(H\_{KV}\\) = number of KV heads.
- \\(D\\) = dimension of each head.
- \\(T\\) = context length in tokens.
- \\(B\\) = bytes per cached value.
- \\(N\\) = number of active sequences.

The first factor of 2 accounts for the keys and values. To convert the result from bytes to decimal GB, divide by \\(10^9\\).

### What is GQA?

Grouped-Query Attention (GQA) is an attention design in which multiple query heads share a smaller number of key-value heads.

Because the cache depends on the number of KV heads, GQA can substantially reduce KV-cache memory compared with architectures that use a separate KV head for every query head.

Key takeaway: The weights usually stay the same size as your conversation grows. The KV cache grows with the active context and number of sequences.

## 7. Precision: FP32, FP16, BF16 and INT8

Precision describes how numbers are represented and stored.

| Format             | What it means                                          | Bits per value |
| ------------------ | ------------------------------------------------------ | -------------- |
| FP32               | 32-bit floating point                                  | 32             |
| FP16               | 16-bit floating point                                  | 16             |
| BF16               | Brain floating point, 16-bit                           | 16             |
| INT8               | 8-bit integer                                          | 8              |
| 4-bit quantization | Compact representation using roughly 4 bits per weight | \~4            |

FP16 and BF16 both use 16 bits, but their floating-point formats differ. They have different ranges and precision characteristics.

The important point for memory calculations is that fewer bits generally mean fewer bytes required to store each weight.

## 8. Quantization: why and how it works

Quantization reduces the number of bits used to represent model weights.

Imagine an original model weight is stored as a floating-point value such as:

\\[ w = 0.1372 \\]

A 4-bit representation cannot preserve every possible floating-point value exactly. Instead, quantization maps the original value to one of a limited set of representable levels.

After quantization, the reconstructed value might be slightly different from the original.

\\[ w' \approx w \\]

The difference between \\(w\\) and \\(w'\\) is called quantization error.

### Benefits

- Reduces model memory requirements.
- Allows larger models to fit on smaller machines.
- Can improve generation speed when memory bandwidth is a bottleneck.

### Trade-offs

- The reconstructed weights are not identical to the original weights.
- Output quality may decline.
- Lower bit counts can increase the risk of noticeable degradation.

A well-designed 4-bit model can still work very well for chat, summarization and retrieval-based question answering, but this is not guaranteed for every model or task.

## 9. How symmetric 4-bit quantization works

We discussed a simple symmetric quantization example from the handbook.

### Step 1: Determine the number of levels

With \\(b\\) bits, the number of possible binary codes is:

\\[ 2^b \\]

For 4 bits:

\\[ 2^4 = 16 \\]

In the handbook's symmetric signed example, the usable integer levels range from \\(-7\\) to \\(+7\\), giving 15 levels. One possible code is left unused in this scheme.

The maximum positive level is:

\\[ q\_{\max}=2^{b-1}-1 \\]

For \\(b=4\\):

\\[ q\_{\max}=2^3-1=7 \\]

### Step 2: Calculate the scale

The scale maps the original floating-point weights to the available integer levels.

\\[ s=\frac{\max |w|}{q\_{\max}} \\]

Here, \\(\max |w|\\) is the largest absolute weight in the tensor or block being quantized.

### Step 3: Quantize the weights

Divide each weight by the scale and round to the nearest integer:

\\[ q=\operatorname{round}\left(\frac{w}{s}\right) \\]

The handbook's implementation also clips the result to the allowed range.

### Step 4: Reconstruct the approximate weight

Multiply the quantized integer by the scale:

\\[ w'=q\\,s \\]

The reconstructed value may not exactly equal the original weight. This is the basic source of quantization error.

These formulas describe the handbook's illustrative symmetric method; actual production quantization schemes can be more sophisticated.

## 10. Figure 2.4: quantization levels and RMSE

We examined two parts of Figure 2.4.

### Figure 2.4(a): weights snapping to levels

The idea is that original weights can take many floating-point values, but a quantized model has fewer representable levels.

- The histogram represents the distribution of weights.
- The dotted vertical lines represent available quantization levels.
- Original weights are mapped to nearby levels.

This introduces some reconstruction error because the quantized values cannot exactly represent every original value.

### Figure 2.4(b): bits per weight versus RMSE

BPW means bits per weight. RMSE means Root Mean Square Error.

RMSE measures the average magnitude of reconstruction errors, giving larger errors more influence because the differences are squared before averaging.

\\[ \operatorname{RMSE} =\sqrt{\frac{1}{n}\sum\_{i=1}^{n}(w_i-w'\_i)^2} \\]

In general, lower bits per weight tend to increase RMSE because there are fewer representable levels. However, the result also depends on the quantization method and the distribution of weights.

### The handbook's experiment

| Bits  | RMSE, one scale per tensor | RMSE, one scale per 32-weight block |
| ----- | -------------------------- | ----------------------------------- |
| 8-bit | 0.00047                    | 0.00011                             |
| 6-bit | 0.00192                    | 0.00044                             |
| 4-bit | 0.00853                    | 0.00197                             |
| 3-bit | 0.01740                    | 0.00458                             |

For the handbook's 4-bit example, blockwise scaling reduces RMSE from 0.00853 to 0.00197—about a 77% reduction in reconstruction error.

These values come from the handbook's illustrative experiment, not a universal benchmark for all LLMs.

### Why do blocks help?

Imagine most weights in a tensor are small, but a few are unusually large. If one scale is used for the entire tensor, those large outliers determine the scale. Many smaller weights then have fewer useful quantization levels available to represent them.

With a separate scale for every block of 32 weights, an outlier in one block has less influence on the quantization of weights in other blocks.

Important distinction: Lower RMSE means the reconstructed weights are numerically closer to the original weights. It does not directly measure how intelligent the model is or guarantee better answers.

## 11. PTQ versus QAT

These are two approaches to quantization.

### PTQ — Post-Training Quantization

The model is trained first and quantized afterward. The original training process does not need to be repeated.

This is the common approach for ready-made quantized models that people download for local inference.

### QAT — Quantization-Aware Training

Quantization effects are taken into account during training so the model can learn to tolerate them better.

It requires a training or fine-tuning process designed for this purpose.

For someone getting started with local LLMs, PTQ is the more relevant concept because you will generally be downloading an already-trained model in a quantized format.

## 12. GGUF, GPTQ and AWQ

We learned that these three names do not mean exactly the same thing.

| Name | What it is                                                  | Typical use                               |
| ---- | ----------------------------------------------------------- | ----------------------------------------- |
| GGUF | A model file format associated with the llama.cpp ecosystem | Local inference on laptops and desktops   |
| GPTQ | A post-training quantization method                         | Commonly used for quantized GPU inference |
| AWQ  | Activation-aware Weight Quantization                        | Commonly used for quantized GPU inference |

### GGUF

GGUF is a single-file format that can store model weights, tokenizer information and metadata.

It is commonly used with:

- Ollama
- llama.cpp
- LM Studio

GGUF supports multiple quantization variants, including Q4_K_M, Q5_K_M and Q8_0.

### GPTQ

GPTQ is a post-training method that quantizes model layers using calibration data and approximations designed to minimize the impact of quantization on layer outputs.

It is commonly encountered in GPU-oriented model deployments.

### AWQ

AWQ stands for Activation-aware Weight Quantization. It uses activation information to help identify which weights are particularly important to preserve, then applies scaling and quantization accordingly.

It is also commonly used for GPU inference.

Memory trick: GGUF is primarily the packaging/file format; GPTQ and AWQ are quantization methods.

The exact runtime and hardware support must still be checked for the particular model.

## 13. What do Q4, Q5 and Q8 mean?

These labels describe quantization variants and their approximate bits per weight.

| Variant | General trade-off                                                   |
| ------- | ------------------------------------------------------------------- |
| Q4      | Lower memory use; quality depends on the quantization scheme        |
| Q5      | More memory than Q4; can preserve more information                  |
| Q8      | More memory than Q4 or Q5; generally preserves weights more closely |

For a 7B model, nominal weight-only estimates are:

| Precision | Approximate memory |
| --------- | ------------------ |
| 4-bit     | 3.5 GB             |
| 5-bit     | 4.375 GB           |
| 8-bit     | 7 GB               |

These estimates are simplified. Real quantized file sizes vary because of scales, metadata and the particular format.

### Breaking down `Q4_K_M`

Suppose you see this filename:

`Model-7B-Q4_K_M.gguf`

- `7B`: approximately 7 billion parameters.
- `Q4`: a 4-bit quantization family.
- `K`: the K-quant family of blockwise quantization schemes.
- `M`: the medium variant in this naming convention.
- `.gguf`: the file format.

The `K` and `M` suffixes matter because two models both described as 4-bit may use different quantization choices and have different file sizes and quality.

The handbook's practical starting point for a laptop or desktop is GGUF with Q4_K_M, moving up to Q5_K_M or Q8_0 if testing reveals a quality gap and you have enough memory.

## 14. Training versus inference

We also distinguished these two terms:

- Training: The model learns by adjusting its parameters using data and an optimization process.
- Inference: The trained model is used to generate predictions, such as the next token in a response.

When you download a ready-made model and run it through Ollama, you are normally performing inference, not training the model from scratch.

Quantizing a finished model is also not the same as training it. PTQ compresses the trained model after training.

## 15. Prefill versus decode

These are two important stages of text generation.

### Prefill — process the prompt

The model processes the input prompt and builds the KV cache for its tokens.

Main performance measure: time to first token (TTFT)—how long it takes before the first generated token appears.

The handbook describes prefill as generally compute-bound.

### Decode — generate the response

The model repeatedly generates the next token, using the earlier context and the KV cache.

Main performance measure: generation speed, usually measured in tokens per second.

The handbook describes decode as generally memory-bandwidth-bound.

### Why does memory bandwidth affect generation speed?

During decode, the processor must repeatedly access the model's active weights. A simplified upper-bound intuition is:

\\[ \text{Tokens/sec} \lesssim \frac{\text{Memory bandwidth}}{\text{Model size in memory}} \\]

For example, the handbook uses an illustrative 4 GB model and compares systems with 50 GB/s and 400 GB/s memory bandwidth. The simplified ceilings are about 12.5 and 100 tokens per second respectively. Real speeds are lower, and other bottlenecks can matter.

This helps explain why quantization can improve generation speed: fewer bytes may need to be read for the weights during each decode step.

## 16. How to estimate if a model fits on your PC

We can now combine the concepts.

A practical memory estimate is:

\\[ \begin{aligned} \text{Total memory} \approx {}& \text{Weight memory}\\\ &+\text{KV cache}\\\ &+\text{Runtime overhead} \end{aligned} \\]

To make a realistic estimate:

1. Find the model's parameter count.
2. Estimate weight memory from its precision or check the actual model file size.
3. Estimate the KV cache using the architecture, context length, cache precision and number of concurrent sequences.
4. Account for runtime and temporary buffers.
5. Check whether the required memory fits into your available RAM or VRAM, allowing a margin.
6. Test the exact model on your own prompts and workload.

The handbook includes a hands-on exercise for comparing approximately 3B, 8B and 13–14B models on a 16 GB machine, using their actual layer counts, KV heads and head dimensions.

A key lesson from that exercise is that a model fitting into memory is not the same as it being fast enough for your use case.

## 17. Common mistakes we learned to avoid

- Estimating memory from the weights alone and forgetting the KV cache.
- Treating long context windows as if they use no additional memory.
- Forgetting that concurrent users need additional cache memory.
- Assuming every token is one word.
- Using the wrong tokenizer to estimate token counts.
- Assuming all models labelled 4-bit have identical size, speed or quality.
- Assuming that a model that fits will necessarily generate quickly.
- Treating quantization RMSE as a direct measurement of model intelligence.
- Choosing a quantized model by filename alone without testing it on the intended task.

## 18. Quick-reference cheat sheet

| Term           | Remember this                                                     |
| -------------- | ----------------------------------------------------------------- |
| LLM            | Large Language Model                                              |
| Token          | A unit of text processed by a model                               |
| Context window | Maximum active token context                                      |
| Parameter      | A learned numerical value in the model                            |
| 7B             | Approximately 7 billion parameters                                |
| Precision      | How numerical values are represented                              |
| Quantization   | Representing weights using fewer bits                             |
| RMSE           | Measures reconstruction error                                     |
| BPW            | Bits per weight                                                   |
| PTQ            | Quantization after training                                       |
| QAT            | Training with quantization effects taken into account             |
| KV cache       | Stored keys and values for active context                         |
| GQA            | Grouped-Query Attention; reduces KV heads relative to query heads |
| RAM            | System memory                                                     |
| VRAM           | GPU memory                                                        |
| GGUF           | Model file format                                                 |
| GPTQ           | Post-training quantization method                                 |
| AWQ            | Activation-aware Weight Quantization                              |
| Q4_K_M         | Common 4-bit K-quant medium variant                               |
| Prefill        | Processing the prompt                                             |
| Decode         | Generating tokens                                                 |
| TTFT           | Time to first token                                               |
| Tokens/sec     | Generation throughput                                             |

## 19. What you should be able to explain now

Use these as self-check questions:

### Test your understanding

7 of 7 answers revealed

1\. Why does a 7B model need less weight memory in 4-bit than in FP16?

Because a nominal 4-bit representation uses about 0.5 byte per weight instead of 2 bytes, reducing weight memory from roughly 14 GB to 3.5 GB.

Hide answer

2\. Why can a model need more memory when the conversation gets longer?

Because the KV cache stores key-value information for active context, and its size generally grows with context length.

Hide answer

3\. Why do blockwise scales help quantization?

A large outlier affects the scale of only its own block rather than forcing one scale across the entire tensor, helping represent smaller weights more accurately.

Hide answer

4\. What is the difference between GGUF and GPTQ?

GGUF is a model file format; GPTQ is a quantization method.

Hide answer

5\. What does Q4_K_M tell you?

It identifies a 4-bit K-quant medium variant, commonly used as a balance of size and quality.

Hide answer

6\. What is the difference between prefill and decode?

Prefill processes the prompt and builds the cache; decode generates tokens one by one.

Hide answer

7\. Why isn't weight memory the whole memory requirement?

Inference also requires the KV cache, runtime overhead and temporary working buffers.

Hide answer

Hide all answers

The big picture: We learned how model size translates into memory, how precision and quantization reduce that memory, why context length creates additional KV-cache costs, how to interpret quantized model names, and why both memory capacity and memory bandwidth matter when running an LLM locally.