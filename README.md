# QML Engine

A stateful inference engine for large language models, built for long-running, multi-user conversations on a single
server.

## Status

The source code and documentation will be published in this repository. Until then, this page only announces the
project and its current measurements.

- Engine: C++17 and CUDA, OpenAI-compatible HTTP API
- Targets: NVIDIA data-center GPUs, on-premises deployment
- License: to be announced with the source release

## Performance (H200, 2026-10-27)

One NVIDIA H200, one request at a time, BF16 weights, streamed through the OpenAI chat API. Time to the first token is the
median of three new prompts per size (nothing reused between requests); decode is the time per token over 256 generated
tokens. Same models and settings for every engine: llama.cpp `llama-server` (commit 7049ff0, flash attention on), vLLM
0.32.0, SGLang 0.5.22. qml-engine runs on its own kernels (no ggml graph) with its store on a local disk.

**Llama-3.1-8B-Instruct, BF16** (ms)

| Prompt (tokens) | qml-engine | llama.cpp | vLLM | SGLang |
|---|---|---|---|---|
| 133 | 15.1 | 25.0 | 24.3 | 14.1 |
| 598 | 26.6 | 77.9 | 40.4 | 25.5 |
| 2,340 | 80.5 | 251.9 | 78.1 | 73.3 |
| 8,892 | 308.3 | 992.7 | 291.4 | 313.7 |
| 26,926 | 1,164.5 | 3,504.0 | 976.2 | 937.0 |
| Decode (ms per token) | 5.62 | 5.10 | 4.97 | 4.99 |

**Qwen3-30B-A3B, BF16, mixture of experts** (ms)

| Prompt (tokens) | qml-engine | llama.cpp | vLLM | SGLang |
|---|---|---|---|---|
| about 130 | 21.2 | 66.2 | 25.7 | 23.1 |
| about 590 | 51.3 | 204.6 | 61.3 | 28.1 |
| about 2,340 | 120.9 | 612.1 | 83.3 | 61.5 |
| about 8,890 | 399.7 | 2,294.7 | 267.2 | 206.3 |
| about 26,900 | 1,379.8 | 7,252.8 | 943.3 | 889.8 |
| Decode (ms per token) | 4.80 | 5.06 | 3.97 | 3.59 |

Prompts of about 130 tokens, the common case in a conversation that keeps its KV, are as fast as vLLM and SGLang;
longer prompts and decode are 1.1 to 2.0 times slower than the faster of the two and are being worked on. These numbers are for a fresh request. A turn of a
session restores the conversation's KV from the engine's store, so only the new turn is prefilled.

## Models

| Model | Weights | GPUs | Path |
|---|---|---|---|
| Qwen3-30B-A3B | Q3_K_M, BF16 | H200, B200 | engine's own kernels |
| Qwen3-32B | GGUF quantized | H200 | engine's own kernels |
| Llama-3.1-8B-Instruct | BF16 | H200 | engine's own kernels |
| Llama-3.2-3B-Instruct | GGUF | H200 | engine's own kernels (long attention not yet on cuDNN) |
| gpt-oss-20b | MXFP4 | H200 | engine's own kernels (attention sinks, sliding window) |
| Qwen3-Next, DeepSeek (MLA), Gemma, vision models | — | — | ggml backend for now |

Own kernel sets per GPU generation: sm_90 (Hopper: WGMMA, TMA) and sm_100 (Blackwell: tcgen05); sm_89 (Ada) runs the
decode kernels. Prefill attention uses cuDNN.

## Contact

KPST (Korea Platform Service Technology Co., Inc.)
