# Clipwise CLI — AI Agent 视频处理工具

![logo](assets/logo.jpg)

本地视频处理 CLI 工具，专为 AI Agent 设计。所有输入输出均为 JSON 格式，Agent 无需阅读源码即可通过结构化接口完成视频处理任务。

## 功能概览

| 工具 | 说明 |
|------|------|
| `probe` | 视频元数据探测（时长、分辨率、音轨等） |
| `clip` | 按时间戳或 AI 分析结果批量切割视频 |
| `highlights` | 一键精华切片：AI 分析 → 切片 → 去水印 → 配音 → 字幕 → 标题 |
| `translate-video` | 一键视频翻译中文化：转录 → 翻译 → 去水印 → 重配音 → 中文字幕 |
| `title` | 在视频上叠加标题文字（支持小红书/极简样式） |
| `subtitle` | 将带时间戳的字幕烧录到视频画面 |
| `dub` | TTS 配音 + 音频混合（edge-tts） |
| `delogo` | 模糊去除水印/Logo |
| `crop` | 裁剪画面区域 |
| `concat` | 多视频拼接（支持交叉过渡） |
| `extract-audio` | 提取音轨为独立音频文件 |

支持的视频输入：本地文件路径、直接视频 URL、主流平台链接（B站、YouTube、小红书等）。

## 安装 Skill（推荐：让 Agent 自行安装）

Clipwise 以 **Skill** 形式分发给 AI Agent（如 Claude Code）。最简单的方式是让 Agent 自己完成安装：

### 方式一：让 Agent 自动安装（推荐）

在你的 AI Agent 对话中直接说：

```
安装 clipwise-cli skill：从 https://github.com/xieqiang/clipwise-cli 克隆仓库，将对应平台的二进制文件放到 PATH 中，并完成 skill 注册。
```

Agent 会自动：
1. 克隆本仓库
2. 识别当前平台，选择对应的二进制文件
3. 将 `clipwise` 放入 PATH 可达的位置
4. 将 SKILL.md 注册为可用 skill
5. 运行 `clipwise version` 验证安装

安装完成后，Agent 即可通过自然语言调用所有视频处理能力。

### 方式二：手动安装

```bash
# 1. 克隆仓库
git clone https://github.com/xieqiang/clipwise-cli.git
cd clipwise-cli

# 2. 选择对应平台的二进制
# clipwise-for-mac          — macOS (amd64/arm64)
# clipwise-for-Linux-amd    — Linux amd64
# clipwise-for-Linux-arm    — Linux arm64
# clipwise-for-windows.exe  — Windows

# 3. 放入 PATH 并赋权（macOS/Linux 示例）
cp clipwise-for-mac /usr/local/bin/clipwise
chmod +x /usr/local/bin/clipwise

# 4. 验证
clipwise version
```

### 外部依赖

- **ffmpeg**（必需）：视频处理核心依赖
- **edge-tts**（可选）：配音功能需要，`pip install edge-tts`

### AI 配置

Clipwise 使用 AI 完成视频分析和翻译。创建 `~/.clipwise/.env`：

```bash
mkdir -p ~/.clipwise
cat > ~/.clipwise/.env << 'EOF'
CLIPWISE_PROVIDER=volc
CLIPWISE_API_KEY=your-api-key
CLIPWISE_MODEL=doubao-seed-2-0-lite-260428
EOF
```

支持的 AI Provider：

| Provider | 说明 | 环境变量 `CLIPWISE_PROVIDER` |
|----------|------|------------------------------|
| 火山引擎 | 推荐，视频理解能力强 | `volc` |
| OpenAI 兼容 | 支持自定义 Base URL | `openai` |
| Claude CLI | 调用本地 Claude Code | `claude` |
| Mock | 测试用，不调用真实 API | `mock` |

## 快速使用

安装 skill 后，直接用自然语言与 Agent 沟通即可：

```
分析这个视频并提取 3 个精华片段
```

```
把这个英文视频翻译成中文，加上中文字幕和配音
```

```
把这几个视频拼接成一个，加上过渡效果
```

Agent 会自动调用对应的 clipwise 命令完成操作。

### 命令行直接使用

也可以直接运行 CLI：

```bash
# 一键精华切片
clipwise highlights --input video.mp4 --output-dir ./out --clips 3

# 视频翻译中文化
clipwise translate-video --input english.mp4 --output-dir ./out

# 探测视频信息
clipwise probe --input video.mp4

# 切割片段
clipwise clip --input video.mp4 --start 10.0 --end 25.5 --output clip.mp4

# 拼接视频
clipwise concat --inputs a.mp4,b.mp4,c.mp4 --output merged.mp4
```

所有命令输出 JSON envelope，日志走 stderr，方便 Agent 解析和管道操作。

## 详细文档

完整工具参数和用法请参考 [SKILL.md](./SKILL.md)。

## 关注

![wx](assets/wx.jpg)
![shp](assets/sph.jpg)

