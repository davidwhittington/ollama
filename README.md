# ollama 🦙

Guides for running local LLM inference and building private RAG pipelines with Ollama. Everything runs on the local network — no cloud APIs, no telemetry, no data leaving the building.

Companion to [`davidwhittington/proxmox`](https://github.com/davidwhittington/proxmox), which covers the VM setup and GPU passthrough side of things.

---

## Guides

### Local LLM + RAG Pipeline on Ubuntu 24.04

**[Read the guide →](https://davidwhittington.github.io/ollama/ollama-rag-setup.html)**

Covers the full stack from bare VM to working RAG:

- Installing Ollama and verifying CPU-only inference
- Exposing the API to the local network via systemd override
- Docker setup, Chroma vector database deployment
- Open WebUI with correct `host.docker.internal` configuration
- `nomic-embed-text` embedding model and how it fits into the pipeline
- Python ingestion script: walks a directory, extracts and chunks text, batch-embeds via Ollama, upserts into Chroma
- Supported file types, skip lists, orphan cleanup, idempotent re-runs
- CLI flags: `--stats`, `--delete`, `--cleanup`, `--path`

**VM setup:** See [`davidwhittington/proxmox — docs/ubuntu-cloudinit-vm.md`](https://github.com/davidwhittington/proxmox/blob/main/docs/ubuntu-cloudinit-vm.md) for provisioning the VM itself.

Raw Markdown: [`docs/ollama-rag-setup.md`](docs/ollama-rag-setup.md)

---

## Stack

| Component | Detail |
|-----------|--------|
| Ollama | 0.18.x, systemd service |
| Embedding model | nomic-embed-text |
| Vector DB | Chroma (v2 API) |
| Frontend | Open WebUI |
| VM OS | Ubuntu 24.04.4 LTS |
| Inference | CPU Phase 1 · GPU Phase 2 (Vega 56 via TB4) |

## Status

Phase 1 (CPU-only) fully working as of 2026-03-18. Phase 2 GPU passthrough documented in [`davidwhittington/proxmox`](https://github.com/davidwhittington/proxmox).

## Changelog

| Date | Entry |
|------|-------|
| 2026-03-18 | Initial commit: Ollama + RAG pipeline guide |
