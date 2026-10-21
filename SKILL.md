---
name: local-voice-dictation
description: Set up fully local voice dictation (press a hotkey, talk, text is typed into the focused window, including Claude Code) and local meeting transcription on a Linux desktop, using Voxtype + NVIDIA Parakeet. Use when the user wants speech-to-text, dictation, voice typing, push-to-talk, or meeting transcription that runs on their own machine.
---

# Local voice dictation + meeting transcription (Linux)

Goal: hotkey → speak → hotkey → text appears in whatever window has focus, all on-device. Optionally, a second hotkey that transcribes meetings (both your mic and the other participants) for later summarizing.

This is a playbook, not an installer. Every machine differs (CPU flags, GPU, desktop, Wayland vs X11, packages), so **survey first, decide, verify each step**. The reference scripts in `scripts/` were tested on Ubuntu 26.04 / GNOME Wayland; adapt them, don't assume.

## Ground rules

- **Local only.** Claude Code's built-in `/voice` streams audio to Anthropic's servers — it is *not* local. Don't offer it as the answer to a "local" request (mention it as the cloud alternative).
- **Don't use interactive picker forms (AskUserQuestion) once dictation works.** Dictated text is typed as keystrokes; if it lands on a picker it acts as menu keypresses and can choose answers the user never picked. Ask in plain text.
- **Anything needing sudo, the user runs** (suggest `! sudo …`). Prefer setups that need none.
- Download into an empty scratch dir and **verify checksums** (`SHA256SUMS.txt` on the Voxtype release) before installing binaries.

## 1. Survey the machine

Collect: CPU model + flags (`avx2`, `avx512f`, `avx_vnni`), RAM, GPU + VRAM (`nvidia-smi`), `$XDG_SESSION_TYPE` and `$XDG_CURRENT_DESKTOP`, distro, which of `wtype ydotool dotool xdotool wl-copy pactl parec notify-send uv` exist, whether the user is in the `input` group, and existing desktop keybindings (to avoid collisions — see §5).

Ask about GPU stability before using it; a CPU is plenty for this model and avoids driver problems entirely.

## 2. Model choice

Default: **NVIDIA Parakeet TDT 0.6B v2, int8** (`parakeet-tdt-0.6b-v2-int8`, ~630 MB, English-only, CC-BY-4.0).

- Near the top of the Open ASR Leaderboard for English, and an order of magnitude faster than the bigger leaders (Cohere Transcribe 2B, Granite Speech 2B+). For dictation, latency beats the last ~0.5% WER.
- On a modern laptop CPU it transcribes 10–20 s of speech in ~0.3–0.6 s with the model resident (~1 GB RAM).
- Multilingual user → `parakeet-tdt-0.6b-v3` (25 European languages) or Whisper `large-v3-turbo`.
- Benchmark before committing: `voxtype --model <m> transcribe --engine parakeet clip.wav` on a known clip (e.g. whisper.cpp's `samples/jfk.wav`). int8 and fp32 gave identical text; int8 uses less than half the RAM.
- Check the live leaderboard — this space moves fast.

## 3. Install Voxtype

[Voxtype](https://github.com/peteonrails/voxtype) (MIT, Rust) — daemon with hotkey/toggle control, many engines, meeting mode, post-processing hooks.

- Use a prebuilt release binary into `~/.local/bin/voxtype` (no sudo). Parakeet needs an **ONNX** build: `onnx-avx512` only if `avx512f` is present (many recent Intel laptop chips lack it), else `onnx-avx2`; `onnx-cuda-*` for NVIDIA GPU.
- `voxtype setup --download --model parakeet-tdt-0.6b-v2-int8` (downloads from models.voxtype.io; retry on transient curl errors).
- Configure with `voxtype config set <key> <value>` (validated; `voxtype config schema` lists keys). Key settings:
  ```
  engine parakeet
  parakeet.model parakeet-tdt-0.6b-v2-int8
  parakeet.model_type tdt
  hotkey.mode toggle
  audio.feedback.enabled true     # start/stop beeps
  audio.max_duration_secs 300     # default 60 is short for rambling
  output.notification.on_transcription false
  ```
- `voxtype setup systemd` installs a user service; it loads the model once and stays resident.
- `voxtype setup check` shows what's missing.

## 4. Getting text typed (the hard part)

Pick by desktop:

| Desktop | Driver | Notes |
|---|---|---|
| wlroots (Sway, Hyprland, River) | `wtype` | Works out of the box. |
| **GNOME / KDE Wayland** | **`eitype`** (libei via the RemoteDesktop portal) | `wtype` does **not** work there. No root needed; GNOME asks for permission once. |
| X11 | `xdotool` | |
| Anything | `ydotool` / `dotool` | Need `/dev/uinput` access (root daemon or udev rule + group). Use only if the above fail. |

`driver_order` can't be set via `config set`; edit `[output]` in `~/.config/voxtype/config.toml`, e.g. `driver_order = ["eitype", "clipboard"]`.

**eitype on GNOME** — PyPI ships only Python bindings (no CLI), and `cargo install` needs Rust + `libxkbcommon-dev`. `scripts/eitype` is a small CLI shim over the wheel covering exactly what Voxtype calls (`[-d MS] [-l LAYOUT] [--variant V] [-M MOD] [-k KEY] TEXT`). Set it up:

```sh
uv venv ~/.local/share/eitype-shim && uv pip install -p ~/.local/share/eitype-shim/bin/python eitype==0.2.2
install -m755 scripts/eitype ~/.local/bin/
~/.local/bin/eitype      # no args: triggers the one-time GNOME permission dialog, saves a restore token
```

Gotchas found the hard way:
- **eitype 0.2.2 wheel bug**: its bundled libxkbcommon lacks `xkb_keymap_key_get_mods_for_level` → `ImportError`. The shim `LD_PRELOAD`s the system `libxkbcommon.so.0` — **check that path** on the target distro (`/usr/lib/x86_64-linux-gnu/` on Debian/Ubuntu, `/usr/lib64/` on Fedora). Re-test after upgrading the wheel; a fixed release may not need it.
- **Long dictations lost their ending**: closing the EI connection immediately drops in-flight key events. The shim sleeps 0.25 s before `close()`. Verify with a long (20 s+) dictation.
- The first portal dialog can be cancelled/time out; just run `eitype` again. The token lives at `~/.cache/eitype/restore_token` (delete it to re-prompt).

## 5. The hotkey — avoid modifier collisions

With the built-in evdev hotkey off (`hotkey.enabled false`), bind a desktop shortcut to `voxtype record toggle` (GNOME: custom keybinding via `gsettings`, append to the existing `custom-keybindings` list — never overwrite it).

**Serious pitfall:** typing starts ~0.3 s after the stop press. If the hotkey uses modifiers (e.g. Ctrl+Alt+Space) and they're still held, the typed letters become shortcuts. On a machine with Ctrl+Alt+T = terminal, every "t" in the dictation opened a new terminal window. Other letters can switch workspaces, open dialogs, etc. Fixes, best first:

1. **Bind a single key with no modifiers** (Insert, Pause, an unused F-key, a Copilot key) — nothing is held, nothing combines.
2. Add the user to the `input` group → Voxtype's `wait_for_modifier_release` works (and hold-to-talk becomes possible). Tradeoff: any process of that user can then read all keyboard input. Needs sudo + re-login.
3. Stopgap: `output.pre_type_delay_ms 600` — delays typing so modifiers are usually released. Not a guarantee.

Hold-to-talk needs either the `input` group or a compositor that supports release bindings (Sway/Hyprland/River); GNOME custom shortcuts are press-only → use toggle mode.

On-screen overlays (`voxtype-osd*`) need layer-shell, which GNOME lacks → `osd.enabled false`, rely on beeps plus GNOME's mic indicator.

## 6. Optional: mark dictated text

So the reader (e.g. Claude) knows a message was spoken and may contain mis-heard words, `scripts/voxtype-mark` appends ` [voice-dictated]` via `output.post_process.command`. Voxtype **also runs this hook on meeting chunks**, so the script skips the marker while a meeting is recording. Since it applies in every app (Slack, email…), ask the user if they want it everywhere.

Recurring mis-hearings can be fixed with `voxtype config set text.replacements.<heard> '<meant>'`.

## 7. Meeting transcription

`voxtype config set meeting.enabled true`, restart. Voxtype records mic ("You") and system-audio loopback ("Remote") separately, transcribes in ~30 s chunks while the meeting runs, and stores text only (audio not retained by default). Works with any meeting app (Meet, Zoom, Cliq…) since it captures the speakers' output, not the app.

- Loopback needs **`parec` and `pactl`** (`pulseaudio-utils`; works on PipeWire). Without them the log says `No monitor source found, using mic only` and the other participants are silently missing.
- The monitor source is picked at meeting start → start it after headphones are connected.
- On laptop speakers the mic hears the remote side too and produces junk under "You" → recommend headphones.
- `scripts/voxtype-meeting-toggle` = start/stop with a desktop notification; bind it to a second shortcut (same modifier caveat).
- Summarize afterwards: `voxtype meeting export latest --speakers --timestamps` → have Claude summarize (decisions, action items). That sends the transcript (not audio) to the model provider — tell the user; a local LLM via Ollama is the fully-local alternative (`meeting.summary.backend local`).
- Recording others has legal/consent implications in many jurisdictions; tell the user to inform participants.

## 8. Verify (don't skip)

1. `voxtype setup check` clean; `systemctl --user status voxtype` active; log shows `Model loaded, ready`.
2. Test transcription on a known clip.
3. User dictates a short sentence into a text editor → correct text appears.
4. User dictates 20 s+ → ending intact.
5. User stops with the hotkey and keeps holding it a moment → no stray windows or shortcuts.
6. Meeting: start, play a clip through the speakers (`pw-play --volume 0.3 clip.wav`), stop → `export latest` shows it under "Remote". Delete test meetings afterwards (`voxtype meeting delete --force <id>`).

Debug: `journalctl --user -u voxtype -f` logs every transcription, post-processing, and driver used.
