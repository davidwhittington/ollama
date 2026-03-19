# Ollama Model Reference

A curated reference for models available through Ollama — what they're for, how big they are, and what you need to run them. Covers the models installed on this stack plus a broader catalog of notable options.

---

## Models Installed on This Stack

| Model | Size | Type | Notes |
|-------|------|------|-------|
| `llama3.2:latest` | 2.0 GB | Chat | Meta's compact baseline — fast, general-purpose |
| `llama3.1:70b` | ~40 GB | Chat | Full-scale 70B — needs 48GB+ RAM or GPU VRAM |
| `codellama:7b` | 3.8 GB | Code | Code completion and explanation, any language |
| `mistral:7b` | 4.4 GB | Chat | Fast and capable — strong all-rounder at 7B scale |
| `deepseek-coder:6.7b` | 3.8 GB | Code | Chinese open-source code model — fully local, no callbacks |
| `nomic-embed-text` | 274 MB | Embedding | Text embeddings only — powers the RAG pipeline |

### Memory Requirements

The rule of thumb for Q4-quantized models:

| Model Size | RAM Needed | Notes |
|-----------|-----------|-------|
| 3B | ~4 GB | Runs on almost anything |
| 7B | ~6–8 GB | Comfortable on 8GB+ |
| 13B | ~10–12 GB | 16GB recommended |
| 34B | ~24–28 GB | 32GB+ needed |
| 70B | ~38–42 GB | 48GB+ recommended |

`llama3.1:70b` runs memory-mapped to disk without enough RAM — technically functional but very slow (~1–2 tok/s on CPU swap). With 48GB RAM and the Vega 56 GPU via Vulkan, it should load fully and run at usable speeds.

---

## Ollama Model Catalog — Notable Models

### Chat and Instruction

| Model | Params | Size | Best For |
|-------|--------|------|---------|
| `llama3.2` | 3B | 2.0 GB | Fast general chat, edge/embedded use |
| `llama3.2:1b` | 1B | 1.3 GB | Minimum viable — very fast, basic quality |
| `llama3.1` | 8B | 4.7 GB | Solid general-purpose, strong instruction following |
| `llama3.1:70b` | 70B | 40 GB | Highest quality open model at this class |
| `llama3.1:405b` | 405B | ~230 GB | Research/enterprise — impractical without serious hardware |
| `mistral` | 7B | 4.1 GB | Punches above its weight, fast, Apache 2.0 licensed |
| `mistral-nemo` | 12B | 7.1 GB | Mistral + Nvidia collaboration, strong context handling |
| `mixtral` | 8x7B MoE | 26 GB | Mixture-of-experts — high quality, but memory-hungry |
| `gemma2` | 9B | 5.4 GB | Google's open model — competitive at 9B |
| `gemma2:2b` | 2B | 1.6 GB | Smallest Gemma, good for lightweight tasks |
| `phi4` | 14B | 8.9 GB | Microsoft's small model with outsized reasoning ability |
| `phi4-mini` | 3.8B | 2.5 GB | Ultra-efficient, good at math and structured tasks |
| `qwen2.5` | 7B | 4.7 GB | Alibaba's model — multilingual, strong coding |
| `qwen2.5:72b` | 72B | 47 GB | Top-tier open model, competitive with GPT-4 class |
| `command-r` | 35B | 20 GB | Cohere's RAG-optimized model — strong retrieval tasks |
| `command-r-plus` | 104B | 59 GB | Cohere's flagship — excellent at tool use and RAG |
| `solar` | 10.7B | 6.1 GB | Upstage's model, strong at knowledge tasks |

### Code Models

| Model | Params | Size | Best For |
|-------|--------|------|---------|
| `codellama` | 7B | 3.8 GB | Meta's code model — multi-language, solid baseline |
| `codellama:13b` | 13B | 7.4 GB | Better quality code, still manageable size |
| `codellama:34b` | 34B | 19 GB | Best CodeLlama tier — approaches GPT-4 on code |
| `deepseek-coder` | 6.7B | 3.8 GB | Strong code model from DeepSeek — fully local |
| `deepseek-coder:33b` | 33B | 19 GB | Competes with GPT-4 on HumanEval |
| `deepseek-coder-v2` | 16B | 8.9 GB | V2 architecture, significant improvement |
| `codegemma` | 7B | 5.0 GB | Google's code model, Apache 2.0 |
| `starcoder2` | 15B | 9.1 GB | BigCode's open model, trained on 600+ languages |
| `qwen2.5-coder` | 7B | 4.7 GB | Strong recent code model from Alibaba |
| `qwen2.5-coder:32b` | 32B | 19 GB | One of the best open-source code models available |

### Reasoning and Math

| Model | Params | Size | Best For |
|-------|--------|------|---------|
| `deepseek-r1` | 7B | 4.7 GB | DeepSeek's reasoning model — chain-of-thought, math |
| `deepseek-r1:14b` | 14B | 9.0 GB | Better reasoning, still runs on 16GB |
| `deepseek-r1:70b` | 70B | 43 GB | Strong reasoning — needs 48GB+ |
| `phi4` | 14B | 8.9 GB | Microsoft's reasoning-focused model |
| `qwq` | 32B | 20 GB | Alibaba QwQ — reasoning model, strong at math |

### Embedding Models

| Model | Size | Notes |
|-------|------|-------|
| `nomic-embed-text` | 274 MB | Fast local embeddings — powers this RAG stack |
| `mxbai-embed-large` | 669 MB | Higher quality embeddings, slower |
| `all-minilm` | 46 MB | Smallest viable option — low quality at extremes |
| `snowflake-arctic-embed` | 137 MB | Strong retrieval performance, compact |
| `nomic-embed-text:v1.5` | 274 MB | Updated nomic, same size, better performance |

### Vision / Multimodal

| Model | Params | Size | Best For |
|-------|--------|------|---------|
| `llava` | 7B | 4.5 GB | Image + text — describe images, answer visual questions |
| `llava:13b` | 13B | 8.0 GB | Better visual quality |
| `llava-phi3` | 3.8B | 2.9 GB | Compact multimodal, good speed |
| `moondream` | 1.8B | 1.7 GB | Lightweight image model — edge use |
| `bakllava` | 7B | 4.5 GB | LLaVA variant with Mistral backbone |

---

## Notes on Specific Models

### `deepseek-coder:6.7b` — Trust and Privacy

DeepSeek is a Chinese AI lab (High-Flyer Capital). The model weights distributed via Ollama are static files — there is no telemetry, no network callback, and no runtime connection to DeepSeek's infrastructure. The model runs entirely offline once pulled. Ollama itself is the only network surface here, and it's bound to localhost/LAN.

The concern some raise about DeepSeek relates to their hosted API product, not the open-weight models. The open weights (DeepSeek-Coder, DeepSeek-R1, etc.) are MIT or Apache 2.0 licensed and are the same static files as any other open model.

Short version: safe to use locally.

### `llama3.1:70b` — What to Expect

On this stack (CPU inference + Vulkan Vega 56):
- With GPU active: 20–40 tok/s depending on context length
- CPU fallback: 1–5 tok/s — technically works, not practical for chat

The 70B model is best used for tasks where quality matters more than speed — long-form writing, complex reasoning, document summarization. For interactive chat, `llama3.1:8b` or `mistral:7b` are better choices.

### `nomic-embed-text` — Embedding vs. Chat

This model is embedding-only. It takes text and returns a vector — it cannot generate responses. It's what powers the RAG pipeline (convert query to vector, find closest chunks in Chroma, inject into context for the chat model).

You cannot use it directly in Open WebUI for chat. That's expected behavior.

---

## Ollama CLI Reference

```bash
# List installed models
ollama list

# Pull a model
ollama pull mistral:7b

# Run a model interactively
ollama run mistral:7b

# Remove a model
ollama rm codellama:7b

# Show model info
ollama show llama3.2

# Check running models
curl http://localhost:11434/api/ps | python3 -m json.tool

# Pull progress (check log)
tail -f /tmp/llama70b-pull.log
```

---

## Changelog

| Date | Entry |
|------|-------|
| 2026-03-18 | Initial model reference — catalog, analysis, RAM guide |
