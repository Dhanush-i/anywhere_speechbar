# SOP-02: Audio Capture

> **Layer 1 – Architecture SOP**  
> Subsystem: Microphone audio capture  
> Golden Rule: Update this SOP **before** changing any audio capture logic in `tools/`.

---

## 1. Purpose

This SOP defines how anywhere-speechbar captures microphone audio from the default Windows audio input device. It governs buffer format, sample rate, file output, error handling, and cleanup.

---

## 2. Audio Capture Specification

| Parameter | Value | Source |
|---|---|---|
| Input device | Default system mic | BR-07; `config.json → audio.device` |
| Sample rate | 16 000 Hz | Whisper requirement |
| Channels | 1 (mono) | Whisper requirement |
| Data type | `float32` | sounddevice default |
| Chunk size | 1 024 frames | Balances latency and overhead |
| Max duration | 120 seconds (safety cap) | Prevents unbounded memory use |

---

## 3. Capture Flow

```
[hotkey_listener] ──state: LISTENING──► [capture_audio.py]
                                              │
                              sounddevice.InputStream opens
                              (default mic, 16 kHz, mono, float32)
                                              │
                              Audio frames appended to in-memory list
                              (numpy arrays, 1 024 frames each)
                                              │
                         [hotkey_listener] ──state: STOP──►
                                              │
                              numpy.concatenate all frames
                              Write to .tmp/session_<uuid>.wav
                              (scipy.io.wavfile or soundfile)
                                              │
                              Returns: path to WAV file + duration_ms
```

---

## 4. Intermediate File Convention

- All temporary audio files are written to `.tmp/` at the project root.
- File naming: `.tmp/session_<uuid4>.wav`
- Cleanup: the WAV file is **deleted** after transcription completes (success or error).
- `.tmp/` is listed in `.gitignore` and must never be committed.

---

## 5. 1-Second Pre-Record Delay

- The audio capture tool does **not** handle the delay — this is the responsibility of `hotkey_listener.py` / the state machine (see SOP-01).
- `capture_audio.py` starts recording **immediately** when called; the 1-second delay has already elapsed before the call is made.

---

## 6. Error Handling

| Error | Behaviour |
|---|---|
| No microphone detected | Log error, emit `diagnostics.errors`, return to IDLE state |
| InputStream open failure | Same as above |
| Capture buffer exceeds 120 s | Stop recording automatically; proceed to transcription |
| Disk write failure for WAV | Log error, attempt transcription from in-memory buffer if possible |

---

## 7. Dependencies

| Package | Purpose |
|---|---|
| `sounddevice` | Cross-platform PortAudio bindings for mic capture |
| `numpy` | Audio frame array accumulation and concatenation |
| `soundfile` | Write captured PCM to WAV file on disk |

---

## 8. Tool File

| Tool | Path |
|---|---|
| Audio capture | `tools/capture_audio.py` |

---

## 9. Linked SOPs

- [SOP-01: Hotkey & State Machine](01_hotkey_and_state_machine.md) — triggers start/stop of capture
- [SOP-03: Transcription (Offline)](03_transcription_offline_faster_whisper.md) — consumes WAV output
