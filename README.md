# video-to-markdown

将 MP4 讲座/课程视频转换为带 PPT 截图和语义分段的可读 Markdown 文档。

适合对录制讲座、培训视频、技术分享进行存档和检索。

## 效果

- 🎙 **语音转录**：使用 whisperX large-v2 模型，支持中英文，自动添加标点、纠正专有名词
- 🖼 **PPT 截图**：自动检测幻灯片切换，提取关键帧，与对应段落文字配对展示
- 📝 **语义分段**：Claude 将转录内容聚合为若干语义完整的段落，每段附标题和时间戳
- 🌐 **语言统一**：非中文内容自动翻译为简体中文，繁体同步转换
- 👤 **演讲人识别**：从自我介绍中提取岗位信息和背景信息，标注在文档顶部；多位演讲人时分别列出

**输出示例：**

```
## 00:02:15 – 00:08:40 ｜ 渲染管线优化思路

本次分享主要围绕两个问题展开：第一，在移动端如何...

![slide](slides/prefix_0003.jpg)
![slide](slides/prefix_0004.jpg)

---
```

## 依赖

| 工具 | 用途 |
|------|------|
| Python 3.9+ | 运行转录和 Markdown 生成脚本 |
| [ffmpeg](https://ffmpeg.org/) | 提取视频截图和场景时间点 |
| [whisperX](https://github.com/m-bain/whisperX) | 语音转录 + 对齐 |
| [Pillow](https://pillow.readthedocs.io/) + [imagehash](https://github.com/JohannesBuchner/imagehash) | pHash 去重（可选）|
| [anthropic](https://pypi.org/project/anthropic/) | 标点修复和语义聚合 |

```bash
pip install whisperx Pillow imagehash anthropic
brew install ffmpeg
```

需要配置 Claude API：在 `~/.zshrc` 中设置 `ANTHROPIC_API_KEY`（或 `ANTHROPIC_AUTH_TOKEN` + `ANTHROPIC_BASE_URL`）。

## 安装

```bash
npx skills add https://github.com/hy0brrr/Skill-Video-to-Markdown
```

或手动克隆到 skills 目录：

```bash
git clone https://github.com/hy0brrr/Skill-Video-to-Markdown.git ~/.agents/skills/video-to-markdown
npx skills link ~/.agents/skills/video-to-markdown
```

安装后，在 Claude Code 中说"帮我转录这个视频"或直接输入 `/video-to-markdown` 即可触发。

## 使用

```
/video-to-markdown <MP4文件路径或文件夹路径>
```

- 单个文件：`/video-to-markdown ~/Downloads/lecture.mp4`
- 整个文件夹：`/video-to-markdown ~/Downloads/videos/`（处理所有 .mp4）

每个视频会在其**同级目录**下生成：

```
video.mp4
transcript.md       ← 最终输出，可直接在 Obsidian / Typora 中阅读
clip.json           ← whisperX 原始转录数据（中间产物）
scene_log.txt       ← 幻灯片时间点
slides/
  prefix_0001.jpg
  prefix_0002.jpg
  ...
```

## 注意事项

- **速度**：CPU 模式下约每分钟视频需 2-3 分钟处理，长视频请耐心等待
- **中断恢复**：若中途失败，已有 `clip.json` 的视频可跳过转录，直接重跑 Markdown 生成
- **动画/录屏视频**：场景切换 > 10 次/分钟时自动判定为非 PPT 类视频，省略截图嵌入
- **路径**：`generate_markdown.py` 需与 `SKILL.md` 在同一目录，或通过环境变量指定路径

## 已知问题

详见 [SKILL.md](./SKILL.md#已知问题与修复)，包含以下问题的完整原因与修复方案：

- 长视频后半段缺少标点（`max_tokens` 截断）
- PPT 内嵌视频产生大量重复截图（ffmpeg 误判 + pHash 去重）
- `info.json` 字段为 `null` 导致崩溃
- 视频无可识别语音（0 segment）
- ffmpeg filter 缺少 `isnan` 导致 0 截图

## License

Proprietary
