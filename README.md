# local-voice-dictation

A playbook for setting up **fully local voice dictation and meeting transcription** on a Linux desktop.

- **Dictation:** press a hotkey, talk, press it again, and your words are typed into whatever window has focus: a terminal, an AI coding agent, an editor, a browser, chat.
- **Meetings:** a second hotkey transcribes a meeting in Meet, Zoom or anything else, capturing your mic and the other participants separately while the call runs.
- **All on your machine:** [Voxtype](https://github.com/peteonrails/voxtype) runs NVIDIA [Parakeet TDT 0.6B v2](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v2) on the CPU, transcribing ~15 s of speech in about half a second.

## How to use this

This is a playbook, not an installer. Every machine differs (CPU flags, GPU, desktop environment, Wayland vs X11, which typing tools your desktop allows, which shortcuts already exist), and an installer that bakes in one machine's answers breaks on the next. Follow it yourself, or hand it to an AI coding agent:

> Read https://github.com/ashishnitinpatil/local-voice-dictation and set this up on my machine.

It was worked out and tested on Ubuntu 26.04, GNOME on Wayland, with an Intel Core Ultra (AVX2) CPU. The scripts in [`scripts/`](scripts/) are the tested versions from that machine. Use them as references and adapt them; don't assume they fit as-is.

---

# The playbook

## Ground rules

- **Survey first, decide, then verify each step.** Don't run things blind.
- **Keep it local.** Claude Code's built-in `/voice` and most hosted dictation apps stream audio to a server. They're fine, but they're not this.
- **Prefer setups that need no root.** Anything that needs `sudo`, the user runs themselves.
- **Verify downloaded binaries** against the release's `SHA256SUMS.txt`, after downloading into an empty scratch directory.
- **Agents: once dictation works, don't use interactive picker or menu prompts.** Dictated text is typed as keystrokes; if it lands on a picker, the words act as menu keypresses and can choose answers the user never picked. Ask in plain text.

## 1. Survey the machine

Collect:
- **CPU:** model and flags (`avx2`, `avx512f`, `avx_vnni`).
- **Memory and GPU:** RAM, and GPU plus VRAM (`nvidia-smi`).
- **Desktop:** `$XDG_SESSION_TYPE`, `$XDG_CURRENT_DESKTOP`, and the distro.
- **Tools present:** which of `wtype ydotool dotool xdotool wl-copy pactl parec notify-send uv` exist.
- **Permissions:** whether the user is in the `input` group.
- **Existing keyboard shortcuts,** to avoid collisions (see §5).

A CPU is plenty for this model. Using the GPU is optional; ask about driver stability first.

## 2. Pick the model

The default is **NVIDIA Parakeet TDT 0.6B v2, int8** (`parakeet-tdt-0.6b-v2-int8`, ~630 MB, English-only, CC-BY-4.0).

- **Why this one:** it's near the top of the [Open ASR Leaderboard](https://huggingface.co/spaces/hf-audio/open_asr_leaderboard) for English, and an order of magnitude faster than the bigger leaders (Cohere Transcribe 2B, IBM Granite Speech 2B+). For dictation, speed matters more than the last ~0.5% of word error rate.
- **Speed on a laptop CPU:** 10–20 s of speech transcribes in ~0.3–0.6 s, with the model kept loaded in ~1 GB of RAM.
- **For other languages:** use `parakeet-tdt-0.6b-v3` (25 European languages) or Whisper `large-v3-turbo`.
- **Benchmark before committing:** run `voxtype --model <m> transcribe --engine parakeet clip.wav` on a clip with known text, such as whisper.cpp's `samples/jfk.wav`. int8 and full precision gave identical text, and int8 uses less than half the RAM.
- **Check the live leaderboard;** this field moves fast.
- **Check the mic input level before blaming the model.** On the test machine, the mic was set to 27% and word errors were 9%. Raising the level cut them to 6.5% with no model change, and the stronger models gained much less. Measure with `pw-record` and `ffmpeg -af volumedetect`. Aim for peaks around −10 to −6 dB, with no samples at 0 dB, which means clipping. GNOME's input slider isn't linear: going from 0.27 to 0.55 (`wpctl set-volume @DEFAULT_AUDIO_SOURCE@`) added ~19 dB and clipped, and 0.40 was right.
- **Test on the user's own voice:** have them read a ~100-word paragraph full of their real jargon and names, then compare models by word error rate. Ignore formatting-only differences such as "ten" vs "10". On real speech, Parakeet int8 matched Cohere Transcribe while being 8× faster. Whisper large-v3-turbo invented a closing "thank you".
- **Fix recurring name errors with replacements,** e.g. `voxtype config set 'text.replacements.cloud code' 'Claude Code'`. Match on phrases, not single common words like "cloud".

## 3. Install Voxtype

[Voxtype](https://github.com/peteonrails/voxtype) (MIT, Rust) is a background service: hotkey or toggle control, many speech engines, a meeting mode, and post-processing hooks.

- **Use a prebuilt release binary** in `~/.local/bin/voxtype`; no root needed. Parakeet needs an **ONNX** build: `onnx-avx512` only if `avx512f` is present (many recent Intel laptop chips lack it), otherwise `onnx-avx2`, and `onnx-cuda-*` for NVIDIA GPUs.
- **Download the model:** `voxtype setup --download --model parakeet-tdt-0.6b-v2-int8`. It comes from models.voxtype.io; retry on transient curl errors.
- **Configure** with `voxtype config set <key> <value>`, which validates keys and values (`voxtype config schema` lists them all). The key settings:
  ```
  engine parakeet
  parakeet.model parakeet-tdt-0.6b-v2-int8
  parakeet.model_type tdt
  hotkey.mode toggle
  audio.feedback.enabled true     # start/stop beeps
  audio.max_duration_secs 300     # the default 60 is short for rambling
  output.notification.on_transcription false
  ```
- **Run it as a service:** `voxtype setup systemd` installs a user service that loads the model once and keeps it loaded.
- **Check what's missing:** `voxtype setup check`.

## 4. Get the text typed (the hard part)

Pick the typing tool by desktop:

| Desktop | Tool | Notes |
|---|---|---|
| wlroots (Sway, Hyprland, River) | `wtype` | Works out of the box. |
| **GNOME / KDE on Wayland** | **`eitype`** (libei, through the RemoteDesktop portal) | `wtype` does **not** work here. No root needed; the desktop asks for permission once. |
| X11 | `xdotool` | |
| Anything | `ydotool` / `dotool` | Needs access to `/dev/uinput` (a root daemon, or a udev rule plus a group). Last resort. |

`driver_order` can't be set with `config set`. Edit `[output]` in `~/.config/voxtype/config.toml` instead, e.g. `driver_order = ["eitype", "clipboard"]`.

**eitype on GNOME:** PyPI ships only Python bindings, with no command-line tool, and `cargo install` needs Rust plus `libxkbcommon-dev`. [`scripts/eitype`](scripts/eitype) is a small command-line wrapper over the Python package. It covers exactly the commands Voxtype sends (`[-d MS] [-l LAYOUT] [--variant V] [-M MOD] [-k KEY] TEXT`). To set it up:

```sh
uv venv ~/.local/share/eitype-shim && uv pip install -p ~/.local/share/eitype-shim/bin/python eitype==0.2.2
install -m755 scripts/eitype ~/.local/bin/
~/.local/bin/eitype      # with no args: shows the one-time GNOME permission dialog and saves a restore token
```

Gotchas found the hard way:
- **The eitype 0.2.2 package has a bug:** its bundled libxkbcommon lacks `xkb_keymap_key_get_mods_for_level`, which causes an `ImportError`. The wrapper works around it by preloading (`LD_PRELOAD`) the system's `libxkbcommon.so.0`. **Check that path** on the target distro: `/usr/lib/x86_64-linux-gnu/` on Debian/Ubuntu, `/usr/lib64/` on Fedora. Re-test after upgrading the package; a fixed release may not need the workaround.
- **Long dictations lost their ending:** closing the connection immediately drops keystrokes that haven't been delivered yet. The wrapper waits 0.25 s before closing. Verify with a dictation of 20 s or more.
- **The first permission dialog can be cancelled or time out.** Just run `eitype` again. The token is saved in `~/.cache/eitype/restore_token`; delete it to be asked again.

## 5. The hotkey: avoid modifier collisions

Turn off Voxtype's built-in hotkey (`hotkey.enabled false`) and bind a desktop shortcut to `voxtype record toggle`. On GNOME, add a custom keybinding with `gsettings`, appending to the existing `custom-keybindings` list rather than overwriting it.

**Serious pitfall:** typing starts ~0.3 s after the stop press. If the hotkey uses modifiers (like Ctrl+Alt+Space) and they're still held, the typed letters become shortcuts. On the test machine, where Ctrl+Alt+T opens a terminal, every "t" in a dictation opened a new terminal window. Other letters can switch workspaces, open dialogs, and so on. Fixes, best first:

1. **Bind a single key with no modifiers**, such as Insert, Pause, or an unused F-key. Nothing is held down, so nothing combines. A **Copilot key** works well too, with one catch: it sends Super+Shift plus a key. Tools disagree on what that key is: xev (through XWayland) says `XF86Assistant`, while GNOME sees `TouchpadOff`. Don't guess the name. Let the user record it in **Settings → Keyboard → Custom Shortcuts** by pressing it; on the test machine GNOME saved `<Shift><Super>TouchpadOff`. Because it includes Super+Shift, keep a short `pre_type_delay_ms` and tap it rather than holding it.
2. **Add the user to the `input` group.** Voxtype's `wait_for_modifier_release` then works, and hold-to-talk becomes possible. The tradeoff: any program the user runs can then read all keyboard input. Needs sudo and a fresh login.
3. **Stopgap:** `output.pre_type_delay_ms 600` delays typing, so modifiers are usually released first. It's not a guarantee.

**People press stop while finishing their last word,** so the word gets cut off: on the test machine, "yeah" became "Huh?". Voxtype has no setting to keep recording after stop. Instead, bind the shortcut to [`scripts/voxtype-dictate-toggle`](scripts/voxtype-dictate-toggle), which waits 0.6 s before a *stop*. That catches the tail, and also gives the keys time to be released, so `pre_type_delay_ms` can stay at 0.

Hold-to-talk needs either the `input` group or a desktop that supports key-release bindings (Sway, Hyprland, River). GNOME custom shortcuts only fire on press, so use toggle mode there.

The on-screen overlays (`voxtype-osd*`) need layer-shell, which GNOME lacks. Set `osd.enabled false` and rely on the beeps plus GNOME's microphone indicator.

## 6. Optional: mark dictated text

[`scripts/voxtype-mark`](scripts/voxtype-mark) appends ` [voice-dictated]` to each dictation, through `output.post_process.command`. That tells the reader, such as an AI agent, that the text was spoken and may contain mis-heard words.
- **Voxtype also runs this hook on meeting transcripts,** so the script skips the marker while a meeting is recording.
- **The marker shows up in every app,** including Slack and email. Ask the user whether they want that.

Words that keep getting mis-heard can be fixed with `voxtype config set text.replacements.<heard> '<meant>'`.

## 7. Meeting transcription

Run `voxtype config set meeting.enabled true`, then restart. Voxtype records the mic ("You") and the computer's audio output ("Remote") separately, and transcribes in ~30 s chunks while the meeting runs. It stores text only; audio isn't kept by default. It works with any meeting app, because it captures the speaker output rather than the app.

- **Capturing the other participants needs `parec` and `pactl`** (the `pulseaudio-utils` package, which works on PipeWire too). Without them, the log says `No monitor source found, using mic only` and the other participants are silently missing.
- **The audio source is picked when the meeting starts,** so start it after connecting headphones.
- **On laptop speakers, the mic hears the other participants too** and produces junk under "You". Recommend headphones.
- **Use a hotkey for meetings too:** [`scripts/voxtype-meeting-toggle`](scripts/voxtype-meeting-toggle) starts or stops a meeting and shows a desktop notification. Bind it to a second shortcut; the same modifier caveat applies.
- **To summarize afterwards,** run `voxtype meeting export latest --speakers --timestamps` and hand the transcript to an AI for decisions and action items. That sends the transcript, but not the audio, to the AI provider. For fully local summaries, use a local model through Ollama (`meeting.summary.backend local`).
- **Consent:** recording other people has legal implications in many places. Tell participants.

## 8. Verify (don't skip)

1. `voxtype setup check` is clean, `systemctl --user status voxtype` is active, and the log shows `Model loaded, ready`.
2. A clip with known text transcribes correctly.
3. Dictating a short sentence into a text editor produces the right text.
4. Dictating for 20 s or more keeps the ending intact.
5. Stopping with the hotkey while holding it a moment longer causes no stray windows or shortcuts.
6. Meeting test: start a meeting, play a clip through the speakers (`pw-play --volume 0.3 clip.wav`), and stop. `export latest` should show the clip under "Remote". Delete test meetings afterwards (`voxtype meeting delete --force <id>`).

To debug, run `journalctl --user -u voxtype -f`. It logs every transcription, post-processing step, and typing tool used.

---

## Credits

[Voxtype](https://github.com/peteonrails/voxtype) (MIT), [NVIDIA Parakeet](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v2) (CC-BY-4.0), [eitype](https://github.com/Adam-D-Lewis/eitype) (Apache-2.0), ONNX Runtime (MIT), [GTCRN](https://github.com/Xiaobin-Rong/gtcrn) (MIT).

## License

MIT
