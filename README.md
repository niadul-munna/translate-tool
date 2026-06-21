# Onubad — English↔Bangla translator for COSMIC

> **অনুবাদ** — fast English↔Bangla translation from your desktop panel. Hotkey, clipboard, swap, history.

![Platform](https://img.shields.io/badge/platform-COSMIC%20%7C%20GNOME-48b9c7)
![Python](https://img.shields.io/badge/python-3-3776AB?logo=python&logoColor=white)
![GTK](https://img.shields.io/badge/GTK-3-4A86CF?logo=gnome&logoColor=white)
![Tray](https://img.shields.io/badge/tray-AppIndicator%20SNI-5e81ac)
![Engine](https://img.shields.io/badge/engine-MyMemory%20%2B%20Google-f6a800)
![Tests](https://img.shields.io/badge/tests-16%20passing-3fb950)
![pip deps](https://img.shields.io/badge/pip%20deps-none-2ea043)

A lightweight English↔Bangla translator that lives in the Pop!_OS **COSMIC**
panel. Tray icon, popup window, global-hotkey clipboard translation, direction
swap with auto-detect, and local history. Works on any desktop with an SNI
status area (also GNOME with AppIndicator, KDE).

## Features

- **Tray icon** in the COSMIC panel (StatusNotifierItem via AppIndicator).
- **Popup**: type English, get Bangla; result auto-copied to clipboard.
- **Clipboard flow**: select text anywhere → translate via hotkey/menu.
- **Direction**: EN→BN default, **⇄ Swap**, or **Auto** (detects Bangla input).
- **History**: last 500 translations, click to reload, clear anytime.
- Online free engine: **MyMemory** with a **Google (unofficial)** fallback. No API key.

## Requirements

System packages (the installer handles them):

- `gir1.2-ayatanaappindicator3-0.1` — panel tray icon
- `python3-gi`, `python3-requests`, `python3-pil` — usually preinstalled
- `wl-clipboard` — Wayland clipboard (`wl-copy` / `wl-paste`)

No pip packages, no virtualenv.

## Install

```bash
./install.sh
```

This installs the tray typelib (asks for sudo), a `~/.local/bin/onubad`
launcher, an app entry, and an autostart entry.

Start immediately:

```bash
~/.local/bin/onubad &
```

The icon appears in the COSMIC panel. On next login it starts automatically.

## Global hotkey (Wayland)

Wayland apps can't grab keys directly, so bind a COSMIC custom shortcut:

1. **COSMIC Settings → Keyboard → Shortcuts → Custom → Add**
2. Command: `~/.local/bin/onubad --clipboard`
3. Bind e.g. `Super+T`.

Now: select English text → copy → `Super+T` → Bangla popup, copied back.

A second launch never opens a duplicate — it signals the running instance over a
unix socket (`$XDG_RUNTIME_DIR/onubad.sock`).

## Ubuntu / GNOME

The same code runs on Ubuntu (GNOME) — only a couple of environment details differ.

```bash
git clone git@github.com:niadul-munna/onubad.git
cd onubad
./install.sh
~/.local/bin/onubad &
```

The icon appears in the top-right tray.

- **Tray icon:** Ubuntu's GNOME ships the AppIndicator extension, so the icon
  shows out of the box. On a **vanilla** GNOME spin, enable it once:
  ```bash
  sudo apt install gnome-shell-extension-appindicator
  ```
  then turn it on in the Extensions app and log out/in.
- **Clipboard:** on Wayland (Ubuntu 22.04+ default) install `wl-clipboard`:
  ```bash
  sudo apt install wl-clipboard
  ```
  On an X11 session it falls back to `xclip` automatically. Check your session
  with `echo $XDG_SESSION_TYPE`.
- **Global hotkey:** Settings → **Keyboard** → **View and Customize Shortcuts**
  → **Custom Shortcuts** → **+**. Command:
  `~/.local/bin/onubad --clipboard`, bind e.g. `Super+T`.

## Usage

| Action | How |
|--------|-----|
| Open popup | Click panel icon → **Open**, or `onubad --open` |
| Translate clipboard | Panel **Translate clipboard**, or hotkey |
| Translate typed text | Type in input → **Translate** (or Ctrl+Enter) |
| Swap direction | **⇄ Swap** button |
| Auto-detect | **Auto** toggle (Bangla input → BN→EN) |
| Reuse past translation | Open **History**, click a row |

## Privacy

- Text is sent to the online translation provider (MyMemory / Google) — required
  for an online engine.
- History is stored **locally** in plaintext at
  `~/.local/share/onubad/history.json`. Use **Clear history** to wipe it.

## Development

```bash
python3 -m unittest discover -s tests   # run the test suite
```

Modules: `translator` (engine), `clipboard`, `history`, `ipc` (single instance),
`popup` (GTK3 UI), `tray` (AppIndicator), `main` (entry/routing).
