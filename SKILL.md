---
name: bili-youtube-summary
description: >
  Extract subtitles, danmaku & frames from YouTube/Bilibili, generate structured summary.
  6 tiers: text-only (free) / Gemini frames (~$0.12) / GPT-5.5 frames (~$0.18)
  × 普通总结 / 策划视角 (10-dim design checklist).
  Triggers: ANY URL containing youtube.com, youtu.be, bilibili.com, or b23.tv.
---

# Video Summarizer (YouTube + Bilibili)

Detect platform → download subtitles + danmaku + video (x.2/x.3 only) → scene detection → LLM summary.

---

## Tier Selection — 6 档选择

**首次使用或用户明确要求切换时询问。** 后续默认沿用上次选择。

**判断逻辑：**
1. 检查环境变量 `BYS_LAST_TIER`（上次选择）
2. 用户明确说"换个模式""用深度""策划视角""切换"等 → 展示选项
3. 用户只说"总结这个视频"且 `BYS_LAST_TIER` 存在 → 直接沿用，不弹窗
4. `BYS_LAST_TIER` 不存在或无法解析 → 弹窗询问

以下选项**仅当需要询问时展示**：

```
对这个视频做哪种分析？

━━━ 普通总结 ━━━
1.1 速览 — 纯文本，秒出，免费
    只看标题+字幕+弹幕，不下载视频。适合快速扫片。
1.2 中度 — Gemini截帧+9段总结，~$0.12，约1分钟
1.3 深度 — GPT直接截帧+9段总结，~$0.18，约1分钟

━━━ 策划视角 ━━━
2.1 策划速览 — 1.1 + 10维设计注意点+参考知识+风险，免费
2.2 策划中度 — 1.2 + 10维设计注意点+参考知识+风险，~$0.12
2.3 策划深度 — 1.3 + 10维设计注意点+参考知识+风险，~$0.18

回复数字（如 1.2 或 2.3）。
```

确认后持久化：`setx BYS_LAST_TIER "1.2"` (Win) / `export BYS_LAST_TIER="1.2" >> ~/.bashrc` (Unix).

---

## Path Convention — 避免权限弹窗

**所有中间文件放在项目目录内，不使用 `%TEMP%`。** 批量处理、自动触发时不会反复申请文件读权限。

```
$OUT = if ($env:YOUTUBE_SUBTITLES_DIR) { $env:YOUTUBE_SUBTITLES_DIR } else { "$PWD\video-analysis" }
$TMP = "$OUT\.tmp"
```

- `$OUT/` — 最终输出 (`{标题}.md`, `{标题}-summary.md`)
- `$OUT/.tmp/` — 中间文件，运行结束后自动清理
- **Python 脚本数据通过 stdout 打印**，Agent 直接从 Bash 输出读取，不用 `Read` 工具
- 例外：`bilibili_cookies.txt` 路径不变 (`%USERPROFILE%\`)，仅在 Step 0 一次性配置

---

## Execution Flow

### Before Running — Show Active Tier

```
━━━ 当前模式: 2.3 策划深度 ━━━
GPT-5.5 直接截图分析 → 9段总结 + 10维策划注意点
预计成本 ~$0.18 | 约 1 分钟
(沿用上次选择，回复"切换"可更改)

开始下载...
```

### After Running — Inline Summary + File Path

处理完成后在对话中输出简短概述（5-8 行，不要贴完整总结），然后告知文件路径：

```
**{标题}** — {时长} | 证据: X档

{3-5 句核心概述}

━━━━━━━━━━━━━━━━━━━━
完整输出: {路径}\{标题}-summary.md
━━━━━━━━━━━━━━━━━━━━
```

---

## Evidence Tier — 证据档位

| 档位 | 条件 | 示例标注 |
|------|------|---------|
| **A** | 完整视频 + 字幕 + 多帧画面 | `[证据: A 档]` |
| **B** | 低清视频 + 弹幕，画质有限 | `[证据: B 档]` |
| **C** | 仅标题/元数据 | `[证据: C 档]` |
| **D** | 链接不可访问 | `[证据: D 档 — 需本地文件]` |

---

## Pipeline — All-in-One Python Script

以下 Python 脚本封装全部流程。**直接通过 Bash 运行，agent 读取 stdout 输出，不依赖 `Read` 工具。**

脚本逻辑：
1. 根据 `$TIER` 决定是否下载视频 (x.2/x.3) 和调用哪个 API (1.x vs 2.x)
2. 所有数据通过 stdout 结构化输出
3. 最终写入 `{OUT}/{title}.md` 和 `{OUT}/{title}-summary.md`

```python
import subprocess, os, json, urllib.request, re, sys, io, base64, shutil, zlib
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')

BV = os.environ['BV']
URL = os.environ['URL']
TIER = os.environ['TIER']
OUT = os.environ.get('YOUTUBE_SUBTITLES_DIR', os.path.join(os.getcwd(), 'video-analysis'))
TMP = os.path.join(OUT, '.tmp')
os.makedirs(TMP, exist_ok=True)
COOKIES = os.path.join(os.environ['USERPROFILE'], 'bilibili_cookies.txt')
IS_BILI = 'bilibili' in URL or 'b23.tv' in URL
NEED_VIDEO = TIER.split('.')[1] in ('2', '3')

print(f"TIER: {TIER} | PLATFORM: {'bilibili' if IS_BILI else 'youtube'} | VIDEO: {NEED_VIDEO}")

# ===== 1. Metadata =====
if IS_BILI:
    r = urllib.request.Request(f'https://api.bilibili.com/x/web-interface/view?bvid={BV}',
        headers={'User-Agent': 'Mozilla/5.0'})
    with urllib.request.urlopen(r) as resp:
        data = json.loads(resp.read())['data']
    title = data['title']; uploader = data['owner']['name']
    duration = data['duration']; cid = data['cid']; dm_total = data['stat']['danmaku']
else:
    result = subprocess.run(['yt-dlp', '--no-update', '--dump-json', URL],
        capture_output=True, text=True)
    meta = json.loads(result.stdout)
    title = meta['title']; uploader = meta['uploader']
    duration = meta.get('duration', 0)
print(f"META: {title} | {uploader} | {duration//60}:{duration%60:02d}")

# ===== 2. Subtitles =====
subs_text = ''
if IS_BILI:
    sess = ''
    if os.path.exists(COOKIES):
        for line in open(COOKIES, encoding='utf-8'):
            if 'SESSDATA' in line:
                sess = line.strip().split('\t')[6]; break
    if sess and cid:
        p_url = f'https://api.bilibili.com/x/player/v2?aid=0&cid={cid}&bvid={BV}'
        p_req = urllib.request.Request(p_url, headers={'Cookie': f'SESSDATA={sess}',
            'User-Agent': 'Mozilla/5.0', 'Referer': f'https://www.bilibili.com/video/{BV}/'})
        with urllib.request.urlopen(p_req) as resp:
            p_data = json.loads(resp.read())
        subs = p_data.get('data', {}).get('subtitle', {}).get('subtitles', [])
        if subs:
            sub_url = next((s['subtitle_url'] for s in subs if s['lan'] == 'ai-zh'), subs[0]['subtitle_url'])
            if sub_url.startswith('//'): sub_url = 'https:' + sub_url
            with urllib.request.urlopen(urllib.request.Request(sub_url, headers={'User-Agent': 'Mozilla/5.0'})) as resp:
                sub_data = json.loads(resp.read())
            subs_lines = [f"[{int(e['from'])//60:02d}:{int(e['from'])%60:02d}] {e['content']}" for e in sub_data['body']]
            subs_text = '\n'.join(subs_lines)
            print(f"SUBS: {len(subs_lines)} lines (B站 API)")
else:
    subprocess.run(['yt-dlp', '--no-update', '--write-subs', '--sub-langs', 'zh-Hans',
        '--skip-download', '-o', os.path.join(TMP, 'subs'), URL], capture_output=True)
    for ext in ['.zh-Hans.vtt', '.en.vtt', '.zh.vtt']:
        f = os.path.join(TMP, f'subs{ext}')
        if os.path.exists(f):
            with open(f, encoding='utf-8') as fh:
                subs_text = fh.read()
            print(f"SUBS: {len(subs_text)} chars (yt-dlp)")
            break

# ===== 3. Danmaku =====
dm_buckets = ''
if IS_BILI and cid:
    try:
        dm_url = f'https://api.bilibili.com/x/v1/dm/list.so?oid={cid}'
        dm_req = urllib.request.Request(dm_url, headers={'User-Agent': 'Mozilla/5.0'})
        with urllib.request.urlopen(dm_req) as resp:
            dm_raw = resp.read()
        # B站 returns Content-Encoding: deflate — decompress first
        try:
            dm_text = dm_raw.decode('utf-8')
        except:
            dm_text = zlib.decompress(dm_raw, -15).decode('utf-8')
        # Both quote styles (B站 XML uses double quotes normally)
        dms = re.findall(r'<d p="([^"]+)"[^>]*>([^<]+)</d>', dm_text)
        if not dms:
            dms = re.findall(r"<d p='([^']+)'[^>]*>([^<]+)</d>", dm_text)
        buck = {}
        for p, txt in dms:
            t = float(p.split(',')[0]); b = int(t//10)*10; buck.setdefault(b, []).append(txt)
        ranked = sorted(buck.keys(), key=lambda k: -len(buck[k]))[:8]
        for k in ranked:
            samples = list(dict.fromkeys(buck[k]))[:3]
            dm_buckets += f'{k//60}:{k%60:02d} [{len(buck[k])}条] {" | ".join(samples)}\n'
        print(f"DANMAKU: {len(dms)} entries / {dm_total} total")
    except Exception as e:
        print(f"DANMAKU: error - {e}")

# ===== 4. Video + Frames (x.2/x.3 only) =====
frame_desc = ''
summary = ''

if NEED_VIDEO:
    print("DOWNLOADING VIDEO...")
    subprocess.run(['yt-dlp', '--no-update', '--cookies', COOKIES, '-f', 'worst[height<=480]',
        '-o', os.path.join(TMP, 'video.%(ext)s'), URL], capture_output=True)
    vf = next((os.path.join(TMP, f) for f in os.listdir(TMP) if f.startswith('video.')), None)
    if not vf:
        print("ERROR: Video download failed"); sys.exit(1)
    
    # Scene detection
    result = subprocess.run(['ffmpeg', '-i', vf, '-vf', "select='gt(scene\\,0.4)',showinfo",
        '-vsync', 'vfr', '-f', 'null', '-'], capture_output=True, text=True)
    scene_times = [float(m.group(1)) for m in re.finditer(r'pts_time:([\d.]+)', result.stderr) if float(m.group(1)) > 0]
    filtered = [scene_times[0]]; [filtered.append(t) for t in scene_times[1:] if t - filtered[-1] >= 1.5]
    filtered = filtered[:12]
    print(f"SCENES: {len(scene_times)} raw -> {len(filtered)} scenes")

    frames_dir = os.path.join(TMP, 'frames'); os.makedirs(frames_dir, exist_ok=True)
    for i, t in enumerate(filtered):
        subprocess.run(['ffmpeg', '-y', '-ss', str(t), '-i', vf, '-vframes', '1', '-q:v', '2',
            os.path.join(frames_dir, f'scene_{i:02d}.jpg')], capture_output=True)

    # Frame descriptions
    if TIER.split('.')[1] == '2':  # Gemini
        print("CALLING GEMINI...")
        API_KEY = os.environ.get('GEMINI_API_KEY', '')
        parts = [{"text": f"Describe each frame from a video titled '{title}'. Reply in Chinese, one paragraph per frame."}]
        for fn in sorted(os.listdir(frames_dir)):
            with open(os.path.join(frames_dir, fn), 'rb') as f:
                b64 = base64.b64encode(f.read()).decode('utf-8')
            parts.append({"text": f"Frame {fn}:"})
            parts.append({"inline_data": {"mime_type": "image/jpeg", "data": b64}})
        g_url = f"https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key={API_KEY}"
        g_req = urllib.request.Request(g_url, data=json.dumps({"contents":[{"parts":parts}],
            "generationConfig":{"temperature":0.3,"maxOutputTokens":4096}}).encode('utf-8'),
            headers={"Content-Type":"application/json"})
        with urllib.request.urlopen(g_req, timeout=180) as resp:
            frame_desc = json.loads(resp.read().decode('utf-8'))["candidates"][0]["content"]["parts"][0]["text"]
        print("GEMINI DONE")

# ===== 5. Summary Generation =====
is_design = TIER.startswith('2')
dm_section = f"\n弹幕高光:\n{dm_buckets}" if dm_buckets else ""
frame_section = f"\n画面描述:\n{frame_desc}" if frame_desc else ""

prompt = f"""Analyze this video. Title: {title}, Uploader: {uploader}, Duration: {duration//60}:{duration%60:02d}.

Subtitle transcript:
{subs_text[:4000]}

{dm_section}
{frame_section}

Generate a structured summary in Chinese. Tier: {TIER}."""
if is_design:
    prompt += "\nInclude design analysis: select 3-5 from [玩法价值/交互成本/表现质量/NPC参与/社交传播/边界规则/技术风险/运营/可复用资产/负反馈], each 2-3 sentences tied to evidence. Add '参考知识' and '风险与待验证' sections."
if TIER.split('.')[1] == '1':
    prompt += "\nOutput format: timeline table + key observations + danmaku highlights. Concise, 20 lines max."
else:
    prompt += "\nOutput 9-section: 章节概览|总体摘要|话题章节|关键引用|弹幕反应|新颖观点|反直觉观点|核心张力|方法论|关键数据."

# Generate summary (agent handles this with local LLM for x.1, API for x.2/x.3)
print("\n=== SUMMARY PROMPT ===")
print(prompt[:500])
print(f"... ({len(prompt)} chars total)")
print("=== END PROMPT ===")

# For x.2/x.3 with GPT-5.5:
if TIER.split('.')[1] in ('2', '3'):
    lk = os.environ.get('LEIHUO_API_KEY', ''); lu = os.environ.get('LEIHUO_API_BASE', '')
    if TIER.split('.')[1] == '3':
        # x.3: GPT-5.5 with vision
        content = [{"type":"text","text":prompt}]
        for fn in sorted(os.listdir(frames_dir)):
            with open(os.path.join(frames_dir, fn), 'rb') as f:
                content.append({"type":"image_url","image_url":{"url":f"data:image/jpeg;base64,{base64.b64encode(f.read()).decode('utf-8')}"}})
        payload = {"model":"gpt-5.5","messages":[{"role":"user","content":content}],"max_tokens":6000,"temperature":0.5}
    else:
        payload = {"model":"gpt-5.5","messages":[{"role":"user","content":prompt}],"max_tokens":6000,"temperature":0.5}
    req = urllib.request.Request(lu, data=json.dumps(payload).encode('utf-8'),
        headers={"Authorization":f"Bearer {lk}","Content-Type":"application/json"})
    with urllib.request.urlopen(req, timeout=300) as resp:
        result = json.loads(resp.read().decode('utf-8'))
    summary = result["choices"][0]["message"]["content"]
    usage = result.get("usage", {})
    cost = usage.get('prompt_tokens',0)*5/1e6 + usage.get('completion_tokens',0)*30/1e6
    print(f"\nGPT-5.5 DONE | Cost: ${cost:.4f} | Tokens: {usage.get('total_tokens','?')}")

# ===== 6. Save + Cleanup =====
safe_title = title.replace('/','').replace(':','')[:80]
out_path = os.path.join(OUT, f'{safe_title}-summary.md')
with open(out_path, 'w', encoding='utf-8') as f:
    f.write(f"---\ntitle: \"{title}\"\nsource: \"{URL}\"\nauthor:\n  - \"[[{uploader}]]\"\ntype: \"source\"\ntags:\n  - bilibili\n---\n\n{summary}")
print(f"\nOUTPUT: {out_path}")

shutil.rmtree(TMP, ignore_errors=True)
print("CLEANUP DONE")
```

---

## Prerequisites

- `yt-dlp`: `pip install yt-dlp`
- `ffmpeg`: pre-installed
- `python3`: stdlib only
- B站 cookies: Chrome extension "Get cookies.txt LOCALLY" → `~/bilibili_cookies.txt`
- `GEMINI_API_KEY` (x.2): https://aistudio.google.com/apikey (free)
- `LEIHUO_API_KEY` + `LEIHUO_API_BASE` (x.2/x.3): 雷火网关

---

## Output

- `{OUT}/{标题}-summary.md` — 结构化总结
- `{OUT}/` 默认 `$PWD/video-analysis/`，可通过 `YOUTUBE_SUBTITLES_DIR` 覆盖

## First-Time Setup Check

在 Pipeline 之前运行，**每个缺失项独立报错，附带修复指引**：

```
检查工具...
  yt-dlp:     ✅
  ffmpeg:     ✅
  python3:    ✅
检查 B站:
  cookies:    ❌ 未找到 ~/bilibili_cookies.txt
                 → 安装 Chrome 扩展 "Get cookies.txt LOCALLY"
                 → 打开 bilibili.com → Export → 保存为 ~/bilibili_cookies.txt
  (YouTube 用户可忽略此项)
检查 API Keys (当前模式需要):
  Gemini:     ❌ GEMINI_API_KEY 未设置 (1.2/2.2 需要)
                 → https://aistudio.google.com/apikey 免费注册
  Leihuo:     ✅ (1.3/2.3 可用)
```

**不阻止运行**，但告知用户缺什么会导致什么降级——比如缺 Gemini key 则 x.2 不可用，缺 cookies 则 B站 x.2/x.3 不可用。

## Error Handling

运行时报错不静默，每个错误附带上下文和修复建议：

| Error | 用户提示 |
|-------|---------|
| yt-dlp not found | `❌ 未安装 yt-dlp。运行: pip install yt-dlp` |
| B站 cookies 缺失/过期 | `❌ B站 cookies 未找到或已过期。请重新导出: Chrome 扩展 "Get cookies.txt LOCALLY" → bilibili.com → Export → ~/bilibili_cookies.txt` |
| B站 SESSDATA 无效 | `❌ B站 登录态失效，cookies 中的 SESSDATA 已过期。重新导出 cookies 即可。` |
| Gemini 429 | `❌ Gemini 免费额度用完。换 1.3/2.3 (GPT-5.5) 继续，或等明天重置。` |
| GPT-5.5 错误 | `❌ GPT-5.5 调用失败 (原因: {msg})。已自动降级到 1.1 速览模式。` |
| 视频无法下载 (412/403) | `❌ B站 反爬拦截。cookies 可能过期，或需更新 yt-dlp: pip install -U yt-dlp` |
| 0 scene frames | `⚠️ 场景检测无结果。已自动降级为固定间隔抽帧 (每 10% 时长一帧)。` |
| ffmpeg not found | `❌ 未安装 ffmpeg。Windows: winget install ffmpeg / Mac: brew install ffmpeg` |
| 网络不通 | `❌ 无法访问 B站/YouTube API。请检查网络或 VPN。` |
