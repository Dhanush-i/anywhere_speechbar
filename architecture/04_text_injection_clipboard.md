# SOP-04: Text Injection via Clipboard

> **Layer 1 – Architecture SOP**  
> Subsystem: Text injection into the currently focused application  
> Golden Rule: Update this SOP **before** changing any text injection logic in `tools/`.

---

## 1. Purpose

This SOP defines how anywhere-speechbar injects the final transcript into the currently focused application window on Windows 11. It governs the clipboard swap strategy, paste keystroke, clipboard restoration, and timing constraints.

---

## 2. Injection Method: Clipboard Swap + Ctrl+V

The most reliable cross-application text injection method on Windows 11:

```
1. Save user's current clipboard contents
2. Write transcript text to clipboard (pyperclip.copy)
3. Wait paste_delay_ms (default 100 ms) for clipboard to settle
4. Send Ctrl+V keystroke to active window (keyboard.send)
5. Wait paste_delay_ms again for paste to complete
6. Restore original clipboard contents (pyperclip.copy)
```

### Why clipboard injection?

| Method | Reliability | Works in browsers | Works in admin windows | Notes |
|---|---|---|---|---|
| Clipboard + Ctrl+V ✅ | High | ✅ | ✅ (if app allows paste) | Chosen method |
| Synthetic keystrokes | Medium | ✅ | ❌ | Slow; encoding issues |
| Win32 SendMessage | Medium | ❌ | Partially | Requires HWND |
| UI Automation | Low | ❌ | ❌ | Complex; fragile |

---

## 3. Implementation Rules

1. **Pre-paste snapshot**: use `pyperclip.paste()` to capture the existing clipboard before writing to it.
2. **Unicode safety**: `pyperclip` on Windows defaults to the `win32` backend which handles Unicode correctly. No additional encoding steps are needed.
3. **Ctrl+V key send**: use `keyboard.send("ctrl+v")` — this is non-blocking on Windows.
4. **Restore delay**: wait at least `paste_delay_ms` milliseconds **after** the Ctrl+V before restoring the clipboard, to ensure the paste has completed before overwriting.
5. **Clipboard restore**: always restore, even if an exception occurs during paste. Use a `try/finally` block.
6. **Admin window limitation**: if the focused window is an elevated (admin) process and the Python process is not elevated, the Ctrl+V may be silently ignored. Log a warning if this is suspected (no clean way to detect); do not crash.

---

## 4. Pseudocode

```python
def inject_text(text: str, paste_delay_ms: int = 100) -> None:
    original_clipboard = pyperclip.paste()
    try:
        pyperclip.copy(text)
        time.sleep(paste_delay_ms / 1000)
        keyboard.send("ctrl+v")
        time.sleep(paste_delay_ms / 1000)
    finally:
        pyperclip.copy(original_clipboard)
```

---

## 5. Post-Processing Integration

Before calling `inject_text`, the transcript passes through `postprocess_text.py`:

| Step | Purpose |
|---|---|
| Capitalize first word | Ensures sentence starts correctly |
| Capitalize "I" | Grammar: standalone `i` → `I` |
| Normalize spaces | Remove double-spaces from STT artifacts |
| Fix end punctuation | Append `.` if no sentence-ending punctuation at end |
| Indian accent corrections | Apply static correction map (common mishears) |
| (Optional) Spell check | Lightweight pyspellchecker or language model pass |

---

## 6. Common Indian Accent Correction Map (initial set)

The following static replacements are applied after STT (case-insensitive match, exact word boundaries):

| Raw output | Corrected |
|---|---|
| `wont` | `won't` |
| `cant` | `can't` |
| `dont` | `don't` |
| `isnt` | `isn't` |
| `wasnt` | `wasn't` |
| `arent` | `aren't` |
| `didnt` | `didn't` |
| `couldnt` | `couldn't` |
| `shouldnt` | `shouldn't` |
| `wouldnt` | `wouldn't` |
| `im` | `I'm` |
| `ive` | `I've` |
| `id` | `I'd` |
| `itll` | `it'll` |

> This list is expanded in `tools/postprocess_text.py`. New entries must be documented here first.

---

## 7. Error Handling

| Error | Behaviour |
|---|---|
| `pyperclip` clipboard read failure | Log warning; proceed with empty `original_clipboard`; restore with empty string |
| `keyboard.send` failure | Log error; return to IDLE; clipboard restoration still attempted in finally block |
| Empty transcript after post-processing | Skip injection entirely; return to IDLE silently |

---

## 8. Tool File

| Tool | Path |
|---|---|
| Text injection | `tools/inject_text_clipboard.py` |
| Post-processing | `tools/postprocess_text.py` |

---

## 9. Linked SOPs

- [SOP-03: Transcription](03_transcription_offline_faster_whisper.md) — provides cleaned transcript
- [SOP-01: Hotkey & State Machine](01_hotkey_and_state_machine.md) — calls injection after TRANSCRIBING
