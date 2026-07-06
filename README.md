# bili-youtube-summary · 视频深度分析 Skill

> 丢给它一条 YouTube 或 Bilibili 链接，自动下载字幕/弹幕/关键帧，生成 9-10 段结构化深度总结。任何支持 Skills 的 AI agent 都能用。

把链接粘进对话即可触发，无需记命令。

---

## 它做什么

| 能力 | 说明 |
|------|------|
| **三模式** | 文本模式（字幕 → 9 段总结）、视觉 Quick（场景检测 → Gemini 免费分析）、视觉 Deep（GPT-5.5 一步到位，~$0.18/次） |
| **弹幕分析** | B站专属：抓取弹幕 → 10 秒密度段聚合 → Top 8 高能时刻 → 观众情绪聚类 |
| **场景检测** | ffmpeg 内容感知切分镜头 → 每场景抽 1 帧 → 多模态模型画面理解 |
| **双平台** | YouTube（自动/人工字幕）+ Bilibili（AI 字幕 + 弹幕，需 cookies） |
| **10 段总结** | 章节概览、总体摘要、话题章节、关键引用、弹幕反应、新颖观点、反直觉观点、核心张力、方法论、关键数据 |
| **Obsidian 兼容** | frontmatter 含 wikilink、ctime/mtime、type: source，直接落 vault |

---

## 安装

```bash
npx skills add jialuolian-netizen/bili-youtube-summary
```

装好后重启 agent 即可。

---

## 使用

```
总结这个视频 https://www.bilibili.com/video/BVxxxx
```

```
--deep 帮我深度分析 https://www.youtube.com/watch?v=xxxx
```

触发词：任何含 `youtube.com / youtu.be / bilibili.com / b23.tv` 的链接，或「总结这个视频」「获取字幕」「视频总结」「提取字幕」。

---

## 前置依赖

| 依赖 | 安装 | 备注 |
|------|------|------|
| `yt-dlp` | `pip install yt-dlp` | 字幕/视频下载 |
| `ffmpeg` | 通常已预装 | 场景检测 + 抽帧 |
| `python3` | 标准库即可 | 清洗/调度 |
| B站 cookies | Chrome 扩展 "Get cookies.txt LOCALLY" → 导出到 `~/bilibili_cookies.txt` | 约 30 天过期，见 `references/cookies-setup.md` |

---

## API Key 配置

> 视觉模式需要，文本模式无需。

| Key | 获取方式 | 费用 | 用途 |
|-----|---------|------|------|
| `GEMINI_API_KEY` | https://aistudio.google.com/apikey（免费注册） | 免费 1500次/天 | `--quick` 模式 |
| `LEIHUO_API_KEY` | 雷火大模型网关控制台 | ~$0.18/次 | `--deep` 模式 |

```bash
# Windows
setx GEMINI_API_KEY "你的key"
setx LEIHUO_API_KEY "你的key"
setx LEIHUO_API_BASE "https://ai.leihuo.netease.com/v1/chat/completions"

# macOS/Linux
export GEMINI_API_KEY="你的key"
export LEIHUO_API_KEY="你的key"
export LEIHUO_API_BASE="https://ai.leihuo.netease.com/v1/chat/completions"
```

两个 key 都没有 → 仅文本模式可用。无字幕视频会报错提示缺少 key。

---

## 模式选择

```
有字幕? ──Yes──→ Text (免费，9 段总结)
    │
    No
    │
    ├── "quick/快速" → Visual Quick (Gemini 免费, ~$0.12)
    │
    └── "deep/深度" → Visual Deep (GPT-5.5, ~$0.18)
```

---

## 输出

- `{标题}.md` — 带时间戳完整转录 + 弹幕密度表（B站）
- `{标题}-summary.md` — 9/10 段结构化深度总结

Obsidian 风格 frontmatter，支持 wikilink、标签。

---

## 环境变量

| 变量 | 作用 | 默认 |
|------|------|------|
| `YOUTUBE_SUBTITLES_DIR` | 产物输出目录 | `.` |
| `GEMINI_API_KEY` | Gemini 视觉分析 | — |
| `LEIHUO_API_KEY` | GPT-5.5 深度分析 | — |
| `LEIHUO_API_BASE` | 雷火网关地址 | `https://ai.leihuo.netease.com/v1/chat/completions` |
| `BILIBILI_COOKIES_FILE` | B站 cookies 路径 | `~/bilibili_cookies.txt` |

---

## License

[MIT](LICENSE)

## Credits

Forked from [xiapuyang/youtube-summary](https://github.com/xiapuyang/youtube-summary). Extended with Windows support, visual mode, danmaku analysis, and multi-model integration.
