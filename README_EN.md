# Talk to Fengge

[中文版](README.md)

**Real-time voice conversation + voice cloning + persona injection. Engineering latency < 1 second.**

Have a real-time voice chat with the AI clone of Fengge (峰哥亡命天涯), a Chinese content creator with 1M+ followers on Bilibili. It's not text-to-speech — it's like a real phone call, with his cloned voice and personality.

Fengge is the first fully working example of this architecture. It supports swapping to other people — you'll need voice samples and a personality description. See the "[Swap to Another Voice and Personality](#swap-to-another-voice-and-personality)" section below.

> [Demo video (56K+ views)](https://x.com/leaf_sanren/status/2069342335268507976)

---

## What Makes This Different?

There are many voice cloning projects and many real-time voice AI projects. But they're usually separate:

- **Real-time conversation** (e.g., GPT-4o Voice) → no custom voice cloning
- **Voice cloning** (e.g., Bark, XTTS) → text-to-speech only, no real-time conversation

This project combines three things:

1. **Voice cloning** — clone any voice from 15-45 seconds of audio
2. **Persona injection** — speech patterns, catchphrases, thinking style — not just the voice, but the personality too
3. **Real-time conversation** — like a phone call, engineering latency under 1 second

## Tech Stack

```
User speaks → STT (speech recognition) → LLM → TTS (voice clone synthesis) → User hears reply
                                           ↑
                                   Persona injection + Memory recall (optional)
```

| Module | Default | Notes |
|--------|---------|-------|
| Real-time audio | [LiveKit](https://livekit.io/) | WebRTC framework for browser-agent audio streaming |
| STT | [Cartesia ink-whisper](https://www.cartesia.ai/ink/) (recommended) | Free tier available, good Chinese support, low latency |
| LLM | [MiniMax-M2.7-highspeed](https://www.minimaxi.com/) (recommended) | Chinese model, lowest TTFB, no VPN needed in China |
| TTS + Voice Cloning | [VoxCPM](https://github.com/openbmb/VoxCPM) (recommended) | Open-source voice cloning, best quality, needs GPU (cloud or local) |
| Persona | Based on [Nuwa Skill](https://github.com/alchaincyf/nuwa-skill) distillation + [Fengge Skill](https://github.com/YixiaJack/feng-ge-skill) + livestream transcripts | Speech style + catchphrases + thinking patterns |
| Memory | [OpenViking](https://github.com/nicepkg/openviking) (optional) | Conversation memory |

### Alternatives

Switch with one line in `.env.local`:

- **STT**: Cartesia ink-whisper (recommended) / Deepgram nova-2
- **LLM**: MiniMax (recommended) / DeepSeek / Gemini
- **TTS**: VoxCPM (recommended, open-source, best cloning quality) / [MOSS-TTS](https://github.com/open-moss/moss-tts-nano) (CPU fallback) / Cartesia Sonic (cloud, $5/mo Pro for cloning) / MiniMax TTS

## Quick Start

### Easiest Way: Let an AI Coding Assistant Help

```bash
git clone https://github.com/YeJe-cpu/talk-to-fengge.git
cd talk-to-fengge
```

Open this project in any AI coding assistant — [Claude Code](https://claude.ai/code), [Cursor](https://cursor.com/), [Codex](https://openai.com/codex), [Windsurf](https://codeium.com/windsurf), or whatever you use — and ask it to help you set up and run the project.

### Manual Setup

<details>
<summary>Expand manual setup steps</summary>

#### 1. Prerequisites

- Python 3.12+
- [uv](https://docs.astral.sh/uv/) (recommended) or pip
- [LiveKit Server](https://docs.livekit.io/home/self-hosting/local/)

#### 2. Install

```bash
# Install Python dependencies
uv sync

# Install LiveKit (macOS)
brew install livekit
```

#### 3. Configure

```bash
cp .env.example .env.local
# Edit .env.local and fill in your API keys
```

You'll need at minimum:
- **Cartesia API Key** (STT, [free signup](https://www.cartesia.ai/))
- **MiniMax API Key** (LLM, [signup](https://www.minimaxi.com/))
- **TTS solution** (see below)

#### 4. TTS Voice Cloning Options

**Option A: VoxCPM (recommended, open-source, best clone quality)**

Needs a GPU (NVIDIA, >= 8GB VRAM). Best with a local GPU for lowest latency; also works with cloud GPUs (e.g., RunPod L4):

```bash
# On the GPU machine
bash runpod_setup.sh  # Install deps + download model (~10 min first time)
```

**Option B: MOSS-TTS (CPU fallback)**

No GPU needed, but lower cloning quality and speed. Set `TTS_PROVIDER=moss` in `.env.local`.

**Option C: Cartesia Sonic (cloud, no GPU)**

Requires Cartesia Pro subscription ($5/mo) for voice cloning. Set `TTS_PROVIDER=cartesia` in `.env.local`.

#### 5. Launch

```bash
# Option 1: Double-click launch script (macOS)
./Talk-to-Me-V3.6.command

# Option 2: Launch components manually
livekit-server --dev --node-ip=127.0.0.1  # Terminal 1
LLM_PROVIDER=minimax python -m worker.main start  # Terminal 2
python -m worker.web_server  # Terminal 3
```

Open http://127.0.0.1:8766 to start chatting.

</details>

## Swap to Another Voice and Personality

Fengge is the built-in example. To swap to someone else, you need two things:

### 1. Voice Samples

Record 15-45 seconds of clear speech (no background music, no noise) and clone the voice with VoxCPM.

### 2. Personality Description

The easiest way: open this repo in an AI coding assistant and tell it:

> "I want to swap the persona to XXX. Here's some material about them: [paste text / links / your description]. Please reference `docs/persona-*.md` and `worker/persona.py` (the Fengge implementation) and generate a new persona config for me."

The AI assistant will use the existing Fengge persona as a template, extract key traits through conversation with you, and generate new persona code.

Richer source material (chat logs, speech transcripts, social media content, video clips) produces better results.

## Architecture

```
┌──────────────────────────────────────────────────┐
│              Browser Frontend                      │
│         HTML + LiveKit Web SDK                     │
│         Mic capture → WebRTC → Play reply          │
└─────────────────────┬────────────────────────────┘
                      │ WebRTC (audio)
                      ▼
┌──────────────────────────────────────────────────┐
│              LiveKit Server (local)                │
│         Room management / Audio routing            │
└──────┬──────────────────────────┬────────────────┘
       │                          │
       ▼                          ▼
┌──────────────────────┐  ┌──────────────────┐
│  Agent Worker (Python) │  │  Web Server       │
│                        │  │  (port 8766)      │
│  Persona + Memory → LLM│  │  Serves frontend  │
│                        │  └──────────────────┘
│  Audio → STT           │
│          ↓             │
│         LLM            │
│          ↓             │
│   TTS (VoxCPM clone)   │
│          ↓             │
│      Audio output      │
└──────────┬─────────────┘
           │ (optional)
           ▼
     ┌──────────┐
     │ OpenViking │
     │  Memory    │
     └──────────┘
```

## Known Limitations & Roadmap

- **Multi-step setup**: STT / LLM / TTS each require separate API keys or services; no one-click deploy yet
- **Actual latency varies**: Engineering latency < 1s, but real conversation feel is ~2-3s depending on network and API response times
- **Memory system limited**: OpenViking captures events and entities from the user side; in scenarios where the AI talks much more than the user, memory accumulation is limited
- **Minimal frontend**: Currently a particle-effect single page, no digital avatar yet

### Planned

- [ ] Digital avatar / animated character
- [ ] One-click deploy + automated persona distillation (auto-extract material → generate persona)
- [ ] More persona templates

**Tell me what you want most in [Issues](https://github.com/YeJe-cpu/talk-to-fengge/issues).**

## Star / Fork / PR

If you find this project interesting, please Star it.

Want to contribute code, new persona templates, or improvements? Fork + PR welcome.

Questions? Open an [Issue](https://github.com/YeJe-cpu/talk-to-fengge/issues).

Want to learn more, exchange ideas, or explore collaboration:

- [X / Twitter](https://x.com/leaf_sanren)
- [Portfolio & Contact](https://www.uncleleaf.cc/)

— **Leaf**

## Acknowledgments

- [LiveKit](https://github.com/livekit/livekit) — Real-time audio/video framework
- [VoxCPM](https://github.com/openbmb/VoxCPM) — Open-source voice cloning model (OpenBMB)
- [OpenViking](https://github.com/nicepkg/openviking) — Local memory system
- [Nuwa Skill](https://github.com/alchaincyf/nuwa-skill) — Persona distillation methodology (underlying tool for the Fengge persona)
- [Fengge Skill](https://github.com/YixiaJack/feng-ge-skill) — Fengge persona distillation output (persona foundation for this project)

## License

MIT
