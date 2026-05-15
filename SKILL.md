---
name: video-to-markdown
description: 将 MP4 视频（讲座/课程/演讲）转换为带 PPT 截图和语义分段的可读 Markdown 文档。使用 whisperX 转录、ffmpeg 提取截图、Claude 进行标点/语义聚合。当用户提到"视频转 Markdown"、"转录视频"、"生成 transcript"，或提供 .mp4 文件路径时使用。
license: Proprietary
---

# video-to-markdown

将一个或多个 MP4 视频文件转换为带 PPT 截图和语义分段的可读 Markdown 文档。

## 使用方式

```
/video-to-markdown <MP4文件路径或文件夹路径>
```

例如：
- `/video-to-markdown ~/Downloads/lecture.mp4`
- `/video-to-markdown ~/Downloads/videos/`（处理文件夹下所有 .mp4）

## 执行流程

收到指令后，按以下步骤处理：

### 第一步：收集待处理文件

- 如果参数是单个 `.mp4` 文件，直接处理该文件
- 如果参数是文件夹，列出其中所有 `.mp4` 文件
- 每个视频的输出目录即为视频文件所在的文件夹

### 第二步：询问行业/领域背景

询问用户：

> 这批视频属于哪个行业或领域？如果有明确的专业背景（例如："游戏开发"、"医疗影像"、"金融合规"），可以帮助更准确地纠正语音识别中的专有名词和术语。如果没有特定背景，跳过即可。

- 若用户提供了背景描述（如"游戏引擎开发，涉及 UE5、Unreal、Niagara 等术语"），记录为 `DOMAIN_CONTEXT`，在步骤 4e 中通过环境变量传入
- 若用户跳过，`DOMAIN_CONTEXT` 为空，generate_markdown.py 仅凭 info.json 中的课程标题做上下文

### 第三步：检查是否已有输出

对每个视频，检查同级目录下的 `transcript.md` 是否已存在：
- 若已存在，**询问用户**是否跳过或重新生成
- 若不存在，直接加入处理队列

### 第四步：给出时间估算

根据视频时长估算处理时间（每 10 分钟视频约需 2-3 分钟），告知用户总估时，等待确认后再开始。

### 第五步：逐个处理

对每个视频依次执行（Python：`/Applications/Xcode.app/Contents/Developer/Library/Frameworks/Python3.framework/Versions/3.9/bin/python3`，ffmpeg/ffprobe：`/opt/homebrew/bin/`）：

**4a. 获取视频时长**
```bash
/opt/homebrew/bin/ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 "<input.mp4>"
```

**4b. 提取 PPT 截图**

截图文件名需加上视频文件夹名作为前缀（避免 Obsidian 全局文件索引混淆同名图片）。
前缀生成规则：将文件夹名中的 `()<>[]空格'"\\#&?%` 替换为 `_`，合并连续下划线，取前 50 字符。

场景过滤规则：阈值 0.3（高于 0.2 以减少误判）+ 最少间隔 3 秒（抑制 PPT 内嵌播放视频产生的连续相似帧）。
**4b 和 4c 必须使用完全相同的 filter，保证截图文件与 scene_log 时间点一一对应。**

```bash
mkdir -p "<video_dir>/slides"
# SLIDE_PREFIX = 文件夹名经过上述规则处理后的前缀
# SCENE_FILTER 在 4b 和 4c 中必须完全一致
SCENE_FILTER="select='gt(scene,0.3)*(isnan(prev_selected_t)+gte(t-prev_selected_t,3))'"
/opt/homebrew/bin/ffmpeg -y -i "<input.mp4>" \
  -vf "${SCENE_FILTER},scale=1280:-1,format=yuvj420p" \
  -vsync vfr "<video_dir>/slides/${SLIDE_PREFIX}_%04d.jpg" \
  -loglevel error
```

**4b-extra. pHash 去重（仅当截图 > 100 张时）**

PPT 内嵌视频即使经过 Method A 过滤后可能仍有大量相似帧残留。超过 100 张截图时，用感知哈希去重：

```python
import imagehash, re, os
from PIL import Image

PHASH_THRESHOLD = 10  # 汉明距离阈值，越小越严格

slides_dir = "<video_dir>/slides"
scene_log  = "<video_dir>/scene_log.txt"

slide_files = sorted([f for f in os.listdir(slides_dir) if f.endswith('.jpg')])

# 加载时间点
pts_times = []
with open(scene_log) as f:
    for line in f:
        m = re.search(r'pts_time:([\d.]+)', line)
        if m:
            pts_times.append(float(m.group(1)))
        elif re.match(r'^[\d.]+$', line.strip()):
            pts_times.append(float(line.strip()))

# 标记并删除重复截图，同步更新 scene_log
keep = [True] * len(slide_files)
last_hash = imagehash.phash(Image.open(os.path.join(slides_dir, slide_files[0])))
for i in range(1, len(slide_files)):
    h = imagehash.phash(Image.open(os.path.join(slides_dir, slide_files[i])))
    if h - last_hash < PHASH_THRESHOLD:
        keep[i] = False
    else:
        last_hash = h

for i, (fname, k) in enumerate(zip(slide_files, keep)):
    if not k:
        os.remove(os.path.join(slides_dir, fname))

with open(scene_log, "w") as f:
    for i, t in enumerate(pts_times):
        if keep[i]:
            f.write(f"{t}\n")
```

**4c. 生成场景时间点日志**

注意：此命令不加 `-loglevel error`，否则 showinfo 输出会被屏蔽导致 scene_log.txt 为空。
```bash
/opt/homebrew/bin/ffmpeg -y -i "<input.mp4>" \
  -vf "${SCENE_FILTER},showinfo" \
  -vsync vfr -f null - \
  2>"<video_dir>/scene_log.txt"
```

**4d. 语音转录**

创建并运行转录脚本（需在顶部 patch torch.load 以兼容 PyTorch 2.6）：
```python
import torch
_orig_load = torch.load
def _patched_load(*args, **kwargs):
    kwargs.setdefault("weights_only", False)
    return _orig_load(*args, **kwargs)
torch.load = _patched_load

import whisperx, json, os
model = whisperx.load_model("large-v2", device="cpu", compute_type="int8", language="zh")
result = model.transcribe("<input.mp4>", language="zh", batch_size=4)
align_model, metadata = whisperx.load_align_model(language_code="zh", device="cpu")
result = whisperx.align(result["segments"], align_model, metadata, "<input.mp4>", device="cpu")
with open("<video_dir>/clip.json", "w") as f:
    json.dump(result, f, ensure_ascii=False, indent=2)
```

**4e. 生成 Markdown**

调用 `generate_markdown.py`（与 `SKILL.md` 同级目录，或通过环境变量 `GENERATE_MD_SCRIPT` 指定绝对路径），设置以下环境变量后执行：

```bash
TRANSCRIPT_JSON=<video_dir>/clip.json \
SCENE_LOG=<video_dir>/scene_log.txt \
SLIDES_DIR=<video_dir>/slides \
OUTPUT_MD=<video_dir>/transcript.md \
VIDEO_DOMAIN_CONTEXT="<DOMAIN_CONTEXT 或留空>" \
python3 generate_markdown.py
```

`generate_markdown.py` 会自动处理以下逻辑：
- **语言统一**：非中文内容翻译为简体中文，繁体转简体
- **纠错**：结合视频所在行业背景纠正同音字和专有名词识别错误；如同级目录有 `info.json`，自动读取课程标题作为额外上下文
- **视频类型检测**：场景变化 > 10次/分钟判定为动画/录屏类，自动省略截图嵌入
- **图片结构**：每个语义段落的所有幻灯片统一展示在文字下方，不插入文字中间
- **演讲人识别**：从自我介绍中提取演讲人的岗位信息和背景信息，写在文档顶部；检测到多位演讲人时分别标注

### 第六步：完成汇报

全部处理完后，列出每个视频的处理结果：
- ✅ 成功：输出路径 + 分段数量 + 是否识别到讲师身份
- ❌ 失败：错误原因

## 注意事项

- whisperX 转录是最耗时的步骤，CPU 模式下约每分钟视频需要 2-3 分钟处理时间
- 如果中途失败，已生成的 `clip.json` 可跳过重新转录，直接从步骤 4c 继续
- Claude API 需配置 `ANTHROPIC_API_KEY`（标准用法），或自定义 `ANTHROPIC_BASE_URL` 指向代理端点
- ffmpeg 和 ffprobe 在 `/opt/homebrew/bin/`

## 已知问题与修复

### 长视频后半段缺少标点

**症状**：生成的 transcript 后半段文字没有标点符号，直接是原始转录的连续文字。

**原因**：`add_punctuation` 将所有 segment 一次性发给 Claude，长视频（segment 数量多、每段文字长）会超出 `max_tokens=8192`，Claude 回复被截断，后续 segment 收不到处理结果，保留原始无标点文本。

**修复**：`generate_markdown.py` 已改为分批处理（默认每批 40 个 segment，常量 `PUNCTUATION_CHUNK_SIZE`）。如遇截断警告（`⚠ 被截断`），可进一步将该常量调小。

**受影响视频的补救**：直接重新运行 `generate_markdown.py`（无需重新转录），只会重新调用 Claude 的标点和语义聚合接口，约 5-10 分钟。

### PPT 内嵌视频产生大量重复截图

**症状**：某个视频生成了数百乃至上千张截图，transcript 中图片密集重复，实际 PPT 页数远少于截图数量。

**原因**：讲师 PPT 中有嵌入的视频区域，视频播放时每一帧都可能被 ffmpeg 的 scene detection 误判为 PPT 切换，产生大量相似截图。

**修复**：
- **Method A（新视频）**：ffmpeg filter 改为 `select='gt(scene,0.3)*(isnan(prev_selected_t)+gte(t-prev_selected_t,3))'`，提高阈值并限制最短间隔 3 秒。`isnan(prev_selected_t)` 是必须的——首帧无上一帧时该值为 NAN，不加此判断会导致零帧被选中（所有视频 0 截图）。已在 `batch_transcribe.py` 中更新。
- **Method B（已有视频）**：对截图数 > 100 的视频，使用 pHash 感知哈希去重，删除相似帧并同步更新 `scene_log.txt`，再重新生成 Markdown。

**已有视频的补救**：运行 `fix_embedded_video.py`，它会自动扫描 output/videos/ 下截图 > 100 张的视频，执行 pHash 去重 + 重新生成 Markdown，无需重新截图或转录。

### info.json 字段为 null 导致崩溃

**症状**：generate_markdown.py 报 `AttributeError: 'NoneType' object has no attribute 'strip'`。

**原因**：`info.json` 中 `name` 或 `intro` 字段值为 `null`（非字符串），直接 `.strip()` 报错。

**修复**：`generate_markdown.py` 中已改为 `(info.get("name") or "").strip()`，兼容 null 值。

### 视频无可识别语音（0 segment）

**症状**：generate_markdown.py 报 `JSONDecodeError: Expecting value: line 1 column 1`，transcript 生成失败。

**原因**：whisperX 未识别到任何语音（静音视频、纯音乐、或转录失败），clip.json 中 segments 为空列表，向 Claude 发送空转录内容时返回空字符串，JSON 解析崩溃。

**修复**：`generate_markdown.py` 已加空 segment 保护——检测到 0 句子时跳过 Claude 调用，仅输出幻灯片截图并注明"无转录内容"。

### ffmpeg filter 缺少 isnan 导致 0 截图

**症状**：视频处理完成但 slides/ 目录为空（0 张截图）。

**原因**：早期使用的 filter `select='gt(scene,0.3)*gte(t-prev_selected_t,3)'` 有 bug——`prev_selected_t` 在首帧时为 NAN，`gte(t-NAN, 3)` 恒为 0，导致没有任何帧被选中。

**修复**：必须使用 `select='gt(scene,0.3)*(isnan(prev_selected_t)+gte(t-prev_selected_t,3))'`，`isnan()` 检查确保首帧能被选中。

**已有视频的补救**：运行 `fix_empty_slides.py`，重新跑 ffmpeg 截图 + 重新生成 Markdown，无需重新转录。
