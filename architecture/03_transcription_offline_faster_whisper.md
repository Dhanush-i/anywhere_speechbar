# SOP-03: Transcription — Offline (faster-whisper + CUDA)

> **Layer 1 – Architecture SOP**  
> Subsystem: Speech-to-text transcription (offline engine)  
> Golden Rule: Update this SOP **before** changing any transcription logic in `tools/`.

---

## 1. Purpose

This SOP defines the offline transcription pipeline for anywhere-speechbar using **faster-whisper** with NVIDIA CUDA acceleration. It covers model loading, inference parameters, output handling, and the optional online fallback path.

---

## 2. Engine Selection Logic

```
config.engine_mode
   ├─ "offline"         → always use faster-whisper (default)
   ├─ "online"          → always use online STT (requires API key)
   └─ "auto"            → try offline first; if error or low confidence, try online
```

---

## 3. Offline Engine: faster-whisper

### 3.1 Model Configuration

| Parameter | Value | Notes |
|---|---|---|
| Model | `medium` | Best accuracy / speed balance for Indian accent on RTX 3050 |
| Device | `cuda` | NVIDIA GPU acceleration (BR-08) |
| Compute type | `float16` | Halved memory; full accuracy on CUDA |
| Beam size | `5` | Good accuracy; increase to 10 for max accuracy at cost of speed |
| VAD filter | `true` | Skips silence; required for clean audio with minimal ambient noise |
| VAD threshold | `0.5` | Default; tune down (0.3) in noisy environments |
| Language | `en` | English only (BR-07) |
| Task | `transcribe` | Not translate |

### 3.2 Model Loading

- Model is loaded **once** at application startup and held in memory for the lifetime of the process.
- Loading is done in a background thread to avoid blocking the UI on first run.
- Model files are cached by faster-whisper in the default Hugging Face cache directory (`~/.cache/huggingface/`).
- First run downloads the model; subsequent runs use the cache.

### 3.3 Inference Call

```python
segments, info = model.transcribe(
    audio_path,           # path to .tmp/session_<id>.wav
    language="en",
    task="transcribe",
    beam_size=5,
    vad_filter=True,
    vad_parameters={"threshold": 0.5}
)
transcript_raw = " ".join(seg.text.strip() for seg in segments)
```

### 3.4 Output

| Field | Type | Description |
|---|---|---|
| `transcript_raw` | `str` | Raw Whisper output (may have minor errors) |
| `engine_used` | `"offline"` | Identifies which engine was used |
| `transcribe_ms` | `int` | Wall-clock time for transcription in ms |

---

## 4. Online Engine (Optional)

> Enabled when `config.online_engine.enabled = true` **or** when `engine_mode = "online"` / `"auto"`.

### 4.1 Supported Providers

| Provider | API | Model |
|---|---|---|
| OpenAI | `openai.Audio.transcribe` | `whisper-1` |
| Google (future) | Cloud Speech-to-Text v2 | TBD |
| Azure (future) | Speech Services SDK | TBD |

### 4.2 Online Call Flow (OpenAI example)

```python
with open(audio_path, "rb") as f:
    response = openai_client.audio.transcriptions.create(
        model="whisper-1",
        file=f,
        language="en"
    )
transcript_raw = response.text
```

### 4.3 API Key

- Stored in `.env` as `ONLINE_STT_API_KEY`.
- Loaded with `python-dotenv` at startup.
- Never committed to source control (`.env` is in `.gitignore`).

---

## 5. Post-Processing (shared, after either engine)

Post-processing is handled by `tools/postprocess_text.py` (see also [SOP-04](04_text_injection_clipboard.md)).

| Step | Rule |
|---|---|
| Capitalize first letter | Always capitalize the first character of the transcript |
| Capitalize "i" → "I" | Replace standalone lowercase `i` with `I` |
| Fix spacing | Remove double spaces; trim leading/trailing whitespace |
| Punctuation normalization | Ensure sentence-ending punctuation exists if missing |
| Indian accent corrections | Mapping table for common mishears (e.g., "wont" → "won't", "cant" → "can't") |
| Spell correction | Optional lightweight pass using a dictionary or language model |

---

## 6. Error Handling

| Error | Behaviour |
|---|---|
| CUDA not available | Log warning; fall back to CPU (slower) |
| Model load failure | Log error; show "STT unavailable" in overlay; return to IDLE |
| Transcription timeout (> 60 s) | Log error; return empty transcript |
| Online API error | Log error; if `engine_mode=auto` try offline; otherwise return error |
| Empty transcript | Log warning; skip paste; return to IDLE silently |

---

## 7. CUDA Prerequisites

```
NVIDIA driver ≥ 527.x
CUDA Toolkit 11.8 or 12.x
cuDNN 8.x (placed in CUDA lib path)
pip install faster-whisper
```

---

## 8. Tool Files

| Tool | Path |
|---|---|
| Offline transcription | `tools/transcribe_offline.py` |
| Online transcription | `tools/transcribe_online.py` |
| Post-processing | `tools/postprocess_text.py` |

---

## 9. Linked SOPs

- [SOP-02: Audio Capture](02_audio_capture.md) — provides WAV input
- [SOP-04: Text Injection](04_text_injection_clipboard.md) — receives cleaned transcript
- [SOP-06: Config Management](06_config_and_persistence.md) — engine/model parameters
