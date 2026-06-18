# plow-seedlab-video-editing

**The single source of truth for the video-editing pipeline seed.**

This repo contains exactly one canonical generative seed: [`video-editing.seed.md`](./video-editing.seed.md).
An AI agent reads the seed and one-shots the video-editing pipeline from it ("Edit this video" →
consistent, gated quality every time). The seed is generative (instructions + contracts + acceptance
journeys), not pre-baked editing code.

## Why this repo exists
The seed previously lived in multiple divergent places (several git branches + an implementation
monorepo), which caused confusion. Per CEO directive there is now **ONE place** — this repo. All other
copies are retired.

## Provenance
Extracted from the hardened canonical version `plow-pbc/seedlab @ bb9fbdb`
(`seeds/video-editing/video-editing.seed.md`, 908 lines, all 3 flows) with two CEO corrections folded in:
- **Gimbal / head-following is OPT-IN** — never the default, never a hard-gate (request-only).
- **The last-3-minutes is a DEV-VALIDATION harness only** — production edits the WHOLE video.

## Three flows (the seed picks one)
- **FLOW 1 — COMBINE**: concatenate human-edited pieces as-is.
- **FLOW 2 — PROCESS (reels)**: raw footage → short vertical 9:16 reel.
- **FLOW 3 — MULTICAM LONG-FORM**: raw footage → long landscape 16:9 video.

## External dependencies (accessible, NOT run/generated here)
- Parakeet STT API (`parakeet-tdt-0.6b-v3`) — referenced via the `STT_URL` input.
- SAM3 API — referenced via the `--sam3-url` input.

These are referenced by variable name with a LAN default; no credentials are stored in this repo.
