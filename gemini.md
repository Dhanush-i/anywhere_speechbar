# gemini.md — Project Map & Source of Truth
> **anywhere-speechbar** | B.L.A.S.T. Master Protocol | Phase 1: Blueprint

---

## 1. Project Identity

| Field | Value |
|---|---|
| **Project name** | anywhere-speechbar |
| **Protocol** | B.L.A.S.T. (Blueprint → Link → Architect → Stylize → Trigger) |
| **Architecture** | A.N.T. 3-layer (architecture/ → navigation → tools/) |
| **Current phase** | Phase 1 – Blueprint ✅ |
| **Platform** | Windows 11 |
| **Primary language** | Python 3.11+ |

---

## 2. North Star

> **Singular desired outcome:**  
> A Windows 11 global dictation utility that:
> 1. Toggles via a user-changeable hotkey (default `Ctrl+Alt+Space`)
> 2. Shows a bottom-center, always-on-top pill overlay ("Listening…") during capture
> 3. Waits 1 second after hotkey press before recording starts
> 4. Transcribes recorded speech using an **offline** engine (faster-whisper, GPU, model=medium) **or** an **online** cloud STT API (user-selectable, fallback optional)
> 5. Post-processes the transcript (casing, punctuation, spelling correction, Indian accent normalization)
> 6. Immediately pastes the final text into the currently focused application via clipboard injection
> 7. Restores the user's clipboard after paste

---

## 3. Behavioral Rules (Hard Constraints)

| # | Rule |
|---|---|
| BR-01 | Hotkey is `Ctrl+Alt+Space` by default; user-changeable via `config.json`. |
| BR-02 | On first hotkey press: wait exactly **1,000 ms**, then start recording. |
| BR-03 | On second hotkey press (or silence-stop): stop recording, transcribe, paste. |
| BR-04 | Overlay pill is **always-on-top**, **bottom-center**, **non-draggable** (matches screenshot). |
| BR-05 | Default commit mode is **instant paste** (no preview dialog). |
| BR-06 | Clipboard must be **restored** to its pre-paste contents after injection. |
| BR-07 | Language is **English only**; model and post-processing must be tuned for Indian accent. |
| BR-08 | Offline engine uses CUDA (RTX 3050) with `float16` compute type. |
| BR-09 | Online engine is optional; disabled by default. |
| BR-10 | All intermediate audio files go into `.tmp/`; cleaned up after each session. |
| BR-11 | **No code in `tools/`** until the Blueprint is confirmed by the user. |
| BR-12 | If logic changes, update the SOP in `architecture/` **before** updating the code. |

---

## 4. Integrations

| Service | Status | Key required |
|---|---|---|
| faster-whisper (CUDA) | Offline — no key | ❌ |
| Online STT provider | TBD (OpenAI / Google / Azure) | TBD |
| PyQt6 (UI) | Local package | ❌ |
| sounddevice (audio) | Local package | ❌ |
| keyboard (hotkey) | Local package | ❌ |
| pyperclip (clipboard) | Local package | ❌ |

---

## 5. Source of Truth (Data Flow)

```
[Default Mic]
     │  PCM audio (16 kHz, mono)
     ▼
[Audio Capture Tool]  ──writes──►  .tmp/session_<id>.wav
     │
     ▼
[Transcription Tool]
     ├─ offline: faster-whisper CUDA
     └─ online:  Cloud STT API (optional)
     │  raw transcript string
     ▼
[Post-Processor]  ──  casing / punctuation / spelling
     │  cleaned transcript string
     ▼
[Text Injection Tool]  ──  clipboard swap + Ctrl+V
     │
     ▼
[Focused Application]  (any text field, anywhere)
     │  (clipboard restored)
```

---

## 6. JSON Data Schema

### 6.1 Config Schema (`config.json`)

```json
{
  "config": {
    "hotkey_toggle": "ctrl+alt+space",
    "start_delay_ms": 1000,
    "language": "en",
    "engine_mode": "offline",
    "offline_engine": {
      "type": "faster-whisper",
      "model": "medium",
      "device": "cuda",
      "compute_type": "float16",
      "vad_filter": true,
      "vad_threshold": 0.5,
      "beam_size": 5
    },
    "online_engine": {
      "enabled": false,
      "provider": "openai",
      "model": "whisper-1",
      "api_key_env": "ONLINE_STT_API_KEY"
    },
    "post_processing": {
      "fix_casing": true,
      "fix_punctuation": true,
      "fix_spelling": true,
      "indian_accent_corrections": true
    },
    "commit_mode": "instant_paste",
    "paste": {
      "method": "clipboard_ctrl_v",
      "restore_clipboard": true,
      "paste_delay_ms": 100
    },
    "ui": {
      "position": "bottom_center",
      "style": "pill",
      "always_on_top": true,
      "bottom_margin_px": 40,
      "width_px": 260,
      "height_px": 52
    },
    "audio": {
      "device": "default",
      "sample_rate_hz": 16000,
      "channels": 1,
      "dtype": "float32"
    }
  }
}
```

### 6.2 Session Schema (in-memory / log)

```json
{
  "session": {
    "session_id": "uuid-v4-string",
    "timestamp_utc": "2024-01-01T00:00:00Z",
    "audio_device": "default",
    "audio_format": {
      "sample_rate_hz": 16000,
      "channels": 1,
      "dtype": "float32"
    },
    "tmp_audio_path": ".tmp/session_<id>.wav",
    "engine_used": "offline"
  }
}
```

### 6.3 Output Schema

```json
{
  "result": {
    "status": "success | cancelled | error",
    "transcript_raw": "string",
    "transcript_clean": "string",
    "engine_used": "offline | online",
    "timings_ms": {
      "delay_ms": 1000,
      "record_ms": 0,
      "transcribe_ms": 0,
      "postprocess_ms": 0,
      "inject_ms": 0
    }
  },
  "diagnostics": {
    "errors": [],
    "warnings": []
  }
}
```

---

## 7. Planned Project Structure

```
anywhere-speechbar/
├── gemini.md                          ← This file (source of truth)
├── README.md                          ← Public-facing intro
├── config.json                        ← User configuration (created on first run)
├── .env                               ← API keys (online STT, if enabled; gitignored)
├── .tmp/                              ← Intermediate audio files (gitignored)
├── architecture/                      ← Layer 1: SOPs (Markdown)
│   ├── 01_hotkey_and_state_machine.md
│   ├── 02_audio_capture.md
│   ├── 03_transcription_offline_faster_whisper.md
│   ├── 04_text_injection_clipboard.md
│   ├── 05_ui_overlay_pill_pyqt6.md
│   └── 06_config_and_persistence.md
└── tools/                             ← Layer 3: Atomic Python tools (Phase 2+)
    ├── hotkey_listener.py
    ├── capture_audio.py
    ├── transcribe_offline.py
    ├── transcribe_online.py
    ├── postprocess_text.py
    ├── inject_text_clipboard.py
    ├── ui_overlay.py
    ├── config_manager.py
    └── app.py                         ← Orchestrator entry point
```

---

## 8. Python Package Manifest (Phase 2 install list)

```
faster-whisper          # offline STT (CUDA)
sounddevice             # microphone capture
numpy                   # audio buffer handling
PyQt6                   # overlay pill UI
keyboard                # global hotkey listener
pyperclip               # clipboard read/write
python-dotenv           # .env loading for API keys
openai                  # optional: OpenAI online STT
```

> CUDA prerequisite: NVIDIA CUDA Toolkit 11.8+ and cuDNN 8 must be installed on host.

---

## 9. Phase Tracker

| Phase | Name | Status |
|---|---|---|
| 0 | Initialization | ✅ Complete |
| 1 | Blueprint (Vision & Logic) | ✅ Complete |
| 2 | Link (Connectivity) | 🔲 Pending user confirmation |
| 3 | Architect (3-Layer Build) | 🔲 Pending Phase 2 |
| 4 | Stylize (Refinement & UI) | 🔲 Pending Phase 3 |
| 5 | Trigger (Deployment) | 🔲 Pending Phase 4 |

---

## 10. Maintenance Log

| Date | Phase | Entry |
|---|---|---|
| 2024-04-03 | 0 | Repository initialized: `anywhere-speechbar`. README created. |
| 2024-04-03 | 1 | Blueprint phase complete. `gemini.md` written as source of truth. `architecture/` SOPs drafted for all 6 subsystems. No code in `tools/` yet per Protocol 0. Awaiting user confirmation to proceed to Phase 2. |

---

## 11. Self-Annealing Protocol

When any tool in `tools/` fails:
1. Read the full stack trace.
2. Identify the failing tool file and the relevant SOP in `architecture/`.
3. Patch the script in `tools/`.
4. Re-run the tool to confirm fix.
5. Update the corresponding `architecture/*.md` SOP to reflect the change.
6. Add a one-line entry to the Maintenance Log above.

---

*End of gemini.md — Phase 1 Blueprint.*
