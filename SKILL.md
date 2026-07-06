---
name: youtube-summary
description: >
  Extract subtitles & frames from YouTube/Bilibili, generate structured 9-section summary.
  Supports: text mode (subtitles → LLM) and visual mode (ffmpeg + Gemini/GPT-5.5).
  Two visual tiers: --deep (GPT-5.5, ~$0.18) or --quick (Gemini, free).
  Triggers: ANY URL containing youtube.com, youtu.be, bilibili.com, or b23.tv.
  Also: "summarize this video / 总结这个视频 / 帮我看看 / 获取字幕 / 视频总结 / 提取字幕".
---

# Video Summarizer (YouTube + Bilibili)

Detect platform → download subtitles + video → scene detection → frame description → LLM summary.

Supports **macOS/Linux (bash)** and **Windows (PowerShell 5.1+)** .

---

## Execution Modes

| Mode | Trigger | Pipeline | Cost | Use Case |
|------|---------|----------|------|----------|
| **Text** (default, when subtitles exist) | `总结这个视频 <URL>` | Subs → clean → LLM 9-section summary | Free (local LLM) | Videos with spoken content |
| **Visual Deep** | `--deep` or explicit request | Scene detect → GPT-5.5 direct 9-section | ~$0.18/video | Design analysis, inspiration |
| **Visual Quick** | `--quick` or default when no subs | Scene detect → Gemini describe + LLM summary | Free | Quick browsing, bulk scanning |

**Auto-detection**: If subtitles exist → Text mode. If no subtitles → Visual Quick mode. Add `--deep` to force GPT-5.5.

---

## Prerequisites (One-Time Setup)

### Required Tools

- `yt-dlp` — subtitle/video download: `pip install yt-dlp`
- `ffmpeg` — scene detection + frame extraction (usually pre-installed)
- `python3` — orchestration scripts

### B站 Cookies (Bilibili only)

B站 needs login cookies. On Windows, Chrome DPAPI decryption is broken in yt-dlp.

**Setup**: Install Chrome extension **"Get cookies.txt LOCALLY"** , open bilibili.com, Export → save as `%USERPROFILE%\bilibili_cookies.txt` (Windows) or `~/bilibili_cookies.txt` (Unix). Re-export every ~30 days.

See `references/cookies-setup.md` for detailed steps.

### API Keys

For visual analysis, configure environment variables (or hardcode in script):

```bash
# Gemini (free, 1500 req/day, for quick mode)
# Get key at: https://aistudio.google.com/apikey
GEMINI_API_KEY="your-gemini-key-here"

# Leihuo GPT-5.5 (for deep mode, ~$0.18/video)
# Get key + base URL from your Leihuo gateway console
LEIHUO_API_KEY="your-leihuo-key-here"
LEIHUO_API_BASE="https://ai.leihuo.netease.com/v1/chat/completions"
```

---

## Full Pipeline

### Step 1 — Download Subtitles + Low-Res Video

```powershell
# Windows PowerShell
$URL = "<user-provided URL>"
$BV = ($URL -split '/')[-1] -replace '\?.*',''
$TMP = Join-Path $env:TEMP "yt_summary_$BV"
New-Item -ItemType Directory -Path $TMP -Force | Out-Null

# Detect platform
if ($URL -match 'bilibili\.com|b23\.tv') {
    $PLATFORM = "bilibili"
    $COOKIES = "--cookies $env:USERPROFILE\bilibili_cookies.txt"
} else {
    $PLATFORM = "youtube"
    $COOKIES = ""
}

# Download subtitles (try ai-zh then zh-Hans then zh then en)
$LANG = "ai-zh"
yt-dlp --no-update $COOKIES.Split(' ') --write-subs --sub-langs $LANG --skip-download -o "$TMP\subs" $URL 2>&1 | Out-Null
$SUB = Get-ChildItem $TMP -Filter "*.$LANG.*" -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName

# Fallback languages if ai-zh not available
if (-not $SUB) {
    foreach ($fallback in @('zh-Hans', 'zh', 'en')) {
        yt-dlp --no-update $COOKIES.Split(' ') --write-subs --sub-langs $fallback --skip-download -o "$TMP\subs" $URL 2>&1 | Out-Null
        $SUB = Get-ChildItem $TMP -Filter "*.$fallback.*" -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName
        if ($SUB) { $LANG = $fallback; break }
    }
}

# Get metadata via B站 API or yt-dlp
if ($PLATFORM -eq "bilibili") {
    $TITLE = (Invoke-RestMethod "https://api.bilibili.com/x/web-interface/view?bvid=$BV").data.title
    $UPLOADER = (Invoke-RestMethod "https://api.bilibili.com/x/web-interface/view?bvid=$BV").data.owner.name
} else {
    $meta = yt-dlp --no-update --dump-json $URL 2>$null | ConvertFrom-Json
    $TITLE = $meta.title; $UPLOADER = $meta.uploader
}

# Download low-res video (always needed for scene detection)
yt-dlp --no-update $COOKIES.Split(' ') -f "worst[height<=480]" -o "$TMP\video.%(ext)s" $URL 2>&1
$VIDEO = Get-ChildItem $TMP -Filter "video.*" | Select-Object -First 1 -ExpandProperty FullName
```

### Step 2 — Scene Detection + Frame Extraction

```python
# scene_detect.py (cross-platform)
import subprocess, os, re

TMP = os.environ['TMP_DIR']  # set by caller
VIDEO = os.path.join(TMP, 'video.mp4')
FRAMES = os.path.join(TMP, 'frames')
os.makedirs(FRAMES, exist_ok=True)

# ffmpeg scene detection: sensitivity 0.4
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

# Dedup close scenes (< 1.5s apart), cap at 12
filtered = []
for t in scene_times:
    if not filtered or t - filtered[-1] >= 1.5:
        filtered.append(t)
filtered = filtered[:12]

# Extract one frame per scene
for i, t in enumerate(filtered):
    out = os.path.join(FRAMES, f'scene_{i:02d}.jpg')
    subprocess.run(['ffmpeg', '-y', '-ss', str(t), '-i', VIDEO,
                    '-vframes', '1', '-q:v', '2', out], capture_output=True)

# Print scene times for next step
for i, t in enumerate(filtered):
    print(f"SCENE|{i}|{t}|{int(t//60)}:{int(t%60):02d}")
```

### Step 3 — Generate Summary

Choose mode based on subtitle availability and user preference:

#### Mode A: Text (subtitles exist)

Clean the subtitle, read `references/output-formats.md` for templates, then generate a 9-section summary using the transcript as input. Follow the Chain of Density rules and attribution policy.

#### Mode B: Visual Quick (no subs, default) — Gemini 2.5 Flash

```python
# Merge scene detection script with Gemini API call
import base64, json, urllib.request, os

API_KEY = os.environ.get('GEMINI_API_KEY')
FRAMES = os.path.join(TMP, 'frames')
frame_files = sorted([f for f in os.listdir(FRAMES) if f.endswith('.jpg')])

# Build Gemini request with all frames
parts = [{"text": f"Describe each frame from a game video. Title: {TITLE}. For each frame: location type, season/weather, architecture style, composition, visual mood. Reply in Chinese, one paragraph per frame."}]
for fname in frame_files:
    with open(os.path.join(FRAMES, fname), 'rb') as f:
        b64 = base64.b64encode(f.read()).decode('utf-8')
    parts.append({"text": f"Frame {fname}:"})
    parts.append({"inline_data": {"mime_type": "image/jpeg", "data": b64}})

url = f"https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key={API_KEY}"
payload = {"contents": [{"parts": parts}], "generationConfig": {"temperature": 0.3, "maxOutputTokens": 4096}}
req = urllib.request.Request(url, data=json.dumps(payload).encode('utf-8'), headers={"Content-Type": "application/json"})

with urllib.request.urlopen(req, timeout=180) as resp:
    gemini_result = json.loads(resp.read().decode('utf-8'))
frame_descriptions = gemini_result["candidates"][0]["content"]["parts"][0]["text"]

# Then pass frame_descriptions to GPT-5.5 or local LLM for 9-section summary
# Use the prompt template from Step 3C below
```

#### Mode C: Visual Deep (--deep or explicit) — GPT-5.5 Direct

```python
import base64, json, urllib.request, os

LEIHUO_KEY = os.environ.get('LEIHUO_API_KEY')
LEIHUO_URL = os.environ.get('LEIHUO_API_BASE', 'https://ai.leihuo.netease.com/v1/chat/completions')

# Build vision message with all frames (GPT-5.5 supports image_url)
content = [{"type": "text", "text": f"Analyze these keyframes from a game video titled '{TITLE}' by {UPLOADER}. For each frame describe: location, season, architecture, composition. Then generate a 9-section summary: 章节概览, 总体摘要, 话题章节, 新颖观点, 反直觉观点, 核心张力, 方法论, 关键数据. Reply in Chinese."}]
for fname in sorted(os.listdir(FRAMES)):
    with open(os.path.join(FRAMES, fname), 'rb') as f:
        b64 = base64.b64encode(f.read()).decode('utf-8')
    content.append({"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{b64}"}})

payload = {
    "model": "gpt-5.5",
    "messages": [{"role": "user", "content": content}],
    "max_tokens": 6000, "temperature": 0.5
}
req = urllib.request.Request(LEIHUO_URL, data=json.dumps(payload).encode('utf-8'),
    headers={"Authorization": f"Bearer {LEIHUO_KEY}", "Content-Type": "application/json"})
with urllib.request.urlopen(req, timeout=180) as resp:
    result = json.loads(resp.read().decode('utf-8'))
summary = result["choices"][0]["message"]["content"]
# Cost: input=~2000 completion=~3500 → ~$0.18
```

#### 9-Section Summary Prompt Template

All modes use this structure. Read `references/output-formats.md` for exact section headers.

```
Analyze this video. Title: {TITLE}, Uploader: {UPLOADER}.

{Subtitle transcript OR frame descriptions}

Generate a 9-section structured summary in Chinese:
1. 章节概览 — timestamped chapter table
2. 总体摘要 — 3-5 sentence synthesis + 3-5 key highlights (Chain of Density)
3. 话题章节 — 4-8 major topics, each 3-5 bullet points (2-3 sentences each)
4. 关键引用 — verbatim quotes in blockquotes (skip if no spoken content)
5. 新颖观点 — non-mainstream insights, 2-3 sentences each
6. 反直觉观点 — "Common belief: X → Actual: Y" format
7. 核心张力 — opposing forces, 2-3 sentences per side
8. 方法论 — reusable design frameworks, sub-bullets for steps
9. 关键数据 — specific numbers only, never fabricate

Tags: #关卡设计 #场景叙事 #{game_name}

Omit any section without relevant content. Never fabricate. Voice: state ideas directly, no "X says/believes" prefixes.
```

---

## Step 4 — Save Output

Save two files to the output directory (default: current directory, or `YOUTUBE_SUBTITLES_DIR`):

1. `{title}.md` — full timestamped transcript (if subtitles exist)
2. `{title}-summary.md` — 9-section structured summary (always generated)

Obsidian-style frontmatter with `title`, `source`, `author`, `published`, `tags`, `type: source`.

---

## Quick Reference: When to Use Which Mode

```
Has subtitles? ──Yes──→ Text (local LLM, free, 9-section summary)
     │
     No
     │
     ├── "fast/quick/浏览" → Visual Quick (Gemini free → GPT-5.5 summary ~$0.12)
     │
     └── "deep/detailed/深度" → Visual Deep (GPT-5.5 direct, ~$0.18)
```

Cost breakdown per video:
- Text mode: $0 (local)
- Visual Quick: $0.12 (Gemini free + GPT-5.5 for final summary)
- Visual Deep: $0.18 (GPT-5.5 all-in-one)

---

## Error Handling

| Error | Action |
|-------|--------|
| yt-dlp not found | Tell user: `pip install yt-dlp` |
| B站 cookies missing | Tell user: run Step 0 in `references/cookies-setup.md` |
| B站 cookies expired | Tell user: re-export from "Get cookies.txt LOCALLY" |
| No subtitles + no video downloaded | Both modes failed. Check URL or network. |
| No subtitles found | Auto-switch to Visual Quick mode (default) |
| Gemini 429 (rate limit) | Tell user: free quota exhausted, wait or switch to `--deep` |
| GPT-5.5 error | Retry once; if persists, fall back to Visual Quick |
| Scene detection yields 0 frames | Use fixed-interval fallback: extract 1 frame every 10% of duration |
| ffmpeg not found | Install ffmpeg: `winget install ffmpeg` (Win) or `brew install ffmpeg` (Mac) |
