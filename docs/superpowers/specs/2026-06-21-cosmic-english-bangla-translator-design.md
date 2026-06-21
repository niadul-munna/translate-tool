# COSMIC English↔Bangla Tray Translator — Design Spec

**Date:** 2026-06-21
**Status:** Approved (design), pending implementation
**Target:** Pop!_OS COSMIC desktop (Wayland). Also works on any DE with an SNI status area (GNOME w/ AppIndicator, KDE).

## 1. Purpose

A lightweight desktop translator living in the COSMIC panel. Default English→Bangla, with reverse and auto-detect. Triggered by tray icon, global hotkey, or clipboard. Keeps a local history.

## 2. Constraints & Environment (verified)

- Desktop: `XDG_CURRENT_DESKTOP=COSMIC`, Wayland session.
- `cosmic-applet-status-area` present → COSMIC panel renders standard StatusNotifierItem (SNI) tray icons. A generic tray app appears in the panel.
- System Python 3 has: `gi` (PyGObject/GTK), `tkinter`, `PIL`, `requests`.
- Missing: `pystray` (install via `pip install --user pystray`). No Rust toolchain (so no native libcosmic applet).
- Clipboard: Wayland — `wl-copy` / `wl-paste` available (`xclip` also present as X11 fallback).
- `notify-send` NOT installed → desktop notifications optional; if used, degrade gracefully when absent.

## 3. Non-Goals (YAGNI for v1)

- No native Rust/libcosmic applet.
- No offline AI model (online free API chosen).
- No account/login, no cloud sync.
- No OCR / image translation, no TTS.

## 4. Architecture

Single Python package run under **system python3** (to reuse system `gi`/`requests`/`PIL`); only `pystray` added via `pip --user`.

```
translate-tool/
  translate_tool/
    __init__.py
    translator.py    # translation engine, providers, language detect
    clipboard.py     # wl-copy / wl-paste wrappers
    history.py       # JSON-backed recent translations
    ipc.py           # single-instance unix-socket server/client
    popup.py         # GTK popup window (input/output/swap/history)
    tray.py          # pystray SNI icon + menu
    main.py          # entry point, CLI args, instance routing
  tests/
    test_translator.py
    test_history.py
    test_ipc.py
  assets/icon.png
  requirements.txt
  install.sh
  README.md
```

Threading model: GTK main loop (`GLib`) runs on the main thread. `pystray` runs detached (its AppIndicator backend integrates with the GLib loop). Network/translation calls run on a worker thread; UI updates marshalled back via `GLib.idle_add`. The IPC socket server runs on a background thread and posts actions to the UI via `GLib.idle_add`.

## 5. Components

### 5.1 `translator.py`
- `translate(text, src="en", dst="bn", timeout=8) -> str`
- Provider chain (try in order, fall through on failure):
  1. **MyMemory** — `GET https://api.mymemory.translated.net/get?q=<text>&langpair=<src>|<dst>`. Free, no key, anon limit ~5000 words/day. Parse `responseData.translatedText`. Treat HTTP 200 with `responseStatus != 200` (e.g. quota) as failure.
  2. **Google unofficial** — `GET https://translate.googleapis.com/translate_a/single?client=gtx&sl=<src>&tl=<dst>&dt=t&q=<text>`. Parse nested JSON array, concat segments.
- `detect_lang(text) -> "bn" | "en"`: if any char in Bengali block U+0980–U+09FF present → `bn`, else `en`. Used by Auto mode.
- Raises `TranslationError` if all providers fail; caller shows the message.
- No provider state; pure functions + module-level provider list for testability (mock `requests`).

### 5.2 `clipboard.py`
- `paste() -> str`: run `wl-paste --no-newline` (fallback `xclip -selection clipboard -o`). Return "" on failure.
- `copy(text)`: pipe to `wl-copy` (fallback `xclip -selection clipboard`).
- Subprocess-based, short timeout, never raise to caller.

### 5.3 `history.py`
- Store: `~/.local/share/translate-tool/history.json` (XDG_DATA_HOME aware). List of `{ts, src, dst, input, output}`, newest first, capped at 500.
- API: `add(src, dst, inp, out)`, `recent(n=50) -> list`, `clear()`.
- `ts` is ISO-8601 string (caller passes time; module reads system clock via `time` at call site — acceptable here, this is runtime not workflow). Atomic write (temp file + rename).

### 5.4 `ipc.py`
- Socket path: `$XDG_RUNTIME_DIR/translate-tool.sock` (fallback `/tmp`).
- `try_connect_and_send(action) -> bool`: connect to existing socket, send action string (`open` | `clipboard`), return True if delivered (instance already running). False if no server.
- `serve(handler)`: bind socket (unlink stale), listen on background thread, call `handler(action)` per message. Cleanup on exit.

### 5.5 `popup.py`
- GTK `Window` (single reused instance, hidden not destroyed on close).
- Widgets: language label + **⇄ swap** button + **Auto** toggle; multiline input `TextView`; **Translate** button (also Ctrl+Enter); output `TextView` (read-only, selectable); **From clipboard**, **Copy result**, **Clear history** buttons; **history** list (`ListBox` in a `ScrolledWindow`/expander) — click row loads input+output.
- `show_and_translate_clipboard()`: pull clipboard, set input, translate.
- Direction state: `en→bn` default; swap flips; Auto runs `detect_lang` on input and sets src/dst (dst = the other of {en,bn}).
- Translate action runs on worker thread → `GLib.idle_add` to fill output + append history + refresh history list.

### 5.6 `tray.py`
- `pystray.Icon` with `assets/icon.png` (PIL-loaded; generated fallback if asset missing).
- Menu: **Open**, **Translate clipboard**, **Quit**. Actions call into popup via `GLib.idle_add`.
- Runs detached so GTK loop owns the main thread.

### 5.7 `main.py`
- Args: `--open`, `--clipboard` (default: start tray). 
- On startup: if `--open`/`--clipboard` and `ipc.try_connect_and_send(...)` succeeds → existing instance handled it, exit 0.
- Else start full app: build popup (hidden), start IPC server (`open`/`clipboard` → popup actions), start tray, run GTK main loop. If launched with an action and no prior instance, perform it after init.

## 6. Data Flow

```
Hotkey/CLI: translate-tool --clipboard
  └─ instance running? ──yes──> IPC send "clipboard" ──> running app: paste → translate → popup
                        ──no───> start app, then paste → translate → popup

Tray left-click ──> popup.show()
Tray "Translate clipboard" ──> popup.show_and_translate_clipboard()
Popup Translate ──> worker: translate() ──> idle_add: output + history.add() + refresh list
Popup history row click ──> load input/output
```

## 7. Global Hotkey (Wayland)

Wayland apps cannot grab global keys. Pattern:
1. App exposes `--clipboard` / `--open` CLI actions backed by single-instance IPC.
2. User binds a key in **COSMIC Settings → Keyboard → Shortcuts → Custom** to `translate-tool --clipboard`.
3. `install.sh` prints these instructions and the exact command path. (Auto-writing COSMIC's shortcut config is best-effort/optional; COSMIC config format may change — manual binding is the documented, reliable path.)

## 8. Error Handling

- All providers fail → output area shows: `⚠ Translation failed: <reason>. Check internet.`
- Empty input → no-op.
- MyMemory quota (responseStatus != 200) → silently fall through to Google provider.
- Clipboard empty/unavailable → status message, no crash.
- Missing `pystray` → `main.py` prints install hint and exits non-zero.
- History file corrupt/unreadable → start empty, log warning, don't crash.

## 9. Installation

`install.sh`:
1. `pip install --user pystray` (+ `requests`/`Pillow` only if missing).
2. Install launcher `.desktop` → `~/.local/share/applications/translate-tool.desktop`.
3. Install autostart `.desktop` → `~/.config/autostart/translate-tool.desktop` (runs tray on login).
4. Symlink/wrapper `translate-tool` → `~/.local/bin/translate-tool` invoking `python3 -m translate_tool.main "$@"`.
5. Print COSMIC hotkey-binding instructions.

`requirements.txt`: `pystray`, `requests`, `Pillow` (PyGObject from system).

## 10. Testing

- **`test_translator.py`** (TDD, primary): mock `requests` —
  - MyMemory success parses `translatedText`.
  - MyMemory quota (responseStatus 403/429) → falls through to Google.
  - Both fail → raises `TranslationError`.
  - `detect_lang`: Bengali string → `bn`; ASCII → `en`; mixed → `bn`.
- **`test_history.py`**: add/recent ordering, cap at 500, clear, corrupt-file recovery (temp dir).
- **`test_ipc.py`**: server up → client send returns True and handler receives action; no server → client returns False.
- **Manual**: icon shows in COSMIC panel; left-click popup; clipboard flow; swap; Auto on Bangla input; hotkey after binding; history persists across restart.

## 11. Build Order

1. `translator.py` + tests (TDD) — core value, no UI.
2. `clipboard.py`, `history.py` (+ tests).
3. `ipc.py` (+ test).
4. `popup.py` GTK.
5. `tray.py` + `main.py` wiring.
6. `assets/icon.png`, `install.sh`, `README.md`.
7. Install + manual verification on COSMIC.
