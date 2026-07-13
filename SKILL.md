---
name: clipwise-cli
description: |
  本地视频处理 CLI 工具（agent-friendly，JSON in/out）。当用户要求分析视频、生成切片、
  提取精华片段、视频转写、生成图文稿、裁剪视频、去水印、添加字幕/标题、配音、拼接视频、
  提取音频、翻译视频、解说视频、AI配音解说，或提到 "clipwise"、"切片"、"视频分析"、"短视频"、"highlight"、"narrate"、"解说"
  等关键词时使用。
---

# clipwise — Agent-First Video CLI

clipwise is a single binary that does atomic video operations. Every operation is one tool, invoked the same way:

```bash
clipwise run <tool> --request '<JSON>'
```

This skill makes the model a fluent clipwise operator: pick the right tool, build a valid request, read the structured response, and recover from errors. **Always actually run the command** — never describe what clipwise would do without executing it.

## The one invocation pattern

Three equivalent ways to pass the request JSON (all set `--request`):

```bash
# 1. Inline JSON  (preferred for short requests)
clipwise run media.probe --request '{"tool":"media.probe","input":{"path":"video.mp4"}}'

# 2. File path
clipwise run media.probe --request request.json

# 3. Stdin
echo '{"tool":"media.probe","input":{"path":"video.mp4"}}' | clipwise run media.probe --request -
```

The JSON object shape is always:

```json
{ "tool": "<name>", "input": { ... }, "options": { ...optional... } }
```

`tool` MUST match the tool name passed to `run`. `options` is optional and carries cross-cutting config (see "Options" below).

## CRITICAL — I/O discipline

clipwise is **agent-first**: stdout is a single JSON Response and nothing else. All human-readable logs go to stderr.

- **Parse stdout as JSON only.** Never grep stdout for prose, never assume a line is a path.
- **Diagnostic text the user "sees" is on stderr** — don't treat it as the result.
- Prefer **absolute paths** for all `input`/`video`/`output`/`output_dir`/`srt_path` arguments. Relative paths resolve against clipwise's CWD and `INPUT_NOT_FOUND` is the most common self-inflicted error.
- All output dirs/files are created by clipwise; **temp/** is the conventional scratch location in this repo.

## The response shape

Success:

```json
{
  "ok": true,
  "tool": "media.probe",
  "run_id": "20260713-154022-ef1da1",
  "duration_ms": 123,
  "data": {
    "artifacts": [{ "kind": "video", "role": "final", "path": "/abs/out.mp4" }],
    "summary":   { "...": "tool-specific key facts" },
    "warnings":  [{ "code": "...", "stage": "...", "message": "..." }],
    "next":      [{ "tool": "media.concat", "reason": "join the clips" }]
  }
}
```

- `data.artifacts[].path` — the files produced. `role: "final"` = the deliverable; `role: "intermediate"` = scratch.
- `data.summary` — concise facts (e.g. probe metadata, clip count). Read this first to answer "what happened".
- `data.warnings` — non-fatal issues; the tool still succeeded. Always surface these to the user.
- `data.next` — suggested follow-up tools. Optional, not binding.

Error (note: `ok:false`, `error` object, no `data`):

```json
{
  "ok": false,
  "tool": "media.probe",
  "run_id": "20260713-154022-013621",
  "error": {
    "code": "INPUT_NOT_FOUND",
    "message": "video.mp4: stat video.mp4: no such file or directory",
    "retryable": false,
    "hint": "Check the file path is absolute and exists"
  }
}
```

## Error codes and what to do

Always branch on `error.code`, never on string-matching `message`. Treat retryable errors with backoff; fix non-retryable ones.

| code | retryable | meaning / fix |
|---|---|---|
| `USAGE_ERROR` | no | bad CLI flags |
| `INPUT_NOT_FOUND` | no | file path wrong/missing — use absolute path |
| `INPUT_INVALID` | no | bad request value (bad time range, empty text, etc.) |
| `FFMPEG_MISSING` | no | install ffmpeg (`brew install ffmpeg` / `apt install ffmpeg`) |
| `FFPROBE_MISSING` | no | ships with ffmpeg |
| `EDGE_TTS_MISSING` | no | `pip install edge-tts` (needed by dub/narrate/translate) |
| `CLAUDE_CLI_MISSING` | no | claude cli not on PATH |
| `API_AUTH_FAILED` | no | check `CLIPWISE_API_KEY` / `--api-key` |
| `API_RATE_LIMIT` | **yes** | wait + retry, lower concurrency |
| `API_TIMEOUT` | **yes** | network/model busy — retry with backoff |
| `API_SERVER_ERROR` | **yes** | upstream 5xx — retry with backoff |
| `AI_PARSE_FAILED` | **yes** | AI returned bad JSON — retry once |
| `VIDEO_TOO_LONG` | no | split first (`workflow.segment_video`), keep segments < 80 min |
| `TOOL_NOT_FOUND` | no | typo in tool name — run `clipwise tools list` |
| `TOOL_MISMATCH` | no | `tool` field in JSON ≠ the `<tool>` arg of `run` |
| `INTERNAL_ERROR` | no | bug; report with `run_id` |
| `CANCELED` | no | Ctrl+C |

## Tool catalog (15 tools)

Pick the **most specific** tool. If a workflow exists for the goal, use the workflow — don't hand-chain media primitives when a pipeline already does it atomically and atomically manages warnings/next-actions.

### Inspect / low-level media

| tool | required input | what it does |
|---|---|---|
| `media.probe` | `path` | metadata: duration, resolution, audio, format. **Run this first** on any unknown video. |
| `media.clip` | `input, start, end, output` | cut one segment by seconds. `re_encode:true` for frame-accurate cuts. |
| `media.clip_batch` | `input, output_dir, segments[]` | cut many segments at once. Each segment: `{start,end,title?}`. |
| `media.concat` | `inputs[], output` | join videos into one. |
| `media.crop` | `input, region, output` | crop to `x:y:w:h`. |
| `media.extract_audio` | `input, output` | pull audio track. Optional `format,start,end`. |
| `media.subtitle` | `input, srt_path, output` | burn an `.srt` into video. Optional `style`. |
| `media.title` | `input, text, output` | overlay title text. Optional `position,style`. |

### AI

| tool | required input | what it does |
|---|---|---|
| `ai.transcribe` | `path` | ASR → timestamped text. Optional `language` (`zh`/`en`/`auto`, default `auto`). |
| `ai.analyze_video` | `path, prompt` | vision-model analysis with a free-text prompt. |

### Workflows (multi-step pipelines)

Workflows are the high-level pipelines. Each takes `video` + `output_dir`, runs AI + ffmpeg steps, and emits final artifacts. **Always prefer a workflow over hand-chaining media tools** — it handles ASR/AI/dubbing/subtitle/delogo atomically and collects non-fatal failures into `warnings`.

**Which workflow?** Pick by intent:

| user wants | tool | one-line |
|---|---|---|
| 把外语视频翻成中文配音+字幕 | `workflow.translate_video` | translate, keep timeline, dub in Chinese |
| 从视频里剪出几个高光片段 | `workflow.highlights` | multiple separate highlight clips |
| 把精华片段拼成一个合集 | `workflow.essence` | one concatenated best-of video |
| 给视频重新写解说词并配音 | `workflow.narrate` | AI re-authored narration |
| 把长视频按主题切成多段 | `workflow.segment_video` | topic split, no dubbing |

`highlights` vs `essence`: highlights → N 个**独立**文件；essence → **单个拼接**合集。`translate` vs `narrate`: translate 逐句译原文、时间线不变；narrate 由 AI **重写**解说词、重排时长。

Common to translate/highlights/narrate/essence: optional `voice` (TTS 声音，默认 `zh-CN-YunxiNeural`)，`subtitle` (`auto` 原片无字幕才加 / `true` 强制 / `false` 关闭)。需要 edge-tts。

Below: each workflow's **steps / inputs / outputs / behavior**.

---

#### `workflow.translate_video` — 翻译视频为中文配音+字幕

把外语视频翻成中文：ASR 转写 → AI 翻译 → 中文 TTS 配音 → 烧录字幕 + 标题。**原始时间线永远不动**——TTS 用 atempo（≤1.2×）压缩塞进原片段窗口，避免音视频漂移。配音和字幕在单个 ffmpeg pass 里一起烧录。

**Input**

| field | type | req | note |
|---|---|---|---|
| `video` | string | ✅ | 输入视频绝对路径 |
| `output_dir` | string | ✅ | 输出目录 |
| `voice` | string | | TTS 声音，默认 `zh-CN-YunxiNeural` |
| `subtitle_style` | enum | | `xhs`(默认) \| `minimal` |
| `subtitle` | enum | | `auto` \| `true` \| `false`；`false` 时仅配音不烧字幕 |

**Pipeline**: `asr`(fatal) → `analyze`(fatal, 翻译+视觉水印/标题/语言检测) → `delogo`(nonfatal, 去水印) → `dub`(nonfatal, TTS+字幕单pass) → `subtitle`(optional, dub没烧时fallback) → `title`(nonfatal, 加标题到顶部)。

**Output**: 单个最终文件 `output_dir/<原文件名>_translated.mp4`（恒为 MP4，无论输入 mkv/webm）。中间文件在 `output_dir/clips/`。

**Summary**: `output`(最终路径), `language`(检测到的源语言), `has_subtitles`(原片是否已有字幕), `title`(生成的标题), `entry_count`(翻译条数), `warnings[]`。

**注意**：超短视频段（<0.7s，如"Yes."）会被自动合并到相邻段再翻译；TTS 即使在 maxTTSSpeed 仍超窗口会被截断并记 warning（非致命，保留原音）。

```bash
clipwise run workflow.translate_video --request '{
  "tool":"workflow.translate_video",
  "input":{"video":"/abs/v.mp4","output_dir":"/abs/temp/translate_out","subtitle":"true"}
}'
```

---

#### `workflow.highlights` — 提取高光片段

ASR + 视频分析找出 Top-N 精彩瞬间，每个瞬间剪成一个独立片段（一个瞬间可能由多段 sequence 拼接，带 fade 转场），去水印、烧字幕、加标题。

**Input**

| field | type | req | note |
|---|---|---|---|
| `video` | string | ✅ | 输入视频 |
| `output_dir` | string | ✅ | 输出目录 |
| `clips` | integer | | 高光片段数量，默认 `3` |
| `voice` | string | | TTS 声音 |
| `title_style` | enum | | `xhs` \| `minimal` |
| `title_position` | enum | | `top` \| `center` \| `bottom` |
| `subtitle` | enum | | `auto` \| `true` \| `false` |

**Pipeline**: `asr`(fatal) → `analyze`(fatal, 视频分析找 TopMoments) → `clip`(fatal, 切sequence→concat) → `delogo`(nonfatal) → `subtitle`(optional) → `title`(nonfatal)。

**Output**: **多个** `output_dir/clips/highlight_0.mp4`, `highlight_1.mp4`, …。每个是独立可发布的片段。

**Summary**: `clips[]`(每段 `{moment_index, file_path, duration}`), `language`, `count`, `title`, `warnings[]`。

```bash
clipwise run workflow.highlights --request '{
  "tool":"workflow.highlights",
  "input":{"video":"/abs/v.mp4","output_dir":"/abs/temp/highlights","clips":5,"title_position":"top"}
}'
```

---

#### `workflow.essence` — 精华合集（拼接成一个视频）

与 highlights 类似（ASR→AI 选精华），但把所有精选片段**拼接成单个合集视频**，适合做"一镜精华版"。默认 5 段，只给首个片段加标题（作为合集开场）。

**Input**

| field | type | req | note |
|---|---|---|---|
| `video` | string | ✅ | 输入视频 |
| `output_dir` | string | ✅ | 输出目录 |
| `clips` | integer | | 选取片段数，默认 `5`（注意：highlights 默认 3） |
| `voice` | string | | TTS 声音 |
| `title_style` | enum | | `xhs` \| `minimal` |
| `subtitle` | enum | | `auto` \| `true` \| `false` |

**Pipeline**: `asr`(fatal, 有 ASR provider 才跑) → `filter`(fatal, 过滤<5字的ASR段) → `analyze`(fatal, AI 选精华片段) → `clip`(fatal, 批量切割) → `delogo`(nonfatal) → `subtitle`(optional) → `title`(nonfatal, 仅首段) → `concat`(fatal, 拼成单文件)。

**Output**: 单个 `output_dir/essence.mp4`。

**Summary**: `output`, `title`, `language`, `moment_count`, `warnings[]`。

```bash
clipwise run workflow.essence --request '{
  "tool":"workflow.essence",
  "input":{"video":"/abs/v.mp4","output_dir":"/abs/temp/essence","clips":6}
}'
```

---

#### `workflow.narrate` — AI 解说重配音

让 AI **重新撰写**中文解说词（基于 ASR 语义，非逐句翻译原文），分段剪辑、每段配音并把视频重对齐到解说时长，最终拼成一个完整解说视频。每段按 AI 决定的 `audio_mode` 处理原音：`original`（保留原音不配音）/ `ambient`（原音降到 0.5）/ `dub`（原音降到 0.2）。

**Input**

| field | type | req | note |
|---|---|---|---|
| `video` | string | ✅ | 输入视频 |
| `output_dir` | string | ✅ | 输出目录 |
| `voice` | string | | TTS 声音 |
| `subtitle_style` | enum | | `xhs` \| `minimal`（字幕样式） |
| `title_style` | enum | | `xhs` \| `minimal`（标题样式） |
| `subtitle` | enum | | `auto` \| `true` \| `false` |

**Pipeline**: `asr`(fatal, 过滤<5字短段) → `analyze`(fatal, AI 生成分段解说词+audio_mode) → `segments`(fatal, 逐段: clip→delogo→TTS→配音+字幕) → `concat`(fatal, fade转场拼接) → `title`(nonfatal)。

**Output**: 单个 `output_dir/narrate_concat.mp4`。每段中间产物在 `output_dir/clips/seg_N/`。

**Summary**: `output`, `title`, `language`, `segment_count`, `warnings[]`。

**注意**：与 translate 不同——narrate 会**改变视频时长**（每段 setpts 对齐到 TTS 时长），是"重编排"而非"原时间线翻译"。

```bash
clipwise run workflow.narrate --request '{
  "tool":"workflow.narrate",
  "input":{"video":"/abs/v.mp4","output_dir":"/abs/temp/narrate"}
}'
```

---

#### `workflow.segment_video` — 按主题分段（纯切分）

把长视频按 AI 检测的**主题切换点**切成多段。**无 ASR、无配音、无字幕、不重编码字幕**——纯切割，用于"把一个长视频拆成多个可独立发布的段落"。每段时间线连续（首段 start=0，末段 end=总时长，相邻无缝）。

**Input**

| field | type | req | note |
|---|---|---|---|
| `video` | string | ✅ | 输入视频 |
| `output_dir` | string | ✅ | 输出目录 |

（无可选参数）

**Pipeline**: `probe`(fatal) → `prepare`(fatal, 上传前缩到 720p/≤500MB) → `analyze`(fatal, AI 识别主题边界，校验时间线连续) → `clip`(fatal, 重编码精确切分) → `manifest`(fatal, 写清单)。

**Output**: 多个 `output_dir/NN_<标题>.mp4`（如 `01_开场介绍.mp4`）+ `output_dir/manifest.json`。manifest 含每段 `{index, file, start, end, title, summary}`。

**Summary**: `segments[]`(每段 `{index, start, end, title, summary}`), `segment_count`, `manifest_path`, `warnings[]`。

**注意**：这是唯一不依赖 edge-tts/ASR 的 workflow，适合给超长视频（>80分钟）先分段再做后续处理。

```bash
clipwise run workflow.segment_video --request '{
  "tool":"workflow.segment_video",
  "input":{"video":"/abs/long.mp4","output_dir":"/abs/temp/segments"}
}'
```

### Discovering specs at runtime

Never trust memory for a schema — introspect the binary (these are zero-dependency, instant):

```bash
clipwise tools list                 # all tools: name, namespace, required inputs
clipwise tools describe <tool>      # full input_schema for one tool
clipwise version                    # build/platform info
```

If a request errors with `TOOL_NOT_FOUND` or `INPUT_INVALID`, **run `tools describe` before retrying** to confirm the exact field names.

## Options (optional, cross-cutting)

Pass inside the request JSON under `options`:

```json
{
  "tool": "ai.analyze_video",
  "input": { "path": "/abs/v.mp4", "prompt": "describe the scene" },
  "options": {
    "provider": { "id": "openai", "model": "gpt-4o", "api_key": "sk-...", "base_url": "..." },
    "asr":      { "provider": "volc", "api_key": "..." },
    "cache":    "read_write",
    "verbose":  true
  }
}
```

- `provider` — override the AI provider/model/key. Falls back to env (`CLIPWISE_PROVIDER`, `CLIPWISE_MODEL`, `CLIPWISE_API_KEY`, `CLIPWISE_BASE_URL`).
- `asr` — ASR provider config. Falls back to `CLIPWISE_ASR_PROVIDER` etc.
- `cache` — `"read_write"` (default) | `"read_only"` | `"disabled"`. Content-addressed by SHA-256 of inputs, so changed prompts/params auto-miss. **Don't manually clear cache.**
- `quiet` / `verbose` — suppress or expand stderr logs.

## Configuration & dependencies

clipwise reads `~/.clipwise/.env` automatically (existing env vars win). Key vars:

- AI: `CLIPWISE_PROVIDER`, `CLIPWISE_MODEL`, `CLIPWISE_API_KEY`, `CLIPWISE_BASE_URL`
- ASR: `CLIPWISE_ASR_PROVIDER`, `CLIPWISE_ASR_API_KEY`, `CLIPWISE_ASR_BASE_URL`, `CLIPWISE_ASR_MODEL`, `CLIPWISE_ASR_LANGUAGE`
- Whisper (local ASR): `CLIPWISE_WHISPER_BINARY`, `CLIPWISE_WHISPER_MODEL`, `CLIPWISE_WHISPER_DEVICE`, `CLIPWISE_WHISPER_COMPUTE_TYPE`, `CLIPWISE_WHISPER_ALIGN_MODEL`, `CLIPWISE_WHISPER_INTERPOLATE_METHOD`

External deps: **ffmpeg** (required), **edge-tts** (for dub/narrate/translate), **whisperx-cli** (optional local ASR). Logs at `~/.clipwise/logs/clipwise-YYYY-MM-DD.log`.

## How to operate — the workflow

1. **Locate the binary.** If `clipwise` isn't on PATH, it builds to the repo root: use `./clipwise` from the project dir, or ask where the binary is. Confirm with `clipwise version`.
2. **Probe unknowns.** Before cutting/analyzing, run `media.probe` to learn duration/resolution — drives sane `start`/`end` values and catches `VIDEO_TOO_LONG` early.
3. **Build the request.** Use absolute paths. Match the `tool` field to the `run` arg. When unsure of a field, run `clipwise tools describe <tool>`.
4. **Execute and parse.** Run `clipwise run <tool> --request '<json>'`. Parse stdout as JSON. Branch on `ok`.
   - `ok:true` → read `data.summary`, surface any `data.warnings`, point the user at `data.artifacts[*].path` where `role:"final"`.
   - `ok:false` → read `error.code`. Retry only if `retryable:true`. Otherwise fix the input/deps and re-run.
5. **Chain when sensible.** Use `data.next` as hints. Prefer a single workflow tool over hand-chaining media primitives.
6. **Report faithfully.** Give the user the artifact paths and any warnings. If a step failed, say so with the code — don't claim success.

## Worked examples

### Translate a video to Chinese
```bash
clipwise run workflow.translate_video --request '{
  "tool":"workflow.translate_video",
  "input":{"video":"/abs/input.mp4","output_dir":"/abs/temp/translate_out"}
}'
```
Read `data.artifacts` for the final dubbed+subtitled video; report `data.warnings`.

### Cut a 10s highlight from 30s–40s, frame-accurate
```bash
clipwise run media.clip --request '{
  "tool":"media.clip",
  "input":{"input":"/abs/v.mp4","start":30,"end":40,"output":"/abs/temp/clip.mp4","re_encode":true}
}'
```

### Batch-cut several segments
```bash
clipwise run media.clip_batch --request '{
  "tool":"media.clip_batch",
  "input":{
    "input":"/abs/v.mp4",
    "output_dir":"/abs/temp/clips",
    "segments":[{"start":12,"end":45,"title":"intro"},{"start":120,"end":180,"title":"demo"}]
  }
}'
```

### Probe then transcribe
```bash
# 1. learn duration / confirm audio track exists
clipwise run media.probe --request '{"tool":"media.probe","input":{"path":"/abs/v.mp4"}}'
# 2. transcribe (language auto-detected; force zh here)
clipwise run ai.transcribe --request '{"tool":"ai.transcribe","input":{"path":"/abs/v.mp4","language":"zh"}}'
```

### Extract 5 highlights (separate clips)
```bash
clipwise run workflow.highlights --request '{
  "tool":"workflow.highlights",
  "input":{"video":"/abs/v.mp4","output_dir":"/abs/temp/hl","clips":5}
}'
# → multiple /abs/temp/hl/clips/highlight_0.mp4 ... highlight_4.mp4
```

### Build a single best-of essence compilation
```bash
clipwise run workflow.essence --request '{
  "tool":"workflow.essence",
  "input":{"video":"/abs/v.mp4","output_dir":"/abs/temp/essence"}
}'
# → single /abs/temp/essence/essence.mp4
```

### AI-narrate a video (re-authored Chinese narration)
```bash
clipwise run workflow.narrate --request '{
  "tool":"workflow.narrate",
  "input":{"video":"/abs/v.mp4","output_dir":"/abs/temp/narrate"}
}'
# → single /abs/temp/narrate/narrate_concat.mp4
```

### Split a long video into topic segments (no dubbing)
```bash
clipwise run workflow.segment_video --request '{
  "tool":"workflow.segment_video",
  "input":{"video":"/abs/long.mp4","output_dir":"/abs/temp/segments"}
}'
# → /abs/temp/segments/01_<title>.mp4, 02_<title>.mp4, ... + manifest.json
```

### Recover from a missing-dep error
If `EDGE_TTS_MISSING` comes back from a dubbing workflow (translate/narrate/essence/highlights):
1. Tell the user edge-tts is required.
2. Offer: `pip install edge-tts`.
3. Re-run the same request — no request change needed.
(`segment_video` does NOT need edge-tts.)
