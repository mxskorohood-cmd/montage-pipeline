---
name: montage-pipeline
description: End-to-end orchestrator for the Montage auto-editing project — the map from raw talking-head clip to published video. Read this FIRST when the repo is opened or someone wants to edit a video. Walks the full pipeline: onboarding (brand-intake) → analysis (transcribe → cut markup → EDL → review → faces → accents → props → declick audio) → review in Remotion Studio → final render → publish texts (publish-pack). Triggers — "запусти проект", "начать монтаж", "новое видео", "смонтируй видео", "start", "run pipeline", "с чего начать", first repo open.
metadata:
  tags: pipeline, orchestrator, montage, remotion, workflow
---

## What this is

The single entry point for the Montage project. It doesn't do the work itself — it
routes to the right stage, skill, and command, and keeps the run in order. Read
`README.md`, `CLAUDE.md`, and `docs/PIPELINE.md` for detail; this skill is the sequence.

**On first open of the repo:** greet the user, show a 3-line summary of what the project
does (from `README.md`), tell them the two ways to start, and wait.
- New look / first run → **`brand-intake`** (configure brand, fonts, effects, sound, pace).
- Have a clip to edit → go to Stage 1 below.

Do not render or push anything without an explicit user command. Preview is Studio-only.

## The pipeline (per video `<id>`, artifacts in `work/<id>/`)

**Stage 0 — Onboarding (optional, once per brand).** Run skill `brand-intake` →
`brand.config.json` (brand, font, `effects`, `sfx`, `remotionEffects`, `editStyle`, inserts).
Apply the config to `docs/brandbook.md`, `src/fonts.ts`, `pipeline/props.py` constants,
and the composition effect/sound layers before editing. See `brand-intake` for the mapping.

**Stage 1 — Ingest + analysis (Python, `.venv`).** Full command list in `docs/PIPELINE.md`:
1. `ffmpeg … audio.wav` (16 kHz mono) — analysis audio.
2. `transcribe.py` (Deepgram, key from `.env`) → `words.json`, `transcript.txt`, `utterances.txt`.
3. `indexed.py` → `indexed.txt` (compact view for markup).
4. **Claude** reads `utterances.txt` + `indexed.txt`, writes `delete.json` (cut fillers/dupes —
   Deepgram filler_words is unreliable for Russian, so Claude does it).
5. `edl.py` → `edl.json` (segments + frames). Never hand-edit; always via `delete.json`.
6. `review.py` → `REVIEW.md`. **Mandatory review point** — show the user, wait for OK.
7. `faces.py` → `faces.json` (face position every 5 s).
8. **Claude** writes `accents.json` (accent words shown on screen).
9. `props.py` → `remotion/props/main169.json`.
10. `audio.py <props>` → declick speaker track `audio_main169.wav` (rerun after ANY edit change).

**Stage 2 — Shorts (optional).** Pick moments → `work/<id>/shorts.json`, then
`shorts.py work\<id> remotion\props` → `props/<id>.json`, then `audio.py` per short.
Subtitle text is hand-cleaned in `shorts.json` (Deepgram lies on numbers/words).

**Stage 3 — Review in Studio.** `cd remotion && npx remotion studio`. User previews Main169
(16:9) and `Short-*` (9:16). Preview is NEVER a file render. Edits loop back:
`delete.json` → `edl.py` → `props.py` → **`audio.py`** (rerun audio every time).

**Stage 4 — Final render (only on explicit user command).** High quality:
`npx remotion render src/index.ts Main169 ../out/main169.mp4 --props=props/main169.json --image-format=png --crf=14 --x264-preset=slow`.
Long job → run in background with a completion notice; watch by output-file growth, not by
process name. Bump `--concurrency` if a single stuck frame can deadlock the run.

**Stage 5 — Publish texts.** Run skill `publish-pack` → `work/<id>/publish.md`
(YouTube description in the author's style, timecodes from `edl.json`, `#`-tags, comma tags,
short Telegram post). Uses the transcript already produced — no re-transcription.

## Guardrails (from CLAUDE.md — do not violate)
- Preview only in Studio; final render only on explicit command; no ProRes.
- Speaker sound = declick track, not `<Video>` audio. No normalization/denoise on the voice.
- Composition size always equals target resolution (1920×1080 / 1080×1920).
- Version pins: TypeScript 5.9.3, zod 4.3.6, mediapipe 0.10.21. `@remotion/media` trims in FRAMES.
- Composition `id`: no `_` (use `-`). PowerShell 5.1: no `&&`; Cyrillic paths in quotes.

## Reply pattern
At each stage: say which stage you're on, run its command(s), show the artifact/summary,
and name the next step. Stop at the two hard gates: `REVIEW.md` (Stage 1.6) and the final
render (Stage 4). Never skip them.
