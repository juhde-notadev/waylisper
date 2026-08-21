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

Clone this repo first — every command below assumes you're running it from inside the cloned directory:

```bash
git clone https://github.com/juhde-notadev/waylisper.git
cd waylisper
```

### 0a. Base packages (Fedora)

```bash
sudo dnf install -y cmake gcc gcc-c++ make git python3-pip xterm alsa-utils wl-clipboard pipewire-alsa jq pulseaudio-utils
```

`pipewire-alsa` is the one easy to miss — it's the actual plugin that makes a PipeWire audio node (like your phone mic) show up as a named ALSA device at all. Without it, `arecord -L` won't list it and step 0c below won't have anything to point at. It's usually already pulled in by default on a full desktop install (Fedora KDE Spin includes it out of the box), but it's a distinct package worth naming explicitly rather than assuming.

`jq` and `pulseaudio-utils` (for `paplay`) are only needed for the optional TTS read-back in step 6 — skip them if you're not setting that up.

On other distros, swap in the equivalent package names/manager — everything here is standard tooling, nothing exotic.

### 0b. Phone-as-mic: AndroidMic

This is a real prerequisite, not something these install steps can fully automate for you — it means installing a desktop-side server and pairing your phone's app to it. Follow [AndroidMic's own README](https://github.com/teamclouday/AndroidMic) for that part. Once it's running and paired, its audio shows up as a PipeWire node — turning that into a nicely-named ALSA device (what step 0c actually needs) takes one more manual bit of config, covered there.

If you're using a different phone-mic app, or a physical USB mic, or anything else — same deal, as long as it ends up visible to `arecord -L` you're fine; AndroidMic isn't a hard requirement, it's just what this was built and tested against.

AndroidMic offers a few connection modes between phone and PC — the choice matters for latency:

- **WiFi, UDP** — lower latency than TCP over WiFi, since UDP skips TCP's ordering/retransmission overhead. Prefer this over TCP if you're staying wireless.
- **USB (Serial or ADB)** — wired, avoids WiFi jitter entirely, the lowest-latency option if you want to push past what WiFi UDP can give you. Requires `adb` installed and USB debugging enabled on the phone for the ADB mode specifically.

See [AndroidMic's README](https://github.com/teamclouday/AndroidMic) for the exact setup steps per mode — they differ enough (pairing over the same network vs. cable + developer mode) that it's worth reading directly rather than a paraphrase here.

### 0c. Low-latency mic capture: go through PipeWire's ALSA plugin, not PulseAudio

This is the actual source of the low latency, so it's worth doing deliberately rather than accepting whatever `dictate` defaults to.

Any app that plays or captures audio on a modern Linux desktop is usually going through one of two paths on top of PipeWire: the **PulseAudio-compatibility layer** (`pipewire-pulse`, what `parecord`/most audio-picker dialogs use — convenient, but adds a real amount of buffering latency), or **PipeWire's native ALSA plugin** — a direct, low-latency path that behaves like talking to a real ALSA device. `arecord -D <device>` uses the latter. This is the same mechanism serious low-latency audio work (DAWs like REAPER, for instance) has relied on for years — this project just applies it to speech input instead of music production.

**This part is manual, and it's the actual reason the latency is as low as it is.** A PipeWire node existing doesn't automatically get a friendly ALSA name — you define that yourself with a PCM alias.

First, find your mic's exact PipeWire node name:

```bash
wpctl status
# or: pactl list sources short
```

Look for your phone mic in the Sources/Audio section — for AndroidMic it typically shows up as something like `Android_Mic_Source`. Then add a PCM alias for it in `~/.asoundrc` (create the file if it doesn't exist):

```
pcm.android_mic {
    type pipewire
    capture_node "Android_Mic_Source"
    hint {
        show on
        description "Android Mic (via PipeWire)"
    }
}
```

Replace `"Android_Mic_Source"` with whatever node name you found, and `android_mic` (both the `pcm.NAME` and the description) with whatever you want to call it. Verify it shows up:

```bash
arecord -L | grep -A2 android_mic
```

If nothing appears, double check the exact node name (it's case-sensitive) and that PipeWire itself sees the device at all (`wpctl status` should list it as a running source before this step will work).

Set it via the `WAYLISPER_ALSA_DEVICE` environment variable (e.g. in the `dictate-autotype` systemd/shortcut invocation, or your shell profile) — see `bin/dictate`, which defaults to `default` if unset. `default` still works without any of this, just without the same latency win.

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

First start downloads the model from Hugging Face (a few hundred MB to ~1.5GB depending on size) and takes ~15-20s to load; after that it's cached and loads in a couple seconds.

If the default model (`large-v3-turbo`) is more than your CPU wants to carry, override it by adding an `Environment=` line to `~/.config/systemd/user/transcribe-daemon.service`, under `[Service]`, before enabling it:

```
Environment=WAYLISPER_MODEL=medium.en
Environment=WAYLISPER_THREADS=8
```

Then `systemctl --user daemon-reload && systemctl --user restart transcribe-daemon.service` to pick up the change.

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

**None of the hard technology here is ours.** Every actual capability — speech recognition, audio routing, virtual input, text-to-speech — comes from other people's work. What this repo contributes is specific: the wiring, the config, and the fixes for the particular ways these pieces broke against each other on a real Wayland/KDE desktop. If something here impresses you, it's almost certainly the project below it doing the real work, not the glue.

- **[Whisper](https://github.com/openai/whisper)** (OpenAI) — the actual speech recognition model doing the transcription.
- **[faster-whisper](https://github.com/SYSTRAN/faster-whisper)** (SYSTRAN) — the CTranslate2-based inference engine `transcribe-daemon` runs Whisper through.
- **[CTranslate2](https://github.com/OpenNMT/CTranslate2)** (OpenNMT) — the fast inference runtime underneath faster-whisper.
- **[whisper.cpp](https://github.com/ggml-org/whisper.cpp)** (ggml-org / Georgi Gerganov) — the CPU-optimized C++ Whisper implementation used as the offline fallback.
- **[ydotool](https://github.com/ReimuNotMoe/ydotool)** (ReimuNotMoe) — the virtual-input tool that makes auto-typing possible on Wayland.
- **[Piper](https://github.com/OHF-Voice/piper1-gpl)** (Open Home Foundation, formerly the Rhasspy project) — the text-to-speech engine behind the optional voice read-back.
- **[AndroidMic](https://github.com/teamclouday/AndroidMic)** (teamclouday) — turns a phone into a PC microphone; what this was built and tested against.
- **xterm** — the terminal `dictate-autotype` launches. Decades-old, unglamorous, and exactly why the auto-close behavior is reliable — see the gotchas above for why that's not a small thing.
- **PipeWire, ALSA, and KDE Plasma/KWin** — the audio and desktop infrastructure everything else sits on top of.

Wiring, scripting, and debugging done collaboratively with [Claude Code](https://claude.com/claude-code) over one long, very caffeinated (and beer-fueled) evening of fighting Wayland desktop quirks. Full credit for the underlying capabilities belongs to the projects above — this repo is the assembly, not the invention.

## License

MIT — see [LICENSE](LICENSE).
