# Ollama + Open WebUI + RAG Pipeline on Ubuntu 24.04

## Introduction

This guide covers deploying Ollama on an Ubuntu 24.04 VM, exposing it to the local network, adding Open WebUI as a frontend, standing up Chroma as a vector database, and building a local RAG pipeline that indexes files from a Mac. Every component runs on your hardware. No data leaves the local network.

**Companion doc:** VM provisioning is covered separately in `davidwhittington/proxmox — docs/ubuntu-cloudinit-vm.md`.

**VM specs used in this guide:**
- Ubuntu 24.04.4 LTS, Proxmox VE guest
- 6 vCPU, 16GB RAM, 100GB disk
- SSH alias: `ollama` (configure your own in `~/.ssh/config`)
- Phase 1: CPU-only inference. Phase 2 (future): GPU passthrough via TB4 eGPU.

---

## Part 1 — Install Ollama

```bash
ssh ollama
curl -fsSL https://ollama.com/install.sh | sh
```

The installer:
- Installs binaries to `/usr/local/`
- Creates an `ollama` system user, adds to `render` and `video` groups
- Registers and starts `ollama.service` via systemd
- Prints a warning if no GPU is detected (CPU-only mode is fully functional)

**Verify:**
```bash
ollama --version
sudo systemctl status ollama --no-pager
```

**Pull a model:**
```bash
ollama pull llama3.2
```

**Test inference:**
```bash
curl http://localhost:11434/api/generate \
  -d '{"model":"llama3.2","prompt":"Hello","stream":false}'
```

---

## Part 2 — Expose the API to the Network

By default Ollama binds to `127.0.0.1:11434` (loopback only). To reach it from other machines on the network, override via a systemd drop-in:

```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
cat <<EOF | sudo tee /etc/systemd/system/ollama.service.d/override.conf
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
EOF

sudo systemctl daemon-reload
sudo systemctl restart ollama
```

Verify:
```bash
sudo ss -tlnp | grep 11434
# Expected: LISTEN 0  4096  *:11434  *:*
```

Test from another machine:
```bash
curl http://{VM_IP}:11434/
# Returns: Ollama is running
```

---

## Part 3 — Install Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker {username}
```

Log out and back in (or `newgrp docker`) for group membership to take effect.

---

## Part 4 — Deploy Chroma (Vector Database)

Chroma stores vector embeddings for the RAG pipeline.

```bash
docker run -d \
  --name chroma \
  --restart always \
  -p 8000:8000 \
  -v chroma-data:/chroma/chroma \
  chromadb/chroma
```

**API version note:** Chroma's current release uses a v2 API. All endpoints follow the pattern:
```
/api/v2/tenants/{tenant}/databases/{database}/collections/...
```
Default tenant: `default_tenant`. Default database: `default_database`.

Heartbeat check:
```bash
curl http://{VM_IP}:8000/api/v2/heartbeat
```

---

## Part 5 — Deploy Open WebUI

Open WebUI is a ChatGPT-style frontend that connects to Ollama for inference and Chroma for RAG.

**Critical:** Use `host.docker.internal` (not `127.0.0.1`) for Ollama and Chroma URLs. Inside the container, `127.0.0.1` resolves to the container itself, not the VM host.

```bash
docker run -d \
  --name open-webui \
  --restart always \
  --add-host=host.docker.internal:host-gateway \
  -p 3000:8080 \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  -e VECTOR_DB=chroma \
  -e CHROMA_HTTP_HOST=host.docker.internal \
  -e CHROMA_HTTP_PORT=8000 \
  -e RAG_EMBEDDING_ENGINE=ollama \
  -e RAG_EMBEDDING_MODEL=nomic-embed-text \
  -e RAG_OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  -v open-webui:/app/backend/data \
  ghcr.io/open-webui/open-webui:main
```

Access at: `http://{VM_IP}:3000`

First visit: create a local admin account. No external accounts or cloud services required.

**Verify Ollama connectivity from inside the container:**
```bash
docker exec open-webui curl -s http://host.docker.internal:11434/api/tags
```

**Model selection:** In the Open WebUI chat interface, select `llama3.2` (or any other chat model) from the model dropdown. Do not select `nomic-embed-text` — it is an embedding-only model and cannot respond to chat prompts.

---

## Part 6 — Pull the Embedding Model

RAG requires a separate embedding model. `nomic-embed-text` is recommended: fast, accurate, runs efficiently on CPU.

```bash
ollama pull nomic-embed-text
```

**What embedding models do:** They convert text into high-dimensional vectors that represent semantic meaning. They have no concept of conversation — they only transform text into numbers. `nomic-embed-text` runs silently in the background during RAG queries; you never interact with it directly.

**RAG flow:**
```
Your question
  → nomic-embed-text → query vector
  → Chroma similarity search → relevant file chunks
  → llama3.2 (question + chunks as context) → answer
```

---

## Part 7 — RAG Ingestion Pipeline

The ingestion script runs on the source machine (Mac), walks a directory, extracts and chunks text, embeds via Ollama, and upserts into Chroma.

### Install dependencies

```bash
pip3 install --break-system-packages requests pypdf python-docx
```

### How it works

1. Walks `/Users/{username}` recursively (or a specified subdirectory)
2. Skips non-content directories (package caches, build artifacts, media libraries, IDE caches)
3. Extracts text based on file type
4. Chunks text into ~800 character segments with 150 character overlap
5. Sends all chunks for a file to Ollama in a single batch embed request
6. Upserts into Chroma with metadata: source path, chunk index, modified time, extension
7. On each full run: removes orphaned chunks (files deleted from disk) before indexing

### Supported file types

| Category | Extensions |
|----------|-----------|
| Docs | `.txt`, `.md`, `.rst`, `.org`, `.tex` |
| Code | `.py`, `.js`, `.ts`, `.swift`, `.go`, `.rs`, `.c`, `.cpp`, `.sh`, `.sql`, and more |
| Config | `.json`, `.yaml`, `.toml`, `.ini`, `.csv`, `.xml` |
| Web | `.html`, `.css`, `.scss`, `.svelte`, `.vue` |
| Documents | `.pdf` (pypdf), `.docx` (python-docx) |

### Skipped directories

- Version control: `.git`
- Package caches: `node_modules`, `go/pkg`, `.cargo`, `.gem`, `.gradle`, `.m2`, `.npm`, `.yarn`
- Build output: `dist`, `build`, `DerivedData`, `target`, `.next`
- Apple media/system: `Music/Music/Media.localized`, `Photos Library.photoslibrary`, `Library/Caches`, `Library/Developer`, `Library/Containers`
- IDE: `Xcode`, `iOS DeviceSupport`
- Misc: `__pycache__`, `venv`, `.venv`, `tmp`, `temp`

### Run the index

```bash
# Full index (background, recommended for large home directories)
nohup python3 ~/rag-ingest.py > ~/rag-ingest.log 2>&1 &

# Monitor progress
tail -f ~/rag-ingest.log

# Check chunk count
python3 ~/rag-ingest.py --stats
```

### CLI flags

| Flag | Description |
|------|-------------|
| `--path /some/dir` | Index a specific directory instead of the default |
| `--stats` | Show total chunk count in Chroma |
| `--delete /path/to/file` | Remove a specific file's chunks from the index |
| `--cleanup` | Remove orphaned chunks only, no indexing |

### Idempotency and cleanup

Chunk IDs are derived from a SHA-256 hash of the file path plus the chunk index. Re-running the script overwrites existing chunks — it never creates duplicates.

On every full run, the script first scans all source paths stored in Chroma and removes any whose files no longer exist on disk. This keeps the index accurate without manual intervention.

### Sensitive files

Files can be removed from the index at any time without re-indexing everything:
```bash
python3 ~/rag-ingest.py --delete /Users/{username}/path/to/sensitive/file
```

---

## Part 8 — Architecture Overview

```
Source machine (/Users/{username})
  └── rag-ingest.py
        ├── text extraction + chunking
        ├── → Ollama:{VM_IP}:11434 (nomic-embed-text) → vectors
        └── → Chroma:{VM_IP}:8000 → upsert chunks + vectors

Browser
  └── Open WebUI → http://{VM_IP}:3000
        ├── → Ollama (host.docker.internal:11434) — chat inference
        ├── → Ollama (host.docker.internal:11434) — query embedding
        └── → Chroma (host.docker.internal:8000) — vector similarity search

All traffic stays on the local network.
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| "does not support chat" in Open WebUI | `nomic-embed-text` selected as chat model | Switch to a chat model (e.g. `llama3.2`) in the model dropdown |
| "Connect a model" on Open WebUI load | `OLLAMA_BASE_URL=http://127.0.0.1` resolves to container | Use `http://host.docker.internal:11434` |
| Embed timeouts during ingest | CPU embedding is slow; Ollama busy | Script retries 3x with backoff; reduce file sizes or add `--path` to index in batches |
| Chroma 410 Gone errors | Chroma v2 uses different API paths | Update to `/api/v2/tenants/default_tenant/databases/default_database/...` |
| VM IP not in ARP table after boot | DHCP lease held by router, not Proxmox | Ping-sweep the subnet: `for i in $(seq 1 254); do ping -c1 -W1 192.168.X.$i &>/dev/null & done; wait; arp -an` |
| Ingest stuck on one directory for hours | Package cache or media library in path | Kill, add directory to `SKIP_DIRS` in the script, restart |

---

## Phase 2 — GPU Passthrough

See `davidwhittington/proxmox — docs/proxmox-egpu-spec.md` for the full Thunderbolt 4 eGPU passthrough implementation. Once complete, Ollama will use the AMD Vega 56 GPU via ROCm/Vulkan backend, reducing 8B model inference from ~2-3 seconds to under 200ms.

---

## Changelog

- 2026-03-18: Initial setup and RAG pipeline documented
