# SOP-05: UI Overlay — Pill (PyQt6)

> **Layer 1 – Architecture SOP**  
> Subsystem: Always-on-top bottom-center pill overlay UI  
> Golden Rule: Update this SOP **before** changing any UI logic in `tools/`.

---

## 1. Purpose

This SOP defines the visual overlay ("pill") that appears during audio capture and transcription. It governs window properties, position, appearance, state-driven content, and lifecycle.

---

## 2. Visual Specification

| Property | Value |
|---|---|
| Shape | Rounded rectangle ("pill") |
| Position | Bottom-center of the primary display |
| Bottom margin | 40 px from the bottom edge of the screen |
| Width | 260 px |
| Height | 52 px |
| Always-on-top | ✅ Yes |
| Draggable | ❌ No (BR-04) |
| Frame / title bar | ❌ None (frameless window) |
| Background | Semi-transparent dark (`rgba(30, 30, 30, 0.92)`) |
| Text color | White (`#FFFFFF`) |
| Font | System sans-serif, 13 px |
| Border radius | 26 px (full pill) |

---

## 3. State-Driven Content

| App State | Pill visible | Pill text | Indicator |
|---|---|---|---|
| `IDLE` | ❌ Hidden | — | — |
| `WAITING` | ✅ Shown | "Starting…" | Static dot |
| `LISTENING` | ✅ Shown | "Listening…" | Pulsing red dot 🔴 |
| `TRANSCRIBING` | ✅ Shown | "Transcribing…" | Spinner or ellipsis animation |
| `INJECTING` | ✅ Shown briefly | "Pasting…" | — |

The overlay is **hidden by default** (no tray icon or visible element when idle).

---

## 4. PyQt6 Implementation Rules

1. **Window flags**: `Qt.WindowType.FramelessWindowHint | Qt.WindowType.WindowStaysOnTopHint | Qt.WindowType.Tool`  
   - `Tool` prevents the window from appearing in the taskbar.
2. **Transparency**: `setAttribute(Qt.WidgetAttribute.WA_TranslucentBackground, True)` and paint the rounded rect in `paintEvent`.
3. **Position calculation**:
   ```python
   screen = QApplication.primaryScreen().geometry()
   x = (screen.width() - PILL_WIDTH) // 2
   y = screen.height() - PILL_HEIGHT - BOTTOM_MARGIN
   self.setGeometry(x, y, PILL_WIDTH, PILL_HEIGHT)
   ```
4. **Thread safety**: The overlay widget **must only be updated from the Qt main thread**. Use `QMetaObject.invokeMethod` or Qt signals/slots to post state updates from background threads.
5. **Pulsing dot**: Implemented with a `QTimer` that toggles dot opacity every 500 ms during `LISTENING` state.
6. **Show/hide**: Call `overlay.show()` on entering WAITING; call `overlay.hide()` after INJECTING or on error.

---

## 5. Window Lifecycle

```
App starts
    │
    ▼
Overlay widget created (hidden)
    │
    ├─► hotkey → WAITING  → overlay.show(); setText("Starting…")
    │
    ├─► LISTENING         → setText("Listening…"); start pulse timer
    │
    ├─► TRANSCRIBING      → setText("Transcribing…"); stop pulse timer; start spinner
    │
    ├─► INJECTING         → setText("Pasting…"); stop spinner
    │
    └─► IDLE              → overlay.hide()
```

---

## 6. Multi-Monitor Note

- The overlay always appears on the **primary display** (`QApplication.primaryScreen()`).
- Multi-monitor support (follow active window to correct screen) is a Phase 4 enhancement.

---

## 7. Accessibility

- The overlay window has no interactive controls and does not accept keyboard focus (`setFocusPolicy(Qt.FocusPolicy.NoFocus)`).
- This ensures the hotkey does not accidentally lose focus in the target application while the overlay is visible.

---

## 8. Tool File

| Tool | Path |
|---|---|
| UI overlay widget | `tools/ui_overlay.py` |

---

## 9. Linked SOPs

- [SOP-01: Hotkey & State Machine](01_hotkey_and_state_machine.md) — drives state transitions that update the overlay
