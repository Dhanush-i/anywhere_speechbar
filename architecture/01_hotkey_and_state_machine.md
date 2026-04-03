# SOP-01: Hotkey & State Machine

> **Layer 1 – Architecture SOP**  
> Subsystem: Global hotkey listener and application state machine  
> Golden Rule: Update this SOP **before** changing any hotkey/state logic in `tools/`.

---

## 1. Purpose

This SOP defines the exact behavior of the global hotkey listener and the application state machine for anywhere-speechbar. It governs when recording starts, stops, and how the application transitions between idle, listening, transcribing, and injecting states.

---

## 2. Default Hotkey

| Action | Hotkey | Configurable |
|---|---|---|
| Toggle listening on/off | `Ctrl + Alt + Space` | ✅ Yes, via `config.json` → `hotkey_toggle` |
| Cancel and discard | `Esc` | 🔲 Future feature |

---

## 3. State Machine

```
┌──────────┐
│  IDLE    │  ◄──────────────────────────────────────┐
└──────────┘                                         │
     │                                               │
     │  hotkey pressed                               │
     ▼                                               │
┌──────────────┐                                     │
│  WAITING     │  (1 000 ms delay — BR-02)           │
│  (countdown) │                                     │
└──────────────┘                                     │
     │                                               │
     │  delay elapsed                                │
     ▼                                               │
┌──────────────┐                                     │
│  LISTENING   │  ◄──  overlay shows "Listening…"    │
│  (recording) │                                     │
└──────────────┘                                     │
     │                                               │
     │  hotkey pressed again (or silence-stop)       │
     ▼                                               │
┌──────────────┐                                     │
│ TRANSCRIBING │  ◄──  overlay shows "Transcribing…" │
└──────────────┘                                     │
     │                                               │
     │  transcript ready                             │
     ▼                                               │
┌──────────────┐                                     │
│  INJECTING   │  ◄──  clipboard swap + Ctrl+V       │
└──────────────┘                                     │
     │                                               │
     │  paste complete / clipboard restored          │
     └─────────────────────────────────────────────►─┘
          (back to IDLE)
```

### Error path

Any unhandled exception in LISTENING, TRANSCRIBING, or INJECTING transitions directly to IDLE with an error logged and the overlay dismissed.

---

## 4. State Definitions

| State | Description | Overlay text | Actions available |
|---|---|---|---|
| `IDLE` | App running in tray/background; no overlay | Hidden | Hotkey → WAITING |
| `WAITING` | 1-second countdown before mic opens | "Starting…" (optional) | — |
| `LISTENING` | Mic open; audio being buffered | "Listening…" | Hotkey → TRANSCRIBING |
| `TRANSCRIBING` | Audio handed to STT engine | "Transcribing…" | — (async) |
| `INJECTING` | Pasting transcript into active window | Brief flash or hide | — |

---

## 5. Hotkey Implementation Rules

1. Use the `keyboard` Python library for global hotkey registration on Windows 11.
2. The hotkey callback must be **non-blocking** — it posts an event to the application's event queue (or calls a thread-safe signal); it must **not** perform I/O directly.
3. Hotkey string is loaded from `config.json` at startup. If the key is invalid, log an error and fall back to `ctrl+alt+space`.
4. The hotkey must be **suppressed** (not forwarded to the active application) to prevent accidental key presses in the target app.
5. Re-registration occurs whenever the user changes the hotkey via settings (Phase 4).

---

## 6. 1-Second Delay Logic

- When the WAITING state is entered, a `threading.Timer(1.0, callback)` is created.
- If the user presses the hotkey again during the WAITING period, the timer is cancelled and the app returns to IDLE.
- The delay is configurable via `config.json` → `start_delay_ms`.

---

## 7. Silence-Based Auto-Stop (Future)

> Planned for Phase 4. Not implemented in Phase 2/3.

- VAD (Voice Activity Detection) can be enabled in faster-whisper (`vad_filter=True`).
- For auto-stop, a rolling silence window (e.g., 2 s of silence after speech) will trigger the TRANSCRIBING transition automatically.
- This will be controlled by a `silence_auto_stop_ms` config key.

---

## 8. Tool File

| Tool | Path |
|---|---|
| Hotkey listener | `tools/hotkey_listener.py` |
| App orchestrator | `tools/app.py` |

---

## 9. Linked SOPs

- [SOP-02: Audio Capture](02_audio_capture.md) — started when state → LISTENING
- [SOP-05: UI Overlay](05_ui_overlay_pill_pyqt6.md) — updated on every state transition
- [SOP-06: Config Management](06_config_and_persistence.md) — hotkey key loaded from config
