---
name: aesop-wiki
description: "Aesop on-device deploy stack — Termux daemons, llama-server supervision, WebSocket bridge, voice pipeline, and model management for Snapdragon 8 Elite. Use when working on the Aesop repo, llamad/aesopd daemons, deploy scripts, bridge protocol, or the Termux side of the on-device AI stack. Also use when the user says 'Aesop,' 'the bridge,' 'llamad,' 'daemons,' or references Termux-side deployment."
---

# Aesop Wiki

## What Aesop Is

Aesop is the **deploy and runtime stack** for on-device AI running in Termux on Snapdragon 8 Elite. It owns the GGML inference plane (llama-server with Gemma 4 12B), the WebSocket bridge to the Horizons app, daemon supervision via runit/termux-services, and the voice pipeline wiring.

The repo lives at `c10vis-poem/aesop`. The Termux-specific operational knowledge is in the `termux-helper` skill — Aesop-wiki covers the project's own code and architecture.

## Repo Structure

```
deploy/phone/daemons/
├── llamad-run          — runit run script for llama-server
├── aesopd-run          — runit run script for WebSocket bridge
└── install-daemons.sh  — symlinks services into termux-services
protocol/
└── bridge-protocol.md  — aesopd wire spec (JSON over RFC 6455)
skills/
└── termux-helper/      — canonical copy of the termux-helper skill
```

## llamad — The GGML Plane

llama-server on port 8081, supervised by runit.

### Model

**Gemma 4 12B IT QAT, Q4_0 quant.** llama.cpp runtime-repacks Q4_0 into the i8mm/dotprod aarch64 layout on Snapdragon 8 Elite; K-quants (Q4_K_XL etc.) don't get that optimization. If both files are on disk, the run script prefers `*q4_0*.gguf`.

Models are already on device. Discovery path searches `~/storage/downloads/` and `~/models/`. Never download or quantize — these are pre-quantized.

### Memory Posture

- **mmap ON** (the default). File-backed pages are reclaimable — pressure evicts pages instead of the low-memory killer taking the whole process. `--no-mmap` turns a slowdown into a death.
- `-fa -ctk q8_0 -ctv q8_0` — roughly halves KV cache at 4k context
- `-t 6` — big cores only; more threads fight the LITTLE cluster
- Resident cost: ~7GB model + ~0.5-1GB KV cache while serving

### Backend Ladder

Inside Termux, the accelerator ladder is:
1. Adreno 830 via OpenCL (if the llama.cpp build carries it)
2. CPU big cores (always available)

**NOT on the ladder: Hexagon DSP / HTP.** Termux cannot reach it. See the `termux-helper` skill Part 2 for the full explanation.

## aesopd — The Bridge

Pure-stdlib RFC 6455 WebSocket server on port 8765. One JSON object per text frame; `id` field correlates requests to streamed replies.

### Routing

- `llm.generate` with `backend: "ggml"` → `POST 127.0.0.1:8081/v1/chat/completions`
- `llm.generate` with `backend: "npu"` → `POST 127.0.0.1:8080/api/v1/generate`

llama-server applies the **GGUF's embedded chat template** server-side. Never hand-roll Gemma turn markers on the client side.

The data planes are independent of the control plane — killing aesopd never interrupts a generation already running over direct HTTP.

## Daemon Supervision

```bash
pkg install termux-services   # then fully restart Termux
bash ~/aesop/deploy/phone/daemons/install-daemons.sh
sv status llamad aesopd
```

- `sv up|down|restart <svc>` — manual control
- `tail -f $PREFIX/var/log/llamad/current` — svlogd output
- `termux-wake-lock` before long runs; `termux-wake-unlock` after

## Recent History

| Commit | Change |
|--------|--------|
| `f28c303` | Version-controlled termux-helper skill; corrected NPU architecture |
| `08cb9f9` | llamad: dropped Hexagon-from-Termux framing, named the real HTP path |
| `49a958a` | llamad: full accelerator offload (-ngl) + honest backend-ladder docs |
| `dc80402` | llamad: Q4_0 preference, mmap + q8_0 KV cache for tight-RAM survival |

## Credit

Mer0vin8ian Production — Cl0vis/Claude collab.
