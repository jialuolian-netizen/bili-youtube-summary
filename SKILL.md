---
name: youtube-summary
description: >
  Extract subtitles from YouTube or Bilibili, save raw transcript, and generate a structured summary.
  Triggers: ANY URL containing youtube.com, youtu.be, bilibili.com, or b23.tv — use this skill
  immediately, do NOT use fetch-content. Also triggers on: "summarize this video / get subtitles" /
  "总结这个视频" / "帮我看看这个视频" / "获取字幕" / "视频总结" / "提取字幕".
---

# Video Subtitle Extractor (YouTube + Bilibili)

Detect platform → download subtitles → clean → save raw → generate summary.

Supports **macOS/Linux (bash)** and **Windows (PowerShell 5.1+)** . Python scripts are cross-platform.

---

## Step 0 — B站 Cookie Setup (One-Time)

Bilibili requires login cookies. Windows Chrome cookie decryption (DPAPI) is broken in yt-dlp.

**One-time setup** — do this once, then cookies work for ~30 days:

1. Install Chrome extension **"Get cookies.txt LOCALLY"** from Chrome Web Store
2. Open `bilibili.com`, log in, click extension → **Export**
3. Save as `%USERPROFILE%\bilibili_cookies.txt` (Windows) or `~/bilibili_cookies.txt` (macOS/Linux)

**Check if cookies exist and are fresh:**
- Windows: `Test-Path "$env:USERPROFILE\bilibili_cookies.txt"`
- Unix: `test -f ~/bilibili_cookies.txt`

If cookies are missing or >30 days old, tell user to re-export from the extension.

The cookies path is `${BILIBILI_COOKIES_FILE:-$HOME/bilibili_cookies.txt}` on Unix, `$env:USERPROFILE\bilibili_cookies.txt` on Windows.

---

## Step 1 — Ensure yt-dlp is available

**Unix (bash):**
```bash
if ! command -v yt-dlp &>/dev/null; then
  echo "yt-dlp not found, installing..."
  pip install -q yt-dlp || pip3 install -q yt-dlp
fi
yt-dlp -U --quiet 2>/dev/null || true
```

**Windows (PowerShell):**
```powershell
if (-not (Get-Command yt-dlp -ErrorAction SilentlyContinue)) {
    pip install -q yt-dlp
}
```

If installation fails, stop and tell the user to install yt-dlp manually (`pip install yt-dlp` or `brew install yt-dlp`).

---

## Step 2 — Detect platform and download subtitles

Detect whether the URL is Bilibili or YouTube, then use the appropriate strategy.

**Unix (bash):**
```bash
URL="<user-provided URL>"
TMPDIR=$(mktemp -d)
SUB_FILE=""
SUBTITLE_LANG=""

if echo "$URL" | grep -qE '(bilibili\.com|b23\.tv)'; then
  PLATFORM="bilibili"
  SITE_NAME="Bilibili"
  SITE_DOMAIN="bilibili.com"
else
  PLATFORM="youtube"
  SITE_NAME="YouTube"
  SITE_DOMAIN="youtube.com"
fi
```

**Windows (PowerShell):**
```powershell
$URL = "<user-provided URL>"
$TMPDIR = Join-Path $env:TEMP "yt_summary_$(Get-Random)"
New-Item -ItemType Directory -Path $TMPDIR -Force | Out-Null

if ($URL -match 'bilibili\.com|b23\.tv') {
    $PLATFORM = "bilibili"
    $SITE_NAME = "Bilibili"
    $SITE_DOMAIN = "bilibili.com"
} else {
    $PLATFORM = "youtube"
    $SITE_NAME = "YouTube"
    $SITE_DOMAIN = "youtube.com"
}
```

### Bilibili branch — Unix

```bash
if [ "$PLATFORM" = "bilibili" ]; then
  BILI_COOKIES="${BILIBILI_COOKIES_FILE:-$HOME/bilibili_cookies.txt}"
  COOKIE_ARGS="--cookies $BILI_COOKIES"

  # List available subtitle langs
  LIST_OUTPUT=$(yt-dlp --no-update --list-subs $COOKIE_ARGS "$URL" 2>&1)
  if echo "$LIST_OUTPUT" | grep -qi "login\|not logged\|需要登录\|please log"; then
    echo "❌ Bilibili cookies expired. Re-export from Get cookies.txt extension."
    rm -rf "$TMPDIR"; exit 1
  fi
  AVAIL_LANGS=$(echo "$LIST_OUTPUT" | awk '/^[a-z]/{print $1}' | grep -v "^Language$")

  for lang in ai-zh zh-Hans zh-CN zh en; do
    if echo "$AVAIL_LANGS" | grep -q "^${lang}$"; then
      yt-dlp --no-update --write-subs --sub-langs "$lang" --skip-download --retries 3 \
        -o "$TMPDIR/bili_%(id)s" $COOKIE_ARGS "$URL" 2>/dev/null
      SUB_FILE=$(find "$TMPDIR" -maxdepth 1 -name "*.${lang}.*" 2>/dev/null | head -1)
      if [ -n "$SUB_FILE" ]; then SUBTITLE_LANG="$lang"; break; fi
    fi
  done
fi
```

### Bilibili branch — Windows

```powershell
if ($PLATFORM -eq "bilibili") {
    $BILI_COOKIES = "$env:USERPROFILE\bilibili_cookies.txt"
    if (-not (Test-Path $BILI_COOKIES)) {
        Write-Host "❌ No Bilibili cookies found. Please run Step 0 setup first."; exit 1
    }
    $COOKIE_ARGS = "--cookies `"$BILI_COOKIES`""

    # List available subtitles
    $listOutput = yt-dlp --no-update --list-subs --cookies $BILI_COOKIES $URL 2>&1
    if ($listOutput -match 'login|not logged|需要登录') {
        Write-Host "❌ Cookies expired. Re-export from Get cookies.txt extension."; exit 1
    }
    # Extract available language codes from --list-subs output
    $availLines = ($listOutput -split "`n" | Where-Object { $_ -match '^\s*(ai-zh|zh-|en|danmaku)\s' })
    $availLangs = ($availLines | ForEach-Object { ($_ -split '\s+')[0] }) -join ' '

    foreach ($lang in @('ai-zh', 'zh-Hans', 'zh-CN', 'zh', 'en')) {
        if ($availLangs -match $lang) {
            yt-dlp --no-update --write-subs --sub-langs $lang --skip-download --retries 3 `
                -o "$TMPDIR\bili_%(id)s" $COOKIE_ARGS.Split(' ') $URL 2>&1 | Out-Null
            $SUB_FILE = Get-ChildItem $TMPDIR -Filter "*.$lang.*" | Select-Object -First 1 -ExpandProperty FullName
            if ($SUB_FILE) { $SUBTITLE_LANG = $lang; break }
        }
    }
}
```

### YouTube branch — Unix

```bash
if [ "$PLATFORM" = "youtube" ]; then
  for lang in zh-Hans zh-CN zh en; do
    yt-dlp --no-update --write-subs --write-auto-subs --sub-langs "$lang" --skip-download \
      --sub-format vtt --retries 3 --sleep-requests 1 -o "$TMPDIR/yt_%(id)s" "$URL" 2>/dev/null
    SUB_FILE=$(find "$TMPDIR" -maxdepth 1 -name "*.${lang}.vtt" 2>/dev/null | head -1)
    if [ -n "$SUB_FILE" ]; then SUBTITLE_LANG="$lang"; break; fi
    sleep 1
  done
fi
```

### YouTube branch — Windows

```powershell
if ($PLATFORM -eq "youtube") {
    foreach ($lang in @('zh-Hans', 'zh-CN', 'zh', 'en')) {
        yt-dlp --no-update --write-subs --write-auto-subs --sub-langs $lang --skip-download `
            --sub-format vtt --retries 3 --sleep-requests 1 -o "$TMPDIR\yt_%(id)s" $URL 2>&1 | Out-Null
        $SUB_FILE = Get-ChildItem $TMPDIR -Filter "*.$lang.vtt" -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName
        if ($SUB_FILE) { $SUBTITLE_LANG = $lang; break }
        Start-Sleep -Seconds 1
    }
}
```

### Fail if no subtitles

**Unix:** `if [ -z "$SUB_FILE" ]; then echo "No subtitles found."; rm -rf "$TMPDIR"; exit 1; fi`

**Windows:**
```powershell
if (-not $SUB_FILE) { Write-Host "No subtitles found."; Remove-Item -Recurse -Force $TMPDIR -ErrorAction SilentlyContinue; exit 1 }
```

---

## Step 3 — Clean subtitle file → plain text + timestamped text

Cross-platform Python script. Write to `cleaned.txt` (no timestamps, for LLM) and `timestamped.txt` (`[mm:ss]` format):

```python
import sys, html, re, os

sub_file, tmpdir = sys.argv[1], sys.argv[2]

def parse_time(t):
    t = t.strip().replace(',', '.')
    parts = t.split(':')
    if len(parts) == 3:
        return int(parts[0]) * 3600 + int(parts[1]) * 60 + float(parts[2])
    return int(parts[0]) * 60 + float(parts[1])

def fmt(secs):
    secs = int(secs)
    return f"[{secs // 60:02d}:{secs % 60:02d}]"

with open(sub_file, encoding='utf-8') as f:
    lines = f.read().splitlines()

cleaned_out = []
ts_out = []
seen = set()
current_secs = 0
ext = os.path.splitext(sub_file)[1].lower()

i = 0
while i < len(lines):
    line = lines[i].strip()
    if ext == '.srt':
        if not line or re.match(r'^\d+$', line):
            i += 1; continue
        m = re.match(r'^(\d{2}:\d{2}:\d{2}[,\.]\d+) -->', line)
        if m:
            current_secs = parse_time(m.group(1))
            i += 1; continue
    else:  # vtt
        if not line or line.startswith('WEBVTT') or line.startswith('NOTE') \
                or line.startswith('Kind:') or line.startswith('Language:'):
            i += 1; continue
        m = re.match(r'^([\d:\.]+) -->', line)
        if m:
            current_secs = parse_time(m.group(1))
            i += 1; continue
    text = html.unescape(re.sub(r'<[^>]+>', '', line)).strip()
    if text and text not in seen:
        seen.add(text)
        cleaned_out.append(text)
        ts_out.append(f"{fmt(current_secs)} {text}")
    i += 1

with open(os.path.join(tmpdir, 'cleaned.txt'), 'w', encoding='utf-8') as f:
    f.write('\n\n'.join(cleaned_out) + '\n')
with open(os.path.join(tmpdir, 'timestamped.txt'), 'w', encoding='utf-8') as f:
    f.write('  \n'.join(ts_out) + '\n')

print(f"cleaned: {len(cleaned_out)} lines, {sum(len(l) for l in cleaned_out)} chars")
```

**Windows invocation:**
```powershell
python -c "<paste the script above, with $SUB_FILE and $TMPDIR substituted>" 
# OR save as temp script and run:
# python ./clean_subs.py $SUB_FILE $TMPDIR
```

---

## Step 4 — Resolve output directory and filename

**Unix:**
```bash
OUTPUT_DIR="${YOUTUBE_SUBTITLES_DIR:-.}"
mkdir -p "$OUTPUT_DIR"
SLUG=$(echo "<title>" | python3 -c "import sys; t=sys.stdin.read().strip(); print(t.replace('/','').replace(':','')[:100])")
```

**Windows:**
```powershell
$OUTPUT_DIR = if ($env:YOUTUBE_SUBTITLES_DIR) { $env:YOUTUBE_SUBTITLES_DIR } else { "." }
New-Item -ItemType Directory -Path $OUTPUT_DIR -Force | Out-Null
$SLUG = ($TITLE -replace '[/:]', '') -replace '[\\*?"<>|]', ''; if ($SLUG.Length -gt 100) { $SLUG = $SLUG.Substring(0,100) }
```

---

## Step 5 — Fetch video metadata

**Windows (PowerShell) — reads metadata via yt-dlp --dump-json:**
```powershell
$metaJson = yt-dlp --no-update --dump-json --no-playlist $COOKIE_ARGS.Split(' ') $URL 2>$null | ConvertFrom-Json
$TITLE = $metaJson.title
$CHANNEL = $metaJson.uploader
$DURATION = $metaJson.duration_string
$DATE = $metaJson.upload_date
$DESC = if ($metaJson.description) { ($metaJson.description -split '\n\n')[0] -replace '\n',' ' } else { '' }
if ($DESC.Length -gt 300) { $DESC = $DESC.Substring(0,300) }
$CHAPTERS = if ($metaJson.chapters) {
    ($metaJson.chapters | ForEach-Object {
        $m = [int]($_.start_time / 60); $s = [int]($_.start_time % 60)
        "  - `"${m}:${s.ToString('00')} $($_.title)`""
    }) -join "`n"
} else { '' }
```

For Unix, use the bash heredoc version from the original skill (see git history).

---

## Step 6 — Save raw transcript

Write `$OUTPUT_DIR/$SLUG.md` with Obsidian-style frontmatter + timestamped transcript.

**Frontmatter template:**
```yaml
---
title: "<title>"
source: "<URL>"
author:
  - "[[<channel>]]"
published: "<YYYYMMDD>"
description: "<DESCRIPTION>"
tags:
  - "<PLATFORM>"
ctime: "<NOW>"
mtime: "<NOW>"
words: "<WORDS>"
site: "<SITE_NAME>"
domain: "<SITE_DOMAIN>"
channel: "<channel>"
duration: "<duration>"
subtitle_lang: "<SUBTITLE_LANG>"
chapters:
<CHAPTERS or empty>
type: "source"
---
```

Append full `timestamped.txt` content as the body. Write with Python to ensure UTF-8 encoding:

**Windows:**
```powershell
$NOW = Get-Date -Format "yyyy-MM-ddTHH:mm"
$WORDS = (Get-Content "$TMPDIR\timestamped.txt" | Measure-Object -Word).Words
# Use Python to write with proper UTF-8
python -c "
import sys, io; sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
title = '$($TITLE -replace "'","''")'
# ... construct frontmatter + append timestamped.txt content
"
```

---

## Step 7 — Validate transcript and check length

If `< 100` chars: stop — subtitle was empty.
If `≤ 120000`: use full cleaned.txt as summary input.
If `> 120000`: run Step 8 map-reduce first.

---

## Step 8 — Map-reduce for long transcripts (> 120k only)

Split into overlapping chunks (~40k chars, 2k overlap). Extract structured notes per chunk.
(Cross-platform Python — original SKILL.md script unchanged.)

---

## Step 9 — Generate summary (LLM-driven)

Read `references/output-formats.md` for exact templates. Generate 9-section summary:

1. **Chapter Overview / 章节概览** — timestamped table
2. **Overall Summary / 总体摘要** — Chain of Density (3 iterations)
3. **Topic Chapters / 话题章节** — 4-8 major topics
4. **Key Quotes / 关键引用** — 10-15 verbatim blockquotes
5. **Novel Ideas / 新颖观点** — fresh, non-mainstream ideas
6. **Counter-intuitive Views / 反直觉观点** — common belief → actual claim format
7. **Core Tensions / 核心张力** — opposing forces
8. **Methodology / 方法论** — frameworks, heuristics
9. **Key Data / 关键数据** — specific numbers only

Voice: No attribution prefixes. Every bullet 2-3 sentences minimum.
Omit sections with no relevant content.

Write summary to `$OUTPUT_DIR/$SLUG-summary.md`.

---

## Step 10 — Clean up

**Windows:**
```powershell
Remove-Item -Recurse -Force $TMPDIR -ErrorAction SilentlyContinue
```

Report output paths: `$SLUG.md` and `$SLUG-summary.md`.

---

## Error Handling

| Error | Action |
|-------|--------|
| `yt-dlp` not found | Tell user: `pip install yt-dlp` |
| B站 cookies missing | Tell user: run Step 0 — install "Get cookies.txt LOCALLY" Chrome extension, export from bilibili.com → `~/bilibili_cookies.txt` |
| B站 cookies expired (login error) | Tell user: re-export cookies from the extension (Step 0) |
| No subtitles found | Stop. Suggest checking video page. |
| `cleaned.txt` < 100 chars | Stop. Subtitle empty or disabled. |
| `--dump-json` fails | Continue without metadata. Use empty strings. |
| Transcript > 120k + map-reduce empty | Stop. Show first 200 chars for diagnosis. |
