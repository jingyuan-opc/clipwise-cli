---
name: clipwise-cli
description: |
  本地视频处理 CLI 工具（agent-friendly，JSON in/out）。当用户要求分析视频、生成切片、
  提取精华片段、视频转写、生成图文稿、裁剪视频、去水印、添加字幕/标题、配音、拼接视频、
  提取音频、翻译视频、解说视频、AI配音解说，或提到 "clipwise"、"切片"、"视频分析"、"短视频"、"highlight"、"narrate"、"解说"
  等关键词时使用。
  支持输入：本地视频文件路径、直接视频 URL、主流平台视频链接（B站、YouTube、小红书等）。
  本 skill 是完整参考文档，无需阅读源码即可正确使用所有工具。
---

# Clipwise — Agent 视频处理工具完整指南

所有 stdout 输出为 JSON envelope，日志走 stderr。每条命令独立执行、幂等、可缓存。

外部依赖：`ffmpeg`（必需），`edge-tts`（可选，配音用）。

---

## 1. 初始化

### 1.1 安装二进制

CLI 工具随 skill 文件一起分发。根据运行平台选择对应的二进制文件，复制/链接为 `clipwise` 并赋予执行权限：

```bash
# 本 skill 目录下包含以下平台二进制：
# clipwise-for-mac          — macOS (amd64/arm64 通用)
# clipwise-for-Linux-amd    — Linux amd64
# clipwise-for-Linux-arm    — Linux arm64
# clipwise-for-windows.exe  — Windows amd64

# macOS / Linux 示例：
cp clipwise-for-mac /usr/local/bin/clipwise   # 或放入任意 PATH 目录
chmod +x /usr/local/bin/clipwise

# 验证安装：
clipwise version
clipwise tools list
```

### 1.2 环境变量配置（.env 文件）

clipwise 启动时自动读取用户目录 `~/.clipwise/.env` 文件中的环境变量。已存在于系统环境中的变量不会被覆盖。

创建配置文件：
```bash
mkdir -p ~/.clipwise
cat > ~/.clipwise/.env << 'EOF'
# AI Provider
CLIPWISE_PROVIDER=volc
CLIPWISE_API_KEY=your-api-key
CLIPWISE_MODEL=doubao-seed-2-0-lite-260428

# 可选
CLIPWISE_BASE_URL=
EOF
```

环境变量优先级：**CLI 参数 > 系统环境变量 > ~/.clipwise/.env 文件**

### 1.3 环境变量说明

| 变量名 | 说明 | 示例 |
|--------|------|------|
| `CLIPWISE_PROVIDER` | AI 提供商：`volc`（火山引擎）/ `openai`（OpenAI 兼容）/ `claude`（本地 Claude CLI）/ `mock`（测试用） | `volc` |
| `CLIPWISE_API_KEY` | API 密钥。`claude` 和 `openai`（本地模型）可省略 | `sk-xxx` |
| `CLIPWISE_MODEL` | 模型名称。与 provider 对应（`claude` provider 必填） | `doubao-seed-2-0-lite-260428` |
| `CLIPWISE_BASE_URL` | API 地址。不设置时使用 provider 默认地址（`claude` 不适用） | `http://localhost:10000/v1` |

> **模型选择建议**：highlights 和 translate-video 场景需要 AI 模型具备强大的**视频理解能力**（多帧分析、时间推理、内容总结）。推荐：
> - **火山引擎（volc）**：`doubao-seed-2-0-lite-260428` 等 doubao-seed 系列
> - **OpenAI 兼容**：Gemini 系列模型（如 `gemini-2.5-flash`）
> - **Claude CLI（claude）**：`opus`、`sonnet` 等（需本地安装 Claude Code CLI，通过 `--model` 指定）
>
> 如果模型视频理解能力有限，可能导致分析结果质量不佳。

---

## 2. AI Provider 参数

所有需要 AI 的工具（highlights, translate-video）共享同一套 provider 参数，可通过 CLI 参数或环境变量设置：

| CLI 参数 | 环境变量 | 默认值 | 说明 |
|----------|----------|--------|------|
| `--provider` | `CLIPWISE_PROVIDER` | `volc` | AI 提供商：`volc` / `openai` / `claude` / `mock` |
| `--api-key` | `CLIPWISE_API_KEY` | — | API 密钥（`claude` 和 `openai` 本地模型可省略） |
| `--model` | `CLIPWISE_MODEL` | — | 模型名称（`mock` 可省略，`claude` 必填） |
| `--base-url` | `CLIPWISE_BASE_URL` | 由 provider 决定 | API 地址（`claude` 不适用） |

---

## 3. 工具完整参考

### 3.1 probe — 视频元数据探测

获取视频的基本属性：时长、分辨率、音轨、格式、文件大小、内容哈希。

```bash
./clipwise probe --input video.mp4
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `--input` | 是 | 输入视频路径 |

输出 `data` 字段：
```json
{
  "input": "video.mp4",
  "duration_seconds": 120.5,
  "width": 1920,
  "height": 1080,
  "has_audio": true,
  "size_bytes": 52428800,
  "format": "mov,mp4,m4a,3gp,3g2,mj2",
  "content_hash": "sha256..."
}
```

---

### 3.2 clip — 切割视频

按时间戳或 AI 分析结果切割视频片段。支持两种模式：

**Mode A（单段切割）**：指定起止时间切出一段。
```bash
./clipwise clip --input video.mp4 --start 10.0 --end 25.5 --output clip1.mp4
```

**Mode B（批量切割）**：根据 highlights 输出的 topMoments JSON 批量切片。
```bash
./clipwise clip --input video.mp4 --moments ./moments.json --output-dir ./clips \
  --name-prefix clip --transition xfade
```

| 参数 | Mode A | Mode B | 默认值 | 说明 |
|------|--------|--------|--------|------|
| `--input` | 必填 | 必填 | — | 输入视频路径 |
| `--start` | 必填 | — | `0` | 起始时间（秒） |
| `--end` | 必填 | — | `0` | 结束时间（秒） |
| `--output` | 必填 | — | — | 输出文件路径 |
| `--moments` / `--moments-json` | — | 必填 | — | topMoments JSON（文件路径/内联/stdin） |
| `--output-dir` | — | 必填 | — | 输出目录 |
| `--name-prefix` | — | 否 | `clip` | 输出文件名前缀 |
| `--transition` | — | 否 | `xfade` | 片段间过渡效果：`none` / `xfade` |

输出 `data.clips`：
```json
[
  { "momentIndex": 0, "filePath": "./clips/clip_000.mp4", "duration": 15.5 },
  { "momentIndex": 1, "filePath": "./clips/clip_001.mp4", "duration": 12.3 }
]
```

---

### 3.3 title — 叠加标题文字

在视频画面上叠加文字标题，支持自定义位置和样式。两种模式：

**Mode A（单文件）**：
```bash
./clipwise title --input clip.mp4 --text "精彩标题" --output titled.mp4
```

**Mode B（批量）**：配合 clip 的输出 + moments 的 title 字段批量加标题。
```bash
./clipwise title --clips ./clips.json --moments ./moments.json \
  --output-dir ./titled --style xhs --position top
```

| 参数 | Mode A | Mode B | 默认值 | 说明 |
|------|--------|--------|--------|------|
| `--input` | 必填 | — | — | 输入视频路径 |
| `--text` | 必填 | — | — | 标题文本 |
| `--output` | 必填 | — | — | 输出文件路径 |
| `--clips` / `--clips-json` | — | 必填 | — | clips JSON（clip 工具的输出） |
| `--moments` / `--moments-json` | — | 必填 | — | topMoments JSON（title 字段作为标题文本） |
| `--output-dir` | — | 必填 | — | 输出目录 |
| `--position` | 否 | 否 | `top` | 标题位置：`top` / `center` / `bottom` |
| `--style` | 否 | 否 | `xhs` | 标题样式预设：`xhs` / `minimal` |

---

### 3.4 delogo — 去除水印

对视频指定区域进行模糊处理以去除水印/Logo。两种模式：

**Mode A（单文件）**：
```bash
./clipwise delogo --input clip.mp4 --region 10:25:200:50 --output clean.mp4
```

**Mode B（批量）**：
```bash
./clipwise delogo --clips ./clips.json --output-dir ./cleaned
```

| 参数 | Mode A | Mode B | 说明 |
|------|--------|--------|------|
| `--input` | 必填 | — | 输入视频路径 |
| `--region` | 必填 | — | 水印区域 `x:y:w:h`（左上角坐标 + 宽高） |
| `--output` | 必填 | — | 输出文件路径 |
| `--clips` / `--clips-json` | — | 必填 | clips JSON |
| `--output-dir` | — | 必填 | 输出目录 |

> `region` 格式：`x:y:w:h`。highlights 的 AI 分析会自动检测水印位置并输出 `watermark` 字段，可直接用于批量去水印。

---

### 3.5 subtitle — 烧录字幕

将带时间戳的字幕文本烧录到视频画面上。

```bash
./clipwise subtitle --clips ./clips.json --subtitles ./subs.json \
  --output-dir ./subtitled --style xhs
```

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `--clips` / `--clips-json` | 是 | — | clips JSON（clip 工具的输出） |
| `--subtitles` / `--subtitles-json` | 是 | — | 字幕 JSON（见格式） |
| `--output-dir` | 是 | — | 输出目录 |
| `--style` | 否 | `xhs` | 字幕样式预设 |

subtitles JSON 格式：
```json
[
  { "clipIndex": 0, "cues": [
    { "start": 0.0, "end": 3.5, "text": "第一句字幕" },
    { "start": 4.0, "end": 7.2, "text": "第二句字幕" }
  ]}
]
```

---

### 3.6 dub — TTS 配音 + 音频混合

使用 edge-tts 生成中文语音并混入视频。可选替换原音或降低原音量叠加配音。

```bash
./clipwise dub --clips ./clips.json --texts ./texts.json \
  --output-dir ./dubbed --voice zh-CN-YunxiNeural --mix-mode replace
```

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `--clips` / `--clips-json` | 是 | — | clips JSON |
| `--texts` / `--texts-json` | 是 | — | 配音文本 JSON（支持 `[]DubText` 或 `[]TopMoment` 格式） |
| `--output-dir` | 是 | — | 输出目录 |
| `--voice` | 否 | `zh-CN-YunxiNeural` | TTS 语音名称（edge-tts 语音列表） |
| `--mix-mode` | 否 | `replace` | `replace`：替换原音频 / `background`：降低原音量叠加配音 |

> 需要 `edge-tts` 已安装。配音文本支持两种格式：`DubText`（带起止时间的逐段文本）和 `TopMoment`（从 highlights 输出直接传入，自动取 summary/title 字段）。

---

### 3.7 concat — 拼接视频

将多个视频文件按顺序拼接为一个文件，支持可选的交叉过渡效果。

```bash
# 逗号分隔路径
./clipwise concat --inputs a.mp4,b.mp4,c.mp4 --output merged.mp4

# JSON 数组
./clipwise concat --inputs-json '["a.mp4","b.mp4"]' --output merged.mp4

# 从文件读取（每行一个路径），支持 stdin
./clipwise concat --inputs-file list.txt --output merged.mp4
```

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `--inputs` | 三选一 | — | 逗号分隔的输入文件路径 |
| `--inputs-json` | 三选一 | — | JSON 数组格式的文件路径 |
| `--inputs-file` | 三选一 | — | 文件路径（每行一个路径，`-` 表示 stdin） |
| `--output` | 是 | — | 输出文件路径 |
| `--transition` | 否 | `none` | 过渡效果：`none` / `xfade` |

---

### 3.8 crop — 裁剪画面

裁剪视频画面到指定矩形区域，常用于竖屏转横屏、去除黑边等。两种模式：

**Mode A（固定区域）**：
```bash
./clipwise crop --input video.mp4 --region 100:0:720:1280 --output cropped.mp4
```

**Mode B（多段裁剪）**：
```bash
./clipwise crop --input video.mp4 --regions-json '[{"region":"100:0:720:1280"}]' --output-dir ./cropped
```

| 参数 | Mode A | Mode B | 说明 |
|------|--------|--------|------|
| `--input` | 必填 | 必填 | 输入视频路径 |
| `--region` | 必填 | — | 裁剪区域 `x:y:w:h`（左上角坐标 + 宽高） |
| `--regions` / `--regions-json` | — | 必填 | 多区域 JSON（文件路径或内联） |
| `--output` | 必填 | — | 输出文件路径 |
| `--output-dir` | — | 必填 | 输出目录 |

---

### 3.9 extract-audio — 提取音轨

从视频中提取音频为独立音频文件，可选截取片段。

```bash
./clipwise extract-audio --input video.mp4 --output audio.wav --format wav
```

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `--input` | 是 | — | 输入视频路径 |
| `--output` | 是 | — | 输出音频文件路径 |
| `--format` | 否 | `wav` | 音频格式：`wav` / `mp3` / `aac` |
| `--start` | 否 | `0` | 截取起始时间（秒） |
| `--end` | 否 | `0` | 截取结束时间（秒），0 表示到文件末尾 |

---

### 3.10 narrate — AI 智能解说（中文化+配音）

将长视频智能分段，为每段选择最佳处理策略：AI 撰写中文解说词并 TTS 配音（覆盖原声），或保留精彩原声并添加中文字幕。最终输出带标题、字幕、配音的完整解说视频。

```bash
# 使用火山引擎
./clipwise narrate --input video.mp4 --output-dir ./out \
  --provider volc --model doubao-seed-2-0-lite-260428

# 使用本地 Claude CLI
./clipwise narrate --input video.mp4 --output-dir ./out \
  --provider claude --model opus
```

流水线步骤：
1. **AI 分析** — 将视频划分为连续片段，每段标注 `narration`（解说覆盖原声）或 `original_audio`（保留原声+字幕），同时生成解说词/字幕文本、爆款标题
2. **切片** — 按分段逐段切割
3. **去水印** — 仅当 AI 检测到水印时执行
4. **逐段处理** — `narration` 片段：TTS 替换原声 + 烧录字幕；`original_audio` 片段：去除硬字幕（如有）+ 烧录中文字幕
5. **拼接** — 将所有处理后的片段合并为完整视频
6. **标题** — 叠加 AI 生成的爆款标题

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `--input` | 是 | — | 输入视频路径 |
| `--output-dir` | 是 | — | 输出目录 |
| `--voice` | 否 | `zh-CN-YunxiNeural` | 解说配音 TTS 语音 |
| `--subtitle-style` | 否 | `xhs` | 字幕样式预设 |
| `--title-style` | 否 | `xhs` | 标题样式预设 |
| `--no-cache` | 否 | `false` | 禁用缓存 |
| `--cache-dir` | 否 | `~/.clipwise/cache` | 缓存目录 |

以及所有 [AI Provider 参数](#2-ai-provider-参数)。

输出 `data` 字段：
```json
{
  "input": "video.mp4",
  "output": "./out/narrate_concat.mp4",
  "title": "当这个AI说完这句话 全场都安静了",
  "language": "en",
  "style": "entertainment",
  "segment_count": 5
}
```

#### NarrateResult — AI 分析结果

```json
{
  "title": "当这个AI说完这句话 全场都安静了",
  "language": "en",
  "style": "entertainment",
  "watermark": { "x": 10, "y": 25, "w": 200, "h": 50 },
  "has_hard_subtitles": true,
  "hard_subtitle_region": { "x": 100, "y": 850, "w": 800, "h": 100 },
  "segments": [
    {
      "start": 0.0,
      "end": 18.0,
      "type": "narration",
      "text": "今天这期视频来自一场硅谷闭门对谈，演讲者是推特创始人Jack Dorsey，他要用一个颠覆性的观点，重新定义公司这件事。",
      "reason": "开场引入，交代背景和悬念"
    },
    {
      "start": 18.0,
      "end": 90.0,
      "type": "original_audio",
      "text": "我觉得AI的未来不在于取代人类 而在于增强人类的能力",
      "reason": "嘉宾核心观点 保留原声感染力"
    }
  ]
}
```

**segments 字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `start` / `end` | float | 片段起止时间（秒） |
| `type` | string | `narration`：AI 解说覆盖原声 / `original_audio`：保留原声 |
| `text` | string | `narration`：中文解说词；`original_audio`：中文字幕文本 |
| `reason` | string | AI 选择该处理方式的理由 |

**其他字段**：
- `title`：AI 生成的爆款标题（不超过20字）
- `language`：检测到的语言代码
- `style`：视频风格判断（`entertainment` / `information`）
- `has_hard_subtitles`：是否检测到原视频有硬字幕
- `hard_subtitle_region`：硬字幕区域坐标（有硬字幕时自动去除并覆盖中文字幕）

---

### 3.11 highlights — 一键精华切片提取

完整工作流：AI 分析视频 → 自动切片 → 去水印 → 配音（外文视频）→ 加字幕 → 加标题。输入一个长视频，输出多个可直接发布的精彩短片段。

```bash
# 使用火山引擎
./clipwise highlights --input video.mp4 --output-dir ./out --clips 3 \
  --provider volc --model doubao-seed-2-0-lite-260428

# 使用本地 Claude CLI
./clipwise highlights --input video.mp4 --output-dir ./out --clips 3 \
  --provider claude --model opus
```

流水线步骤（按需跳过）：
1. **AI 分析** — 使用内置 highlights 专用 prompt 模板分析视频，找出精彩片段、检测水印和语言
2. **切片** — 批量切割指定数量的片段
3. **去水印** — 仅当 AI 检测到水印时执行
4. **配音** — 仅当检测到非中文语言时执行（英文 → 中文 TTS）
5. **字幕** — 为每个片段添加中文摘要字幕
6. **标题** — 为每个片段叠加标题文字

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `--input` | 是 | — | 输入视频路径 |
| `--output-dir` | 是 | — | 输出目录 |
| `--clips` | 否 | `5` | 提取的精彩片段数量 |
| `--voice` | 否 | `zh-CN-YunxiNeural` | 配音 TTS 语音（仅外文视频时生效） |
| `--title-style` | 否 | `xhs` | 标题样式预设：`xhs` / `minimal` |
| `--title-position` | 否 | `top` | 标题位置：`top` / `center` / `bottom` |
| `--no-cache` | 否 | `false` | 禁用缓存，强制重新分析 |

以及所有 [AI Provider 参数](#2-ai-provider-参数)。

highlights 输出 `data` 字段：
```json
{
  "clips": [
    { "momentIndex": 0, "filePath": "./out/clip_000.mp4", "duration": 15.5 },
    { "momentIndex": 1, "filePath": "./out/clip_001.mp4", "duration": 12.3 }
  ],
  "titled": [
    { "file_path": "./out/titled_clip_000.mp4" },
    { "file_path": "./out/titled_clip_001.mp4" }
  ],
  "language": "en",
  "dubbed": true,
  "count": 2
}
```

#### AnalysisResult — AI 分析结果

highlights 内部 AI 分析返回的完整结构（`data` 中不直接暴露，但可从缓存或手动分析场景获取）：

```json
{
  "topMoments": [
    {
      "index": 0,
      "title": "震撼开场：极限运动精彩集锦",
      "summary": "运动员从悬崖跳伞的壮观画面...",
      "score": 0.95,
      "reason": "视觉冲击力强，节奏紧凑",
      "sequence": [
        { "start": 10.5, "end": 18.2, "role": "action", "text": "运动员准备跳伞" },
        { "start": 18.2, "end": 25.0, "role": "highlight", "text": "空中翻转特写" }
      ]
    }
  ],
  "transcript": {
    "title": "极限运动年度集锦",
    "summary": "本视频汇集了年度最佳极限运动画面...",
    "fullText": "完整转录文本..."
  },
  "detectedLanguage": "en",
  "watermark": { "x": 10, "y": 25, "w": 200, "h": 50 },
  "hasSubtitles": false
}
```

**topMoments 字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `index` | int | 片段序号，后续工具通过此字段关联 |
| `title` | string | 片段标题，用于 title 工具叠加文字 |
| `summary` | string | 片段摘要，用于 subtitle 工具烧录字幕 |
| `score` | float | 精彩度评分（0-1），越高越精彩 |
| `reason` | string | AI 认为精彩的理由 |
| `sequence` | array | 片段内的细分段落，每段含 start/end/role/text |

**其他字段**：
- `detectedLanguage`：检测到的语言代码（`zh`/`en`/...），非中文时自动触发配音
- `watermark`：水印位置，存在时自动触发去水印（`null` 表示未检测到）
- `transcript`：完整视频的文字转录

> topMoments JSON 可直接传给 `clip --moments`、`title --moments`、`dub --texts` 等命令的输入。

---

### 3.12 translate-video — 一键视频翻译中文化

完整工作流：AI 分析语言 → 逐句转录翻译 → 去水印 → 重配音 → 烧录中文字幕。将外文视频转为带中文字幕+配音的完整视频。

```bash
# 使用火山引擎
./clipwise translate-video --input english_video.mp4 --output-dir ./out \
  --provider volc --model doubao-seed-2-0-lite-260428

# 使用本地 Claude CLI
./clipwise translate-video --input english_video.mp4 --output-dir ./out \
  --provider claude --model sonnet
```

流水线步骤：
1. **AI 分析** — 检测语言，逐句转录原文并翻译为中文（带精确时间戳）
2. **去水印** — 仅当 AI 检测到水印时执行
3. **重配音** — 用中文 TTS 替换原始音频，自动重映射时间轴
4. **字幕** — 烧录中文字幕

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `--input` | 是 | — | 输入视频路径 |
| `--output-dir` | 是 | — | 输出目录 |
| `--voice` | 否 | `zh-CN-YunxiNeural` | 配音 TTS 语音 |
| `--subtitle-style` | 否 | `xhs` | 字幕样式预设 |
| `--no-cache` | 否 | `false` | 禁用缓存 |

以及所有 [AI Provider 参数](#2-ai-provider-参数)。

输出 `data` 字段：
```json
{
  "input": "english_video.mp4",
  "output": "./out/clip_000.mp4",
  "language": "en",
  "sentence_count": 23
}
```

---

## 4. 常用工作流

### 工作流 A：精华切片提取

一键完成（推荐）：
```bash
./clipwise highlights --input video.mp4 --output-dir ./out --clips 3
```

手动分步（适合需要精细控制的场景）：
```bash
# 1. 探测视频信息
./clipwise probe --input video.mp4

# 2. 切割已知片段
./clipwise clip --input video.mp4 --start 10.0 --end 25.5 --output clip1.mp4

# 3. 加标题
./clipwise title --input clip1.mp4 --text "精彩标题" --output titled.mp4

# 4. 拼接多段（可选）
./clipwise concat --inputs ./titled/clip_000.mp4,./titled/clip_001.mp4 --output final.mp4
```

### 工作流 B：翻译 + 配音

```bash
# 一键完成外文视频中文化
./clipwise translate-video --input english_video.mp4 --output-dir ./out
```

### 工作流 C：AI 智能解说

适合需要将外文/长视频转为中文解说短视频的场景。AI 自动规划解说节奏、撰写解说词、保留精彩原声。

```bash
./clipwise narrate --input video.mp4 --output-dir ./out --provider claude --model opus
```

---

## 5. JSON 输入通用规则

所有支持 JSON 输入的参数（如 `--moments`, `--clips`, `--texts`, `--subtitles`）都有三种传法：

| 写法 | 示例 | 说明 |
|------|------|------|
| `--X <path>` | `--moments ./data.json` | 从文件读取 |
| `--X-json '<inline>'` | `--moments-json '[{"start":0}]'` | 内联 JSON |
| `--X -` | `--moments -` | 从 stdin 读取 |

---

## 6. 全局参数

所有子命令通用：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--quiet` | `false` | 静默 stderr 日志 |
| `--verbose` | `false` | 详细调试日志 |

---

## 7. 错误处理

所有错误输出结构化 JSON：
```json
{
  "ok": false,
  "error": {
    "code": "API_AUTH_FAILED",
    "message": "invalid API key",
    "fix_hint": "Check --api-key or CLIPWISE_API_KEY",
    "retryable": false
  }
}
```

常见 exit code：

| Code | 含义 |
|------|------|
| 0 | 成功 |
| 1 | 内部错误 |
| 2 | 用法错误 |
| 3 | 输入文件错误 |
| 4 | 外部依赖缺失（ffmpeg 等） |
| 5 | 网络/API 错误 |
| 6 | AI 返回解析失败 |
| 130 | 用户取消（Ctrl+C） |
