# local-voice-dictation

A [Claude Code](https://claude.com/claude-code) skill that sets up **fully local voice dictation and meeting transcription** on a Linux desktop.

- Press a hotkey, talk, press it again: your words are typed into whatever window has focus (Claude Code, editor, browser, chat).
- A second hotkey transcribes meetings (Meet, Zoom, anything): your mic and the other participants, separately, while the call runs.
- Speech recognition runs on your machine: [Voxtype](https://github.com/peteonrails/voxtype) + NVIDIA [Parakeet TDT 0.6B v2](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v2) on the CPU, transcribing ~15 s of speech in about half a second.

## Why a skill, not an installer

Getting this right depends on the machine: CPU instruction sets, GPU, desktop environment, Wayland vs X11, which typing tool your desktop allows, which shortcuts already exist. An installer bakes in one machine's answers and breaks on the next. The skill gives Claude the decisions, the pitfalls we hit, and a verification checklist, so it can adapt the setup to your machine with you.

## Use

```sh
git clone https://github.com/ashishnitinpatil/local-voice-dictation ~/.claude/skills/local-voice-dictation
```

Then in Claude Code: *"set up local voice dictation"*.

## What's inside

- `SKILL.md`: the playbook. It covers model choice, installing Voxtype, how text gets typed on each desktop, hotkey pitfalls, meeting mode, and verification.
- `scripts/eitype`: a minimal `eitype` CLI for GNOME/KDE Wayland, built on the eitype Python bindings, with workarounds for two bugs in them.
- `scripts/voxtype-mark`: appends `[voice-dictated]` to dictations, but not to meeting transcripts.
- `scripts/voxtype-meeting-toggle`: starts or stops meeting transcription, with a desktop notification.

Tested on Ubuntu 26.04, GNOME on Wayland, with an Intel Core Ultra (AVX2) CPU.

## Credits

[Voxtype](https://github.com/peteonrails/voxtype) (MIT), [NVIDIA Parakeet](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v2) (CC-BY-4.0), [eitype](https://github.com/Adam-D-Lewis/eitype) (Apache-2.0), ONNX Runtime (MIT), [GTCRN](https://github.com/Xiaobin-Rong/gtcrn) (MIT).

## License

MIT
