# SOP-06: Config & Persistence

> **Layer 1 – Architecture SOP**  
> Subsystem: Configuration loading, validation, and persistence  
> Golden Rule: Update this SOP **before** changing any config logic in `tools/`.

---

## 1. Purpose

This SOP defines how anywhere-speechbar loads, validates, stores, and updates its configuration. It governs the `config.json` file format, default values, runtime overrides, and `.env` handling for API keys.

---

## 2. Config File Location

| Environment | Path |
|---|---|
| Development / portable | `<project_root>/config.json` |
| Installed (future) | `%APPDATA%\anywhere-speechbar\config.json` |

The project-root path is used for Phase 2/3. The AppData path is a Phase 5 (deployment) concern.

---

## 3. Config File: Full Default Schema

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

---

## 4. Config Loading Rules

1. On startup, `config_manager.py` attempts to load `config.json` from the project root.
2. If the file does **not exist**, the default schema (above) is written to disk and used.
3. If the file exists but is **malformed JSON**, log an error and fall back to in-memory defaults (do not overwrite user's file).
4. If the file exists but is missing keys, **merge with defaults**: use the user's values where provided, fill missing keys from defaults.
5. Config is stored as a plain Python `dict` in memory after loading. No live file-watching in Phase 2/3.

---

## 5. Environment Variables (`.env`)

| Key | Purpose | Required |
|---|---|---|
| `ONLINE_STT_API_KEY` | API key for the online STT provider | Only when `online_engine.enabled = true` |

- `.env` is loaded with `python-dotenv` at startup (`load_dotenv()`).
- `.env` is listed in `.gitignore` and **never committed**.
- If the key is missing when online mode is requested, log an error and fall back to offline.

---

## 6. Runtime Config Access Pattern

All tools access config via a singleton `ConfigManager` instance:

```python
from tools.config_manager import config

# Read
hotkey = config.get("hotkey_toggle")          # "ctrl+alt+space"
model  = config.get("offline_engine.model")   # "medium"  (dot-path access)

# Write (persists to disk)
config.set("hotkey_toggle", "ctrl+shift+space")
```

Dot-path access (`"offline_engine.model"`) is supported for nested keys.

---

## 7. Config Validation Rules

| Key | Type | Valid values |
|---|---|---|
| `hotkey_toggle` | `str` | Any valid `keyboard` library combo string |
| `start_delay_ms` | `int` | 0 – 5 000 |
| `language` | `str` | `"en"` (only value in Phase 2/3) |
| `engine_mode` | `str` | `"offline"`, `"online"`, `"auto"` |
| `offline_engine.model` | `str` | `"tiny"`, `"base"`, `"small"`, `"medium"`, `"large-v2"` |
| `offline_engine.device` | `str` | `"cuda"`, `"cpu"` |
| `offline_engine.compute_type` | `str` | `"float16"`, `"int8"`, `"float32"` |
| `commit_mode` | `str` | `"instant_paste"`, `"preview"` (preview in Phase 4) |
| `paste.paste_delay_ms` | `int` | 50 – 1 000 |

If a value fails validation, log a warning and replace with the default.

---

## 8. `.gitignore` Entries (required)

```
.env
.tmp/
__pycache__/
*.pyc
config.json
```

> `config.json` may contain user-specific paths or device names; keeping it out of source control avoids conflicts. A `config.example.json` with all defaults is committed instead.

---

## 9. Tool File

| Tool | Path |
|---|---|
| Config manager | `tools/config_manager.py` |

---

## 10. Linked SOPs

- All other SOPs depend on config values loaded by this subsystem.
