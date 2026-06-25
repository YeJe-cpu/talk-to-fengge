# Talk to 峰哥

[English](README_EN.md)

**实时语音对话 + 音色克隆 + 人格注入，工程延迟 < 1 秒。**

和 B 站百万粉丝博主「峰哥亡命天涯」的 AI 分身实时语音聊天。不是文字转语音——是真的像打电话一样聊天，声音和性格都是峰哥的。

峰哥是这套架构的第一个完整示例。架构上支持替换成其他人——你需要准备语音素材和人格描述，具体见下方「[换成其他人的声音和性格](#换成其他人的声音和性格)」章节。

> [Demo 视频（5.6 万+ 围观）](https://x.com/leaf_sanren/status/2069342335268507976)

---

## 这个项目有什么不同？

市面上有很多语音克隆项目，也有很多实时语音对话项目。但它们通常是割裂的：

- **能实时对话的**（如 GPT-4o Voice）→ 不支持自定义音色克隆
- **能克隆音色的**（如 Bark、XTTS）→ 只能文本转语音，不能实时对话

这个项目把三件事合在了一起：

1. **音色克隆**——用 15-45 秒的语音素材克隆任何人的声音
2. **人格注入**——说话风格、口头禅、思维方式，不只是声音像，性格也像
3. **实时对话**——像打电话一样聊天，工程链路延迟压到 1 秒以内

## 技术栈

```
用户说话 → STT（语音识别）→ LLM（大语言模型）→ TTS（音色克隆合成）→ 用户听到回复
                                  ↑
                          人格注入 + 记忆召回（可选）
```

| 模块 | 默认方案 | 说明 |
|------|---------|------|
| 实时音视频 | [LiveKit](https://livekit.io/) | WebRTC 框架，处理浏览器与 Agent 之间的音频流 |
| 语音识别 (STT) | [Cartesia ink-whisper](https://www.cartesia.ai/ink/)（推荐） | 免费层可用，中文效果好，延迟低 |
| 大语言模型 (LLM) | [MiniMax-M2.7-highspeed](https://www.minimaxi.com/)（推荐） | 国产模型，TTFB 极低，无需 VPN |
| 语音合成 + 音色克隆 (TTS) | [VoxCPM](https://github.com/openbmb/VoxCPM)（推荐） | 开源音色克隆，效果最好，需 GPU（云或本地） |
| 人格系统 | 基于 [女娲 Skill](https://github.com/alchaincyf/nuwa-skill) 蒸馏 + [峰哥 Skill](https://github.com/YixiaJack/feng-ge-skill) + 直播语料补充 | 说话风格 + 口头禅 + 思维方式 |
| 记忆系统 | [OpenViking](https://github.com/nicepkg/openviking)（可选） | 对话记忆沉淀与召回 |

### 备选方案

通过 `.env.local` 一行切换：

- **STT**：Cartesia ink-whisper（推荐）/ Deepgram nova-2
- **LLM**：MiniMax（推荐，国产无需 VPN）/ DeepSeek / Gemini
- **TTS**：VoxCPM（推荐，开源，克隆效果最好）/ [MOSS-TTS](https://github.com/open-moss/moss-tts-nano)（CPU 可跑，兜底方案）/ Cartesia Sonic（云端，需 $5/月 Pro 订阅）/ MiniMax TTS

## 快速开始

### 最简单的方式：让 AI 编程助手帮你配

```bash
git clone https://github.com/YeJe-cpu/talk-to-fengge.git
cd talk-to-fengge
```

然后把这个项目扔给任何 AI 编程助手——[Claude Code](https://claude.ai/code)、[Cursor](https://cursor.com/)、[Codex](https://openai.com/codex)、[Windsurf](https://codeium.com/windsurf)，或者你用的其他工具都行。告诉它「帮我配置并启动这个项目」，Agent 会读取 `.env.example`，引导你填 API key、装依赖、启动服务。

### 手动配置

<details>
<summary>展开手动配置步骤</summary>

#### 1. 前置依赖

- Python 3.12+
- [uv](https://docs.astral.sh/uv/)（推荐）或 pip
- [LiveKit Server](https://docs.livekit.io/home/self-hosting/local/)

#### 2. 安装

```bash
# 安装 Python 依赖
uv sync

# 安装 LiveKit（macOS）
brew install livekit
```

#### 3. 配置

```bash
cp .env.example .env.local
# 编辑 .env.local，填入你的 API key
```

你至少需要：
- **Cartesia API Key**（STT，[免费注册](https://www.cartesia.ai/)）
- **MiniMax API Key**（LLM，[注册](https://www.minimaxi.com/)，国产，无需 VPN）
- **TTS 方案**（见下方）

#### 4. TTS 音色克隆方案

**方案 A：VoxCPM（推荐，开源免费，克隆效果最好）**

需要 GPU（NVIDIA，显存 >= 8GB）。本机有 GPU 直接本地跑效果最好、延迟最低；也可以用云 GPU（如 RunPod L4）：

```bash
# 在 GPU 机器上
bash runpod_setup.sh  # 安装依赖 + 下载模型（首次约 10 分钟）
```

**方案 B：MOSS-TTS（CPU 可跑，兜底方案）**

不需要 GPU，但音色克隆效果和速度不如 VoxCPM。在 `.env.local` 设置 `TTS_PROVIDER=moss`。

**方案 C：Cartesia Sonic（云端，无需 GPU）**

需要 Cartesia Pro 订阅（$5/月）才能克隆音色。在 `.env.local` 设置 `TTS_PROVIDER=cartesia`。

#### 5. 启动

```bash
# 方式一：双击启动脚本（macOS）
./Talk-to-Me-V3.6.command

# 方式二：手动启动各组件
livekit-server --dev --node-ip=127.0.0.1  # 终端 1
LLM_PROVIDER=minimax python -m worker.main start  # 终端 2
python -m worker.web_server  # 终端 3
```

打开 http://127.0.0.1:8766 开始聊天。

</details>

## 换成其他人的声音和性格

峰哥是内置的完整示例。如果你想换成其他人，需要准备两样东西：

### 1. 声音素材

录一段 15-45 秒的清晰人声（无背景音乐、无噪音），用 VoxCPM 克隆音色。

### 2. 人格描述

最省事的方式：把这个 repo 用 AI 编程助手打开，告诉它：

> 「我想把人格换成 XXX。这是 ta 的一些素材：[粘贴文字 / 链接 / 你对 ta 的描述]。请参考 `docs/persona-*.md` 和 `worker/persona.py` 里峰哥的写法，帮我生成新的人格配置。」

AI 助手会读现有的峰哥人格作为模板，通过跟你对话提取关键特征，生成新的 persona 代码。

如果你有更丰富的素材（聊天记录、语音转文字、社交媒体内容、直播切片），效果会更好。

## 架构

```
┌──────────────────────────────────────────────────┐
│                  浏览器前端                         │
│         HTML + LiveKit Web SDK                    │
│         麦克风采集 → WebRTC → 播放回复               │
└─────────────────────┬────────────────────────────┘
                      │ WebRTC (audio)
                      ▼
┌──────────────────────────────────────────────────┐
│              LiveKit Server（本地）                 │
│         房间管理 / 音频流转发                        │
└──────┬──────────────────────────┬────────────────┘
       │                          │
       ▼                          ▼
┌──────────────────────┐  ┌──────────────────┐
│  Agent Worker (Python) │  │  Web Server       │
│                        │  │  (端口 8766)       │
│  人格 + 记忆 → LLM     │  │  提供前端页面       │
│                        │  └──────────────────┘
│  音频 → STT            │
│          ↓             │
│         LLM            │
│          ↓             │
│   TTS（VoxCPM 音色克隆）│
│          ↓             │
│        音频输出         │
└──────────┬─────────────┘
           │ (可选)
           ▼
     ┌──────────┐
     │ OpenViking │
     │ 记忆服务    │
     └──────────┘
```

## 已知不足 & 后续方向

- **部署环节多**：STT / LLM / TTS 各需要不同的 API key 或服务，尚没有一键部署方案
- **实际延迟受网络影响**：工程链路延迟 < 1 秒，但实际对话体感约 2~3 秒，受网络环境和 API 响应速度影响
- **记忆系统受限**：OpenViking 主要识别用户侧的事件和主体来沉淀记忆，但在峰哥场景下 AI 话多用户话少，导致可沉淀的记忆有限
- **前端简陋**：目前是星空粒子效果的单页，还没有灵动的数字人 / 虚拟形象

### 后续想做的

- [ ] 灵动数字人 / 虚拟形象接入
- [ ] 一键部署 + 自动抓取素材生成 persona（自动化人格蒸馏）
- [ ] 更多克隆人格模板

**欢迎在 [Issues](https://github.com/YeJe-cpu/talk-to-fengge/issues) 告诉我你最想要哪个功能。**

## Star / Fork / PR

如果你觉得这个项目有意思，请给个 Star。

想贡献代码、新的人格模板、或者改进建议？欢迎 Fork + PR。

有问题？开 [Issue](https://github.com/YeJe-cpu/talk-to-fengge/issues) 聊。

想了解更多、交流想法、或者聊合作：

- [X / Twitter](https://x.com/leaf_sanren)
- [作品集 & 联系方式](https://www.uncleleaf.cc/)

— **Leaf**

## 致谢

- [LiveKit](https://github.com/livekit/livekit) — 实时音视频框架
- [VoxCPM](https://github.com/openbmb/VoxCPM) — 开源音色克隆模型（OpenBMB）
- [OpenViking](https://github.com/nicepkg/openviking) — 本地记忆系统
- [女娲 Skill](https://github.com/alchaincyf/nuwa-skill) — 人格蒸馏方法论（峰哥人格的底层蒸馏工具）
- [峰哥 Skill](https://github.com/YixiaJack/feng-ge-skill) — 峰哥人格蒸馏产出（本项目的人格基础）

## License

MIT
