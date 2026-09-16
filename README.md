# OpenBangla Keyboard — PR #475 (New UI/UX) + EN/বাং Toggle Button

> This fork's `master` contains upstream PR #475 (`new-design`) **plus** a
> TopBar/tray **EN ⇄ বাং language toggle button**, built and tested on
> **Linux Mint 22.3** (Ubuntu 24.04 Noble base, Cinnamon, X11).
> Upstream README lives in `README.adoc`.

## What was added on top of PR #475

Environment: **Linux Mint 22.3 Zena** (Ubuntu 24.04 Noble base), **Cinnamon**, X11, x86_64.

This document explains everything that was done on this machine, step by step,
so it can be reproduced or reviewed.

## 1. Removed `ibus-avro`, installed OpenBangla Keyboard 2.0.0

```bash
sudo apt remove -y ibus-avro
wget https://github.com/OpenBangla/OpenBangla-Keyboard/releases/download/2.0.0/OpenBangla-Keyboard_2.0.0-ubuntu24.04.deb -O /tmp/OpenBangla.deb
sudo apt install -y /tmp/OpenBangla.deb
```

IBus settings were reset/adjusted along the way:

```bash
gsettings set org.freedesktop.ibus.general preload-engines "['xkb:us::eng', 'OpenBangla']"
gsettings set org.freedesktop.ibus.panel show 2               # always show floating panel
gsettings set org.freedesktop.ibus.panel show-im-name true
```

TopBar autostart entry created at `~/.config/autostart/openbangla-keyboard.desktop`
so the floating TopBar survives reboots.

## 2. Built and installed PR #475 — “New UI/UX Design”

PR: `OpenBangla/OpenBangla-Keyboard#475` (open, branch `new-design` @ `1eca2da`,
6 commits: Settings dialog redesign, Layout Viewer redesign, layout JSON v3, …).

```bash
git clone --recursive https://github.com/OpenBangla/OpenBangla-Keyboard.git /tmp/openbangla-pr475
cd /tmp/openbangla-pr475
git fetch origin pull/475/head:pr475 && git checkout pr475
git submodule update --init --recursive
```

Build dependencies (same as CI, IBus-only):

```bash
sudo apt-get install -y build-essential clang cmake libibus-1.0-dev \
  libzstd-dev qtbase5-dev qtbase5-dev-tools libqt5svg5-dev pkg-config
```

**Gotcha — Rust too old:** Ubuntu Noble ships Rust **1.75**, but this branch's
`src/platform/Cargo.lock` uses lockfile **v4** (needs Cargo ≥ 1.78), so CMake
failed with `failed to parse lock file`. Fixed by installing stable Rust via rustup
(1.98) and putting `$HOME/.cargo/bin` first on `PATH`.

Configure / build / install (same flags as upstream CI, IBus backend):

```bash
export PATH="$HOME/.cargo/bin:$PATH"
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DENABLE_IBUS=ON -DCMAKE_INSTALL_PREFIX=/usr
cmake --build build -j$(nproc)
sudo cmake --install build
```

Result: version **3.0.0**, GUI at `/usr/bin/openbangla-gui`,
engine at `/usr/libexec/ibus-engine-openbangla`.

## 3. Fixed engine crash on startup (stale 2.0.0 setting)

Symptom: `ibus engine OpenBangla` timed out (`Set global engine failed`);
running the engine manually showed a Rust panic:

```
panicked at src/fixed/method.rs:109:66: called `Option::unwrap()` on a `None` value
```

Root cause: the old 2.0.0 user setting `~/.config/OpenBangla/Keyboard.conf`
still pointed `layout/path` at the on-disk `avrophonetic.json`. The new engine
treats **any file path as a fixed layout**, so the phonetic JSON failed
`FixedMethod`'s flat-map parse → `None` → `unwrap()` panic.

Fix: point the setting at the built-in id instead:

```ini
[layout]
name=Avro Phonetic
path=avro_phonetic
```

## 4. Language-switching investigation (the deep part)

Goal: switch Bangla ⇄ English with a keypress.

Findings, each verified with `xdotool`-synthesized keys and `ibus engine` reads:

1. **IBus's own hotkeys are dead on this system.** Neither `triggers`
   (`Ctrl+Space`) nor `next-engine` (`Super+Space`, then `F12`) ever fired —
   even with a fresh daemon and NumLock off. A `python-xlib` grab test proved
   XTEST events *do* trigger X grabs here, so the test method was valid; the
   IBus hotkey path itself never reacts.
2. **Cinnamon-native switching works.** Added the engine to Cinnamon's own
   sources so Cinnamon drives the switch (this is the correct integration
   for Cinnamon anyway):

   ```bash
   gsettings set org.cinnamon.desktop.input-sources sources "[('xkb', 'us'), ('ibus', 'OpenBangla')]"
   ```

3. **NumLock quirk:** with NumLock ON, Cinnamon ignores `Super+Space` (and
   `Ctrl+Space`) combos; a single key (`F12`) works in every NumLock state.
   Final binding covers both:

   ```bash
   gsettings set org.cinnamon.desktop.keybindings.wm switch-input-source "['<Super>space', 'F12']"
   ```

   (`Super+Space` was also freed from Cinnamon earlier and later restored as
   part of this binding; IBus `next-engine` was left at its default
   `Alt+Shift_L` to avoid conflicts.)
4. Note: `ibus engine OpenBangla` prints success state correctly but **exits 1**
   even when the switch completes (slow `EngineChanged` signal?), so scripts
   must check `ibus engine` output, not the exit code.

Working result: `F12` toggles EN ⇄ বাং reliably; `Win+Space` works when
NumLock is off.

## 5. EN/বাং toggle button (feature commit on top of PR #475)

Files changed: `src/frontend/TopBar.{h,cpp,ui}` (+112 lines).

- TopBar widened 260 → 314 px with a 6th button showing **বাং** (highlighted)
  when the `OpenBangla` engine is active, **EN** otherwise.
- The button polls `ibus engine` once a second (pausing 1.5 s right after a
  manual toggle so the in-flight switch isn't misread) and hides itself when
  no `ibus` helper exists (e.g. Fcitx-only builds).
- The tray menu gained a checkable **“Bangla (বাংলা)”** item wired to the
  same switch.
- Lesson learned: `QAbstractButton::setChecked()` is a **no-op on
  non-checkable buttons** — the button must be `checkable=true`, and then
  `clicked()` already carries the desired state.
- Verified end-to-end by clicking the real TopBar with `xdotool` at its
  on-screen coordinates and reading `ibus engine` both ways
  (plus `strace` confirming click → `ibus engine …` execution).

## 6. Current working config recap (Mint/Cinnamon)

- IBus preload: `['xkb:us::eng', 'OpenBangla']`, panel always shown.
- Cinnamon sources include `('ibus', 'OpenBangla')`; switch keys `Super+Space` + `F12`.
- TopBar autostarted from `/usr/bin/openbangla-gui`.
