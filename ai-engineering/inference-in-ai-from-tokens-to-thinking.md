---
publish_desk_active: true
author: "ilak"
status: "published"
createdAt: "2026-05-17"
updatedAt: "2026-05-17"
publish_desk:
  title: "Inference in AI: From Tokens to Thinking"
  slug: inference-in-ai-from-tokens-to-thinking
  excerpt: "The definitive visual guide to AI inference — from first principles through token generation, KV cache, GPU serving, optimization, reasoning-time compute, and the economics of production AI."
  category: ai-engineering
  tags:
    - inference
    - llm serving
    - gpu
    - kv cache
    - transformers
    - vllm
    - quantization
    - agents
    - systems engineering
    - production ai
  premium: false
  magazines:
    - ai-engineering
---

# Inference in AI: From Tokens to Thinking

> The runtime of intelligence — how frozen models turn prompts into predictions, how GPUs serve billions of tokens, and why inference is becoming the defining systems problem of the AI era.

An interactive visual guide for beginners through advanced AI engineers. **Companion presentation:** `/presentations/inference-in-ai` in the Publish Desk app.

---

## 1. The Big Idea

Training teaches a model; **inference uses it**. Every chat, agent, copilot, and multimodal app is inference — and its cost, latency, and quality now dominate product economics.

```
┌─────────────────────────┐         ┌─────────────────────────┐
│        TRAINING         │         │        INFERENCE        │
│  Learn weights from data│  ───▶   │  Run frozen weights     │
│  Batch-heavy · offline  │ weights │  Request-driven · online│
│  Backward pass + optim  │         │  Forward pass only      │
│  "Factory: build brain" │         │  "Live brain: think now"│
└─────────────────────────┘         └─────────────────────────┘
```

- **Inference** is forward-only execution of learned weights on new inputs.
- Unlike training, there is **no gradient** — only prediction under latency and memory constraints.
- Modern products ship intelligence as a service: inference is the **runtime**, not an afterthought.
- GPU-hours at inference scale often **exceed one-time training spend** for popular models.

> **Systems insight:** Think of training as building a highway network and inference as rush-hour traffic management. The architecture is fixed; success is scheduling, capacity, and tail latency.

---

## 2. How LLM Inference Actually Works

A large language model does not think in paragraphs. It predicts **one token at a time**, conditioned on everything before it — prompt plus all generated tokens so far.

```
Prompt → Tokenize → Embed → Transformer → Logits → Sample → Token
  │                                                              │
  └──────────────── autoregressive loop ──────────────────────────┘
```

**Step by step:**

1. Your prompt is **tokenized** into a sequence of subword IDs.
2. **Embeddings** map tokens to vectors; positional encoding (often **RoPE**) encodes order.
3. Each transformer layer applies **causal self-attention** — tokens may only attend to the past.
4. The final layer emits **logits** (scores over the vocabulary); **sampling** picks the next token.
5. That token is **appended**; the loop repeats until stop criteria or max length.

**Streaming** exposes tokens as they are generated — the UX of "the model is typing" is literally serial decode steps completing on the server.

---

## 3. The Transformer During Inference

The same architecture that learned from the web must run thousands of times per second per GPU — with a **growing memory footprint** as context lengthens.

```
┌──────────────────────────────────────┐
│           Output logits              │
├──────────────────────────────────────┤
│     Layer N: Self-Attention + FFN      │
├──────────────────────────────────────┤
│              ...                     │
├──────────────────────────────────────┤
│     Embeddings + RoPE positions      │
└──────────────────────────────────────┘
```

- **Causal masking** ensures autoregressive correctness: no peeking at future tokens.
- **KV cache** stores key/value projections for prior tokens so attention does not recompute history.
- Feed-forward blocks and residual paths dominate FLOPs; **LayerNorm** stabilizes activations.
- At decode time, batch size is often small — **memory bandwidth**, not raw FLOPs, becomes the bottleneck.

### Why KV cache is critical

Without caching, each new token would re-run attention over the **entire** prefix — O(n²) work per step. The KV cache holds past keys and values in GPU HBM so each decode step only computes attention for the **new** query token against cached history.

### Compute vs memory

| Phase   | Typical bottleneck      | Why                          |
|---------|-------------------------|------------------------------|
| Prefill | Compute (tensor cores)  | Parallel over prompt tokens  |
| Decode  | Memory bandwidth (HBM)  | Small matmuls, large KV read |

---

## 4. Token Generation & Decoding

Logits are a probability landscape over ~100k tokens. **Decoding strategies** shape creativity, factuality, and cost.

```
                    logits
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
      "the"          "a"           "in"
     p=0.31         p=0.22         p=0.18
```

| Strategy              | Behavior                                      |
|-----------------------|-----------------------------------------------|
| Greedy                | Argmax — fast, repetitive                     |
| Temperature           | Scales logits before softmax                  |
| Top-k / Top-p         | Truncate tail of distribution                 |
| Beam search           | Keep multiple hypotheses (translation-era)   |
| Speculative decoding  | Draft with small model, verify with large one |
| Constrained decoding  | Grammar / JSON / tool schemas                   |

- **Temperature:** low → deterministic; high → creative (higher entropy).
- **Top-p (nucleus):** sample from smallest set whose cumulative probability ≥ p.
- **Speculative decoding:** when acceptance rate is high, yields "free" tokens/sec.

---

## 5. Inference Performance

Users feel **time-to-first-token (TTFT)**; operators optimize **tokens/sec** and **GPU utilization**. Prefill and decode are different phases with different bottlenecks.

```
PREFILL (parallel)          DECODE (serial)
[████████████████]    →     [█][█][█][█][█]
 process full prompt          one token per step
 sets TTFT                    memory-bound
```

- **Prefill** processes the prompt in parallel — compute-heavy, sets TTFT.
- **Decode** generates one or few tokens per step — memory-bandwidth-heavy.
- **Continuous batching** merges new requests without waiting for entire batches to finish.
- **Tail latency** spikes when queues saturate or KV memory fragments.

**Production metrics:** TTFT, inter-token latency, tokens/sec/GPU, queue depth, KV cache utilization, batch size distribution.

---

## 6. GPU & Hardware Inference

NVIDIA H100s, Google TPUs, Apple Neural Engine, and custom ASICs chase more useful **tokens per watt per dollar**.

```
                    ┌─────────────────┐
                    │ Inference gateway│
                    └────────┬────────┘
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
        ┌──────┐          ┌──────┐          ┌──────┐
        │ GPU 1│          │ GPU 2│          │ GPU 3│
        │ TP/PP│          │ TP/PP│          │ TP/PP│
        └──────┘          └──────┘          └──────┘
              NCCL · continuous batching · KV-aware scheduling
```

- **Tensor cores** accelerate matrix multiplies in FP16/BF16/INT8.
- **HBM bandwidth** feeds weights and KV cache — often the decode bottleneck.
- **FlashAttention** fuses attention kernels to reduce memory traffic.
- **Tensor / pipeline parallelism** shard large models across devices (NCCL collectives).

| Tool            | Role                              |
|-----------------|-----------------------------------|
| CUDA / cuDNN    | Kernel execution on NVIDIA GPUs   |
| FlashAttention  | IO-aware attention fusion         |
| CUDA Graphs     | Reduce CPU launch overhead        |
| TensorRT-LLM    | Compiled inference graphs         |

---

## 7. Modern Inference Optimization

Serving stacks squeeze latency and cost through **precision**, **memory layout**, and **scheduling** — not bigger models alone.

**Quantization ladder:**

```
FP32 ████████████████████████████████
FP16 ████████████████████
INT8 ████████████
INT4 ██████
```

- **Quantization (INT8/INT4)** shrinks weights and activations with calibrated tradeoffs.
- **PagedAttention (vLLM)** stores KV in non-contiguous blocks — higher utilization, less fragmentation.
- **Distillation / pruning** compress models for edge; **MoE** routes tokens to expert subsets only.
- **llama.cpp**, **TensorRT-LLM**, **SGLang** target deployment with fusion and custom kernels.

| Stack      | Contribution                          |
|------------|---------------------------------------|
| vLLM       | PagedAttention + continuous batching  |
| SGLang     | RadixAttention, structured generation |
| DeepSpeed  | Multi-GPU inference                   |
| Triton     | Model serving at scale                |

---

## 8. Distributed & Cloud Inference

Production inference is **fleets of GPUs** behind gateways, autoscalers, and regional routing — not a single process on one card.

- Load balancers route by model, SLA tier, or geography.
- **Disaggregated inference** separates prefill and decode onto specialized pools.
- Serverless endpoints scale to zero; dedicated clusters optimize **$/token**.
- **Ray Serve** and **Kubernetes** orchestrate replicas, health checks, and rollouts.
- **AI gateways** handle auth, rate limits, model routing, and observability.

**Advanced:** multi-node decoding, inference orchestration layers, scheduler-aware serving, KV cache compression across replicas.

---

## 9. Reasoning-Time Compute

Inference is no longer one forward pass. Agents chain tool calls, tree search, and reflection — **multiplying tokens by design** for higher-quality answers.

```
                      query
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
        path A        path B        path C
           │            │            │
           └────────────┼────────────┘
                        ▼
                  best answer
            (self-consistency / vote)
```

- **Chain-of-thought** elicits intermediate steps before the final answer.
- **Test-time scaling** runs multiple samples or search branches, then aggregates.
- **Tool calling** extends inference with retrieval, code execution, and APIs.
- **Planning loops** treat the LLM as a controller in a larger computation graph.

> Inference is becoming **dynamic computation** — search, planning, and verification — not static token streaming.

---

## 10. Multimodal Inference

Vision, audio, and video encoders produce token-like embeddings fused with text in shared transformer stacks.

```
  Text ──┐
  Image ─┼──▶ Fusion transformer ──▶ unified decode
  Audio ─┘
```

- Images are **patch-embedded** and projected into the language model embedding space.
- **Cross-attention** layers align modalities when encoders stay separate.
- Speech pipelines chunk audio into **streaming inference** windows.
- Video stacks sample frames or use temporal encoders — throughput vs fidelity tradeoffs.

---

## 11. Edge & On-Device Inference

SLMs, aggressive quantization, and Apple Silicon NPUs bring inference **local** — for privacy, offline use, and sub-100ms interaction loops.

| Edge / on-device        | Cloud                           |
|-------------------------|---------------------------------|
| Low latency             | Largest models                  |
| Privacy                 | Elastic scale                   |
| Offline                 | Shared GPU pools                |
| Quantized SLMs          | Agent orchestration at scale    |

- **GGUF**, **MLX**, and **ONNX** runtimes target laptop and mobile constraints.
- **TinyML** covers microcontrollers; cloud handles frontier-scale reasoning.
- **Hybrid** architectures route sensitive tasks on-device, hard problems to cloud.

---

## 12. The Economics of Inference

**Cost per token** ≈ GPU hourly cost ÷ (useful tokens/sec × utilization). Inference margin shapes every AI product.

```
GPU $/hr → Utilization → Tokens/sec → Cost/token → Margin
```

- Underutilized GPUs destroy margin — **scheduling and batching are business-critical**.
- Premium APIs charge per million tokens; **caching and distillation** defend margin.
- Inference infrastructure is the **new cloud layer**: gateways, observability, finops for AI.

Why inference dominates economics: every user message triggers compute; usage scales with MAU × messages × tokens × model size — while training is largely amortized.

---

## 13. The Future of Inference

Sparse models, persistent agent memory, and inference-native OS schedulers will treat **thinking as a first-class resource** — like CPU cycles once were.

- **Adaptive compute** allocates more steps to hard queries, fewer to trivial ones.
- **Memory-augmented inference** externalizes long context into retrieval stores.
- **Neuromorphic** accelerators explore radically different energy profiles.
- **Industrial copilots** embed inference in closed-loop physical systems.
- **World models** and **persistent agents** multiply inference into always-on substrates.

Horizons: real-time reasoning substrates, inference-native operating systems, autonomous agent networks, scientific reasoning architectures.

---

## 14. The Inference Stack (Synthesis)

From your first keystroke to a reasoning agent orchestrating tools across a GPU cluster — inference is where AI meets the world.

```
┌─────────────────────────────────────┐
│           User prompt               │
├─────────────────────────────────────┤
│    Tokenization & embeddings        │
├─────────────────────────────────────┤
│   Transformer + KV cache (decode)   │
├─────────────────────────────────────┤
│      Decoding & sampling            │
├─────────────────────────────────────┤
│     GPU / accelerator execution     │
├─────────────────────────────────────┤
│   Cluster, gateway, orchestration   │
├─────────────────────────────────────┤
│  Agents, tools, reasoning loops       │
└─────────────────────────────────────┘
```

**Training built the model. Inference delivers intelligence** — one token, one request, one decision at a time.

---

## Further reading (engineering stack)

- **vLLM** — PagedAttention, continuous batching
- **TensorRT-LLM** — NVIDIA compiled serving
- **llama.cpp** — CPU/GGUF edge inference
- **SGLang** — structured generation, RadixAttention
- **FlashAttention** — fused attention kernels
- **Ray Serve / Triton** — production orchestration
