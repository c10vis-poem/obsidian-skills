---
name: horizons-repo-skill
description: "Portable copy of the Horizons Android app's stable architecture — for use outside the Novus-Agenti repo (Obsidian vault, Drive, sessions without repo access). Use when working on the Horizons Kotlin/Compose app, its HomeGrid UI, theme system, font resources, NPU/HTP integration, or the loopback bridge to Termux, and the actual repo isn't checked out. Also use when the user says 'Horizons,' 'the app,' 'home screen,' 'router,' or references the Android side of the on-device AI stack. Not the canonical source — Novus-Agenti's own CLAUDE.md is the wiki of record; this mirrors its stable architecture only, never current state."
---

# Horizons Repo Skill

Portable mirror of the Horizons/Novus-Agenti architecture, for contexts without repo access (Obsidian, Drive, other sessions). **The actual `CLAUDE.md` in `c10vis-poem/Novus-Agenti` is the wiki of record** — its `## State of the Union` section is the only source of current state. This file carries only stable architecture that doesn't change week to week; it is not a substitute for reading CLAUDE.md when the repo is available, and it never duplicates SOTU content.

There is also a separate, differently-scoped skill inside the repo itself: `skills/horizons-wiki/SKILL.md` — a cache-optimized bundle loader (points at CLAUDE.md + two knowledge docs, no inline content). That one and this one solve different problems and are not meant to be reconciled into one file.

## What Horizons Is

Horizons is an Android app (Kotlin, Jetpack Compose, WebView hybrid) that serves as the **frontend and NPU gateway** for an on-device AI stack running on Snapdragon 8 Elite. It owns the app-context vendor libraries that Termux cannot reach, and exposes them over HTTP on loopback.

The app is both a user-facing interface (HomeGrid, settings, chat, terminal panels) and a backend router that bridges the NPU/HTP plane to the Termux GGML plane.

## Architecture

### Loopback Bridge

Android shares `127.0.0.1` across all processes. Horizons and Termux communicate over this shared loopback — no IPC, no intents, just HTTP.

| Port | Owner | Service |
|------|-------|---------|
| 8080 | **Horizons** (app) | `ort_engine` — GENIE-compiled NPU contexts (Gemma 4 E4B, Qwen 2B QAI) |
| 8081 | **Termux** | `llamad` → llama-server — Gemma 4 12B Q4_0 GGML |
| 8091 | **Horizons** (app) | Media pipeline — Moonshine STT / Kokoro TTS |
| 8765 | **Termux** | `aesopd` WebSocket bridge — routing, events, control plane |

**Port discipline is non-negotiable.** Horizons must never bind 8081 or 8765. Termux must never bind 8080 or 8091. Collisions are silent and miserable to debug.

### NPU Access

Horizons runs in app context with access to Qualcomm vendor libraries (QNN, GENIE, QAIRT). It loads GENIE-compiled context binaries and serves inference over HTTP.

Termux **cannot** reach the Hexagon DSP — FastRPC vendor libs are outside its sandbox and SELinux blocks access. Any plan that has Termux loading QNN directly is wrong. Termux reaches the NPU by asking Horizons over HTTP.

See the `termux-helper` skill for the Termux side of this split.

### Future: Kotlin-Accessible llama.cpp

Next mission: Horizons will also host a lightweight Kotlin-accessible llama.cpp instance that Termux agents can call into, hitting the HTP backend through the app's vendor lib access. This turns Horizons into a true backend router — the Termux agent in local shell mode shuts down its own llama-server and routes through the app instead.

## UI — HomeGrid (v5, complete)

The home screen is the `HomeGrid` composable. Current state as of commit `984b061` on `claude/homegrid-v5-tuned`:

### Layout Constants

- `TILE_INSET = 42.dp` — tiles pulled inward from edges
- `TOP_LIFT = 28.dp` — top three tiles lifted
- `HUB_LIFT = 28.dp` — center crystal hub lifted
- `LOGO_DP = 56.dp` (renders at 42dp via 0.75 multiplier)
- `SLOGAN_DP = 16.dp`
- ROUTER plate offset: `crystalSize * 0.97f + 40.dp`

### Typography

Both brand fonts are **static instances** cut with fontTools. Variable fonts with `fontVariationSettings` through XML resources are unreliable in Compose — static instances with baked-in weight values are the solution.

| Font | Weight | Use |
|------|--------|-----|
| Orbitron ExtraBold 800 | `orbitron_extrabold_static.ttf` | Wordmark "MO[)u14R_11C.", "(Next-Gen Certified)" |
| Google Sans Code Normal 400 | `google_sans_code_regular_static.ttf` | Strapline "*Pioneer_Tech," |

**Orbitron has no U+00D8 (Ø) glyph.** Android silently substitutes a fallback face for just that character, making it look like a completely different font. Always use plain `O`.

Type sizes use `dp` (not `sp`) to avoid device font-scale clipping.

### Colors (inline, not from theme)

The V5 palette is defined inline in HomeGrid.kt, not pulled from HorizonsTheme:

| Tile | Color |
|------|-------|
| Chat | `#4FE9A6` |
| Horizons | `#40C4FF` |
| Settings | `#FF5577` |
| Terminal | `#00FF41` |
| Archives | `#E8A838` |

### Starfield

- `BgStar` data class with 4 depth tiers
- Cross-glint on tier 3 stars
- Subtle twinkle: brightness oscillates 0.94–1.00 over a 7.6s cycle
- Telemetry clusters each encircle a twinkling star

### Font Resources

All XML weight resources and variable .ttf files have been **removed**. Only the two static instances remain in `res/font/`:
- `orbitron_extrabold_static.ttf`
- `google_sans_code_regular_static.ttf`

`audiowide.ttf` was also removed (replaced by Orbitron).

## Build

- `build-apk.yml` triggers on push to ALL branches and overwrites the shared `latest-debug` release tag — any push to any branch clobbers the release
- To build from a specific branch, use `workflow_dispatch` targeting that branch
- Snapshot branch `claude/homegrid-v5-SNAPSHOT-good-1837dc2` preserves the pre-layout-tuning build

## Known Issues

### Startup Crash (~10s)

`Breadcrumb.kt` was doing a full-file read of `crash.log` on the `startForegroundService()` 10-second deadline path. Fix pushed (tail-window read, in-memory fallback for FGS path, crash.log capped at 256KiB with rotation) but not confirmed on device yet.

### Theme Colors Stale

`HorizonsTheme.kt` contains colors that don't match V5's inline palette. V5 defines its own private color vals in HomeGrid.kt. The theme file needs updating to match, but V5 doesn't depend on it.

## Branches

| Branch | Purpose |
|--------|---------|
| `claude/homegrid-v5-tuned` | Active development, PR #31 |
| `claude/homegrid-v5-SNAPSHOT-good-1837dc2` | Pre-tuning snapshot |
| `main` | Base, behind active work |

## Credit

Mer0vin8ian Production — Cl0vis/Claude collab.
