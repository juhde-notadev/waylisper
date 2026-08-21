# Waylisper

Push-to-talk dictation for Wayland desktops (built for and tested on Fedora + KDE Plasma 6), using [OpenAI Whisper](https://github.com/openai/whisper) for transcription. Tap a hotkey, talk, tap it again, and your words show up typed — and optionally auto-submitted — wherever your cursor is. No cloud, no accounts, everything runs locally.

It was built to solve a specific, mundane problem: talking is faster than typing, especially for long messages, and existing dictation tools either weren't accurate enough, didn't work cleanly under Wayland, or fought the desktop environment at every step. This is the result of actually getting it to work end to end.

## What it does

- Records from any ALSA-visible microphone (built and tested using [AndroidMic](https://github.com/teamclouday/AndroidMic) — a phone's mic, routed to the desktop as an audio input)
- Transcribes with a persistent, always-loaded [faster-whisper](https://github.com/SYSTRAN/faster-whisper) daemon (falls back to [whisper.cpp](https://github.com/ggml-org/whisper.cpp) if the daemon isn't running)
- Copies the transcript to the clipboard, or — via a global hotkey — types it directly into whatever window has focus and submits it, with zero manual paste/switch-window steps
- Optionally reads Claude Code's responses back to you out loud via [Piper](https://github.com/OHF-Voice/piper1-gpl) TTS (a nice pairing if you're using this for AI-assisted work, but genuinely optional — the dictation half stands on its own)

## Architecture

```
   phone mic (AndroidMic)
          │  PipeWire / ALSA
          ▼
     [ dictate ]  ── records to WAV, press Enter to stop
          │
          ▼
  [ transcribe-daemon ]  ── faster-whisper, model loaded once, stays resident
          │  (Unix socket)
          ▼
      transcript text
          │
          ├──► clipboard (wl-copy)                 [plain `dictate`]
          │
          └──► [ dictate-autotype ] ──► ydotool ──► typed + Enter into
                                                     whatever has focus
                                                     [hotkey-triggered]
```

`dictate` is the core script — it just records, transcribes, and copies to clipboard. `dictate-autotype` wraps it in a disposable terminal window and adds the auto-type/auto-submit behavior, meant to be bound to a physical hotkey.

## Prerequisites

Linux, Wayland session — built and tested against KDE Plasma 6 / KWin specifically. The hotkey-binding step is KDE-specific; everything else (recording, transcription, typing) is desktop-agnostic and should work under any Wayland compositor. Everything else you need is covered in Installation below, starting from a bare system.

## Installation

### 0a. Base packages (Fedora)

```bash
sudo dnf install -y cmake gcc gcc-c++ make git python3-pip xterm alsa-utils wl-clipboard
```

On other distros, swap in the equivalent package names/manager — everything here is standard tooling, nothing exotic.

### 0b. Phone-as-mic: AndroidMic

This is a real prerequisite, not something these install steps can fully automate for you — it means installing a desktop-side server and pairing your phone's app to it. Follow [AndroidMic's own README](https://github.com/teamclouday/AndroidMic) for that part. Once it's running and paired, its audio source should just appear as both a PipeWire node and (per step 0c below) an ALSA device — nothing else to configure on the Waylisper side.

If you're using a different phone-mic app, or a physical USB mic, or anything else — same deal, as long as it ends up visible to `arecord -L` you're fine; AndroidMic isn't a hard requirement, it's just what this was built and tested against.

### 0c. Low-latency mic capture: go through PipeWire's ALSA plugin, not PulseAudio

This is the actual source of the low latency, so it's worth doing deliberately rather than accepting whatever `dictate` defaults to.

Any app that plays or captures audio on a modern Linux desktop is usually going through one of two paths on top of PipeWire: the **PulseAudio-compatibility layer** (`pipewire-pulse`, what `parecord`/most audio-picker dialogs use — convenient, but adds a real amount of buffering latency), or **PipeWire's native ALSA plugin** — a direct, low-latency path that behaves like talking to a real ALSA device. `arecord -D <device>` uses the latter. This is the same mechanism serious low-latency audio work (DAWs like REAPER, for instance) has relied on for years — this project just applies it to speech input instead of music production.

To find your device's ALSA name:

```bash
arecord -L
```

Look for an entry matching your mic source (for AndroidMic specifically, PipeWire registers it as a named ALSA device automatically — no manual config needed, it just shows up). If nothing obvious appears, check whether the app exposing your mic source registers its own ALSA alias, or fall back to `default` (works, just without the same latency guarantee).

Set it via the `WAYLISPER_ALSA_DEVICE` environment variable (e.g. in the `dictate-autotype` systemd/shortcut invocation, or your shell profile) — see `bin/dictate`, which defaults to `default` if unset.

### 1. Transcription: faster-whisper + the daemon

```bash
pip install --user faster-whisper
mkdir -p ~/.local/bin
cp bin/transcribe-daemon bin/transcribe-client ~/.local/bin/
chmod +x ~/.local/bin/transcribe-daemon ~/.local/bin/transcribe-client

mkdir -p ~/.config/systemd/user
cp systemd/transcribe-daemon.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now transcribe-daemon.service
```

First start downloads the model from Hugging Face (a few hundred MB to ~1.5GB depending on size) and takes ~15-20s to load; after that it's cached and loads in a couple seconds. Override the model or thread count with `WAYLISPER_MODEL` / `WAYLISPER_THREADS` env vars in the service file if the default (`large-v3-turbo`) is more than your CPU wants to carry.

### 2. (Optional) whisper.cpp fallback

Only needed if you want dictation to keep working when the daemon's down.

```bash
git clone --depth 1 https://github.com/ggml-org/whisper.cpp.git ~/src/whisper.cpp
cd ~/src/whisper.cpp
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j"$(nproc)"
bash models/download-ggml-model.sh large-v3-turbo-q8_0
```

`q8_0` is a near-lossless quantization and meaningfully faster than full precision on CPU. `bin/dictate` already points at `~/src/whisper.cpp/build/bin/whisper-cli` and this model path by default — only edit those if you put things somewhere else.

### 3. Auto-type: ydotool

Not packaged for every distro, build from source:

```bash
git clone https://github.com/ReimuNotMoe/ydotool.git
cd ydotool
cmake -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_DOCS=OFF
cmake --build build -j"$(nproc)"
sudo install -m 755 build/ydotool /usr/local/bin/ydotool
sudo install -m 755 build/ydotoold /usr/local/sbin/ydotoold
```

Edit `systemd/ydotoold.service`, replacing `GROUP_GID` with your own group's numeric GID (`getent group wheel` or similar — this lets your user talk to the daemon without needing a fresh login):

```bash
sudo cp systemd/ydotoold.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now ydotoold.service
```

### 4. Install the scripts

```bash
cp bin/dictate bin/dictate-autotype ~/.local/bin/
chmod +x ~/.local/bin/dictate ~/.local/bin/dictate-autotype
```

Set `WAYLISPER_ALSA_DEVICE` (or edit the default in `dictate`) to whatever `arecord -L` shows for your mic source.

### 5. Bind the hotkey (KDE Plasma 6)

System Settings → Shortcuts → Custom Shortcuts → Add New → Global Shortcut → Command/URL. Command: the **full path** to `dictate-autotype` (e.g. `/home/you/.local/bin/dictate-autotype`) — global shortcuts run with a restricted `PATH` that doesn't include `~/.local/bin`, so the bare command name won't resolve. Pick whatever trigger key you want.

### 6. (Optional) TTS read-back for Claude Code

```bash
cp bin/speak-response ~/.local/bin/
chmod +x ~/.local/bin/speak-response
pip install --user piper-tts
```

Download a voice — `en_US-lessac-high` is a solid natural-sounding default:

```bash
mkdir -p ~/.local/share/piper-voices
curl -sL -o ~/.local/share/piper-voices/en_US-lessac-high.onnx \
  "https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_US/lessac/high/en_US-lessac-high.onnx"
curl -sL -o ~/.local/share/piper-voices/en_US-lessac-high.onnx.json \
  "https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_US/lessac/high/en_US-lessac-high.onnx.json"
```

Browse [more voices here](https://huggingface.co/rhasspy/piper-voices) if you want a different one — `bin/speak-response` already points at this path by default, only edit `MODEL` in that script if you picked something else. Add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "~/.local/bin/speak-response", "async": true }
        ]
      }
    ]
  }
}
```

## Usage

- **Anywhere, manual**: run `dictate` in a terminal. Speak, press Enter, get the transcript on your clipboard.
- **Anywhere, hands-free**: tap your bound hotkey, speak, press Enter — the transcript types itself into whatever had focus and submits with Enter.

## Gotchas we hit building this (so you don't have to)

- **Global shortcuts run with a stripped-down `PATH`.** KWin/kglobalaccel doesn't source your shell's rc files, so `~/.local/bin` isn't there — use full paths in Custom Shortcut commands.
- **`kitty` (and probably other single-instance-by-default terminals) actively fights this pattern.** Its `--hold` flag explicitly only affects "the first window" of an instance, and it has a close-confirmation dialog that can silently block a scripted close with nobody around to click it. Plain `xterm` has none of this baggage — it closes when its child exits, full stop.
- **`wl-copy` daemonizes.** Unlike X11 clipboard tools, it forks into the background and stays alive to actually serve the clipboard data. If you wrap it in something holding a `flock` (like the overlap guard in `dictate-autotype`), it'll inherit and hold that lock open indefinitely unless you explicitly close the fd for it (`wl-copy 200>&-`).
- **Model load time matters as much as inference time** for a one-shot CLI tool. `faster-whisper` is meaningfully faster than `whisper.cpp` per-transcription, but reloading the model fresh every invocation (~15-20s) would make it slower overall — hence the persistent daemon.
- **A phone-as-mic PipeWire source can have several seconds of graph-negotiation startup latency** that a native ALSA device doesn't — direct ALSA capture sidesteps this along with `pipewire-pulse`'s extra buffering.

## Credits

Built collaboratively with [Claude Code](https://claude.com/claude-code) over one long, very caffeinated (and beer-fueled) evening of debugging Wayland desktop quirks. Standing on the shoulders of `faster-whisper`, `whisper.cpp`, `ydotool`, `Piper`, and `AndroidMic` — this repo is just the specific wiring, not those projects' work.

## License

MIT — see [LICENSE](LICENSE).
