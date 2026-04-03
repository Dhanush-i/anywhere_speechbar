# anywhere-speechbar

> A Windows 11 global dictation utility — speak anywhere, paste everywhere.

## What it does

- Toggle with a global hotkey (`Ctrl+Alt+Space` by default, user-changeable)
- Shows a bottom-center, always-on-top pill overlay ("Listening…") during capture
- Records audio from the default microphone (with a 1-second startup delay)
- Transcribes using **offline** faster-whisper (CUDA/GPU, model=medium) or an **online** cloud STT API
- Post-processes the transcript: casing, punctuation, spelling, and Indian accent corrections
- Instantly pastes the final text into whichever app is currently focused (clipboard injection)
- Restores your clipboard after paste

## Project Status

| Phase | Name | Status |
|---|---|---|
| 1 | Blueprint (Vision & Logic) | ✅ Complete |
| 2 | Link (Connectivity) | 🔲 Pending |
| 3 | Architect (3-Layer Build) | 🔲 Pending |
| 4 | Stylize (Refinement & UI) | 🔲 Pending |
| 5 | Trigger (Deployment) | 🔲 Pending |

## Documentation

- [`gemini.md`](gemini.md) — Project source of truth: config schema, behavioral rules, data schema, maintenance log
- [`architecture/`](architecture/) — Technical SOPs per subsystem:
  - [01 – Hotkey & State Machine](architecture/01_hotkey_and_state_machine.md)
  - [02 – Audio Capture](architecture/02_audio_capture.md)
  - [03 – Transcription (Offline: faster-whisper)](architecture/03_transcription_offline_faster_whisper.md)
  - [04 – Text Injection via Clipboard](architecture/04_text_injection_clipboard.md)
  - [05 – UI Overlay Pill (PyQt6)](architecture/05_ui_overlay_pill_pyqt6.md)
  - [06 – Config & Persistence](architecture/06_config_and_persistence.md)

## Requirements (Phase 2+)

- Windows 11
- Python 3.11+
- NVIDIA GPU (RTX 3050 or better) with CUDA 11.8+
- See [`gemini.md`](gemini.md) for full package list

## B.L.A.S.T. Protocol

This project follows the **B.L.A.S.T. Master Protocol** (Blueprint → Link → Architect → Stylize → Trigger) and the **A.N.T. 3-layer architecture** (architecture/ → navigation → tools/).
