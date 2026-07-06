---
name: bili-youtube-summary
description: >
  Extract subtitles, danmaku & frames from YouTube/Bilibili, generate structured summary.
  Three output tiers: L1 overview (速览), L2 9-section deep analysis (深度), L2.1 +design dimensions (设计).
  Visual mode via ffmpeg + Gemini (free) or GPT-5.5 (~$0.18).
  Triggers: ANY URL containing youtube.com, youtu.be, bilibili.com, or b23.tv.
  Also: "总结/看看/分析/设计/灵感/拆解/竞品/启发 这个视频".
---

# Video Summarizer (YouTube + Bilibili)

Detect platform → download subtitles + danmaku + video → scene detection → LLM summary.

---

## Intent Routing — 先判断输出层级

**在开始任何处理前，先根据用户措辞确定输出层级：**

| 层级 | 触发词 | 输出内容 | 适用场景 |
|------|--------|---------|---------|
| **L1 速览** | `总结` `看看` `讲了什么` `获取字幕` `提取字幕` | 标题 + 时间轴 + 关键观察 + 弹幕高光 + 证据档位 | 快速扫片、大量浏览 |
| **L2 深度** | `深度` `详细` `全面总结` `--deep` | L1 + 9 段结构化总结 | 单视频深入理解 |
| **L2.1 设计** | `设计` `灵感` `拆解` `竞品` `启发` `落地` `策划` | L2 + 10 维设计注意点 + 参考知识 + 风险清单 | 策划视角的竞品分析和设计参考 |

**默认规则**：用户只说"总结"且无深度/设计关键词 → L1。包含视频链接且无任何意图词 → L1。

---

## Evidence Tier — 证据档位

每个输出必须标注证据档位，让读者知道结论的可靠程度：

| 档位 | 条件 | 示例标注 |
|------|------|---------|
| **A** | 完整视频 + 字幕 + 多帧画面 | `[证据: A 档 — 完整视频+字幕+12帧]` |
| **B** | 低清视频 + 封面/标题/弹幕，画质有限 | `[证据: B 档 — 480p视频+弹幕]` |
| **C** | 仅标题/封面/元数据 | `[证据: C 档 — 仅元数据]` |
| **D** | 链接不可访问 | `[证据: D 档 — 需本地文件]` |

---

## Output Templates

### L1 速览

```markdown
**[证据: X 档 — 具体说明]**

**{标题}**
UP主: {作者} | 时长: {时长} | 播放: {播放量}

**时间轴速览**
- 00:00-XX:XX：...
- XX:XX-XX:XX：...

**关键观察**
- 画面确认：...
- 字幕/口播：...

{弹幕高光 — B站专属：Top 5 密度段 + 代表弹幕}

**不确定项**
- ...
```

### L2 深度

L1 全部内容 + 以下 9 段：

```markdown
## 章节概览
| 时间 | 章节 | 内容 |
## 总体摘要
(Chain of Density, 3-5句 + 3-5亮点)
## 话题章节
(4-8 话题, 每话题 3-5 条 2-3 句)
## 关键引用
(口播原文 blockquote, 跳过无口播视频)
## 弹幕反应
(B站: 密度段 + 情绪聚类 + 代表弹幕)
## 新颖观点
## 反直觉观点
(Common belief → Actual claim 格式)
## 核心张力
(对立力量, 2-3句/面)
## 方法论
(可复用框架)
## 关键数据
(仅精确数字, 不编造)
```

### L2.1 设计

L2 全部内容 + 以下 3 段（从 10 维中选 3-5 个最相关的，每节绑定视频画面证据）：

```markdown
## 和游戏设计相关的注意点
从以下维度选 3-5 个，每个 2-3 句，绑定视频中的具体观察：

| 维度 | 关注点 |
|------|--------|
| 玩法价值 | 动机、循环、目标 |
| 交互成本 | 入口、层级、误触、学习成本 |
| 表现质量 | 动作、镜头、灯光、音效、氛围 |
| NPC/世界参与 | 动态行为、环境反馈、生活感 |
| 社交传播 | 截图、录制、滤镜、UGC传播点 |
| 边界规则 | 打断、碰撞、穿模、遮挡、多人入镜 |
| 技术风险 | 性能、LOD、移动端、联网同步 |
| 运营内容 | 打卡点、活动模板、节日主题 |
| 可复用资产 | 动作/镜头/UI 能否沉淀为工具链 |
| 负反馈控制 | 撤销、跳过、关闭、屏蔽 |

## 可供参考的知识
从视频观察延伸出的设计模式、检查清单，不是直接方案。做成"可参考的知识点/检查项"。

## 风险与待验证
不能确认的推测、画质限制导致的模糊判断、需要进一步调研的问题。
```

---

## Pipeline (Data Extraction)

以下 Steps 1-3 为数据提取，所有层级共用。Step 4 按层级选择输出模板。

### Step 1 — Download Subtitles + Danmaku + Low-Res Video

```powershell
$URL = "<user-provided URL>"
$BV = ($URL -split '/')[-1] -replace '\?.*',''
$TMP = Join-Path $env:TEMP "yt_summary_$BV"
New-Item -ItemType Directory -Path $TMP -Force | Out-Null

if ($URL -match 'bilibili\.com|b23\.tv') {
    $PLATFORM = "bilibili"
    $COOKIES = "--cookies $env:USERPROFILE\bilibili_cookies.txt"
} else {
    $PLATFORM = "youtube"
    $COOKIES = ""
}

# Subtitles
$LANG = "ai-zh"
yt-dlp --no-update $COOKIES.Split(' ') --write-subs --sub-langs $LANG --skip-download -o "$TMP\subs" $URL 2>&1 | Out-Null
$SUB = Get-ChildItem $TMP -Filter "*.$LANG.*" -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName
if (-not $SUB) {
    foreach ($fallback in @('zh-Hans', 'zh', 'en')) {
        yt-dlp --no-update $COOKIES.Split(' ') --write-subs --sub-langs $fallback --skip-download -o "$TMP\subs" $URL 2>&1 | Out-Null
        $SUB = Get-ChildItem $TMP -Filter "*.$fallback.*" -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName
        if ($SUB) { $LANG = $fallback; break }
    }
}

# Metadata + danmaku
if ($PLATFORM -eq "bilibili") {
    $META = Invoke-RestMethod "https://api.bilibili.com/x/web-interface/view?bvid=$BV"
    $TITLE = $META.data.title
    $UPLOADER = $META.data.owner.name
    $CID = $META.data.cid
    $DM_COUNT = $META.data.stat.danmaku
    if ($CID) {
        $DM_RAW = (Invoke-WebRequest "https://api.bilibili.com/x/v1/dm/list.so?oid=$CID").Content
        $regex = [regex]'<d p="([^"]+)"[^>]*>([^<]+)</d>'
        $dm_entries = @()
        foreach ($m in $regex.Matches($DM_RAW)) {
            $p = $m.Groups[1].Value -split ','
            $dm_entries += @{time=[float]$p[0]; text=$m.Groups[2].Value}
        }
        $buckets = @{}
        foreach ($d in $dm_entries) {
            $b = [math]::Floor($d.time / 10) * 10
            if (-not $buckets[$b]) { $buckets[$b] = @() }
            $buckets[$b] += $d.text
        }
        $ranked = $buckets.Keys | Sort-Object { -$buckets[$_].Count } | Select-Object -First 8
        foreach ($k in $ranked) {
            $samples = ($buckets[$k] | Sort-Object | Get-Unique | Select-Object -First 5) -join " ; "
            $mm = [math]::Floor($k / 60); $ss = $k % 60
            Write-Output "DMBUCKET|$k|$($mm):$($ss.ToString('00'))|$($buckets[$k].Count)|$samples"
        }
    }
} else {
    $meta = yt-dlp --no-update --dump-json $URL 2>$null | ConvertFrom-Json
    $TITLE = $meta.title; $UPLOADER = $meta.uploader
}

# Low-res video
yt-dlp --no-update $COOKIES.Split(' ') -f "worst[height<=480]" -o "$TMP\video.%(ext)s" $URL 2>&1
$VIDEO = Get-ChildItem $TMP -Filter "video.*" | Select-Object -First 1 -ExpandProperty FullName
```

### Step 2 — Scene Detection

```python
import subprocess, os, re

TMP = os.environ['TMP_DIR']
VIDEO = os.path.join(TMP, 'video.mp4')
FRAMES = os.path.join(TMP, 'frames')
os.makedirs(FRAMES, exist_ok=True)

result = subprocess.run([
    'ffmpeg', '-i', VIDEO,
    '-vf', "select='gt(scene\\,0.4)',showinfo",
    '-vsync', 'vfr', '-f', 'null', '-'
], capture_output=True, text=True)

scene_times = []
for line in result.stderr.split('\n'):
    m = re.search(r'pts_time:([\d.]+)', line)
    if m and float(m.group(1)) > 0:
        scene_times.append(float(m.group(1)))

filtered = []
for t in scene_times:
    if not filtered or t - filtered[-1] >= 1.5:
        filtered.append(t)
filtered = filtered[:12]

for i, t in enumerate(filtered):
    out = os.path.join(FRAMES, f'scene_{i:02d}.jpg')
    subprocess.run(['ffmpeg', '-y', '-ss', str(t), '-i', VIDEO,
                    '-vframes', '1', '-q:v', '2', out], capture_output=True)

for i, t in enumerate(filtered):
    print(f"SCENE|{i}|{t}|{int(t//60)}:{int(t%60):02d}")
```

### Step 3 — Generate Summary per Tier

#### L1: Local LLM / Gemini Quick

Use subtitle transcript (if exists) or Gemini frame descriptions (if no subs). Prompt for a concise overview: title + timeline + key observations + danmaku highlights. No deep analysis sections.

#### L2: GPT-5.5 or Local LLM

Generate full 9-section output using the prompt template below. If no subtitles, use Gemini frame descriptions as input.

#### L2.1: GPT-5.5

Generate L2 + design analysis sections. Requires GPT-5.5 via Leihuo gateway. Use combined input: subtitles (or Gemini frame descriptions) + danmaku data + metadata.

**Prompt template for L2+:**

```
Analyze this video. Title: {TITLE}, Uploader: {UPLOADER}.
Danmaku: {DM_COUNT} total (B站 only).

{Subtitle transcript OR frame descriptions}

{Danmaku density buckets — if B站}

Generate output at tier {L2 or L2.1} following the output templates.
For L2.1: select 3-5 most relevant dimensions from the 10-dim checklist below.
Every observation must be tied to specific video evidence (timestamp or frame).
Use "画面显示" for visual evidence, "口播提到" for spoken content, "弹幕集中" for danmaku patterns.
Never write "should do" — write "note that" or "may want to consider".
```

### Step 4 — Determine Evidence Tier

After data extraction, assign the evidence tier:
- Got video frames + subtitles + danmaku → A
- Got low-res video + metadata → B
- Only metadata (title/cover/description) → C
- Nothing accessible → D (ask for local file)

Include in output header.

---

## Prerequisites (One-Time Setup)

### Required Tools
- `yt-dlp`: `pip install yt-dlp`
- `ffmpeg`: pre-installed on most systems
- `python3`: standard library only

### B站 Cookies
Install Chrome extension **"Get cookies.txt LOCALLY"** , open bilibili.com, Export → save as `%USERPROFILE%\bilibili_cookies.txt` (Windows) or `~/bilibili_cookies.txt` (Unix). Re-export ~30 days. See `references/cookies-setup.md`.

### API Keys
> Visual mode only; L1/L2 text mode (subtitles exist) works without keys.

| Key | Source | Cost | Use |
|-----|--------|------|-----|
| `GEMINI_API_KEY` | https://aistudio.google.com/apikey | Free 1500/day | Frame descriptions |
| `LEIHUO_API_KEY` | Leihuo gateway console | ~$0.18/video | L2/L2.1 GPT-5.5 |
| `LEIHUO_API_BASE` | Leihuo gateway | — | `https://ai.leihuo.netease.com/v1/chat/completions` |

Set via `setx KEY "value"` (Win) or `export KEY="value"` (Unix).

---

## Quick Reference

```
User intent:
  "总结/看看" ──────────→ L1 速览 (free, fast)
  "深度/详细" ──────────→ L2 9-section (~$0.12-0.18)
  "设计/灵感/拆解/竞品" → L2.1 +10 dimensions (~$0.18)
```

Cost per video:
- L1 text: $0 (local LLM)
- L1 visual: $0 (Gemini free)
- L2: ~$0.12 (Gemini + GPT-5.5) or ~$0.18 (GPT-5.5 direct)
- L2.1: ~$0.18 (GPT-5.5 direct)

---

## Error Handling

| Error | Action |
|-------|--------|
| yt-dlp not found | `pip install yt-dlp` |
| B站 cookies missing/expired | See `references/cookies-setup.md` |
| No subtitles found | Auto-use frame descriptions as input |
| Gemini 429 | Free quota exhausted; wait or use `--deep` |
| GPT-5.5 error | Retry once; fallback to L1 |
| 0 scene frames | Fallback: 1 frame / 10% of duration |
| ffmpeg not found | `winget install ffmpeg` (Win) / `brew install ffmpeg` (Mac) |

---

## Output Files

- `{title}.md` — transcript (if subs) + danmaku table (if B站)
- `{title}-summary.md` — structured summary (L1/L2/L2.1)

Obsidian-style frontmatter: `title`, `source`, `author`, `published`, `tags`, `type: source`.
