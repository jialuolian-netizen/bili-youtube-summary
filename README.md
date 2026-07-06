# bili-youtube-summary · 视频深度分析 Skill

> 丢给它一条 YouTube 或 Bilibili 链接，自动下载字幕/弹幕/关键帧，**三级输出**满足不同场景。任何支持 Skills 的 AI agent 都能用。

---

## 三级输出

| 层级 | 触发词 | 输出 | 成本 | 适合 |
|------|--------|------|------|------|
| **L1 速览** | `总结` `看看` `讲了什么` | 标题 + 时间轴 + 关键观察 + 弹幕高光 + 证据档位 | 免费 | 快速扫片 |
| **L2 深度** | `深度` `详细` `--deep` | L1 + 9 段结构化总结（章节概览、话题、引用、观点、张力、方法论…） | ~$0.12 | 单视频深入 |
| **L2.1 设计** | `设计` `灵感` `拆解` `竞品` `启发` `落地` | L2 + 10 维设计注意点 + 参考知识 + 风险清单 | ~$0.18 | 策划竞品分析 |

每份输出标注**证据档位**（A 完整视频 / B 低清 / C 仅元数据 / D 不可访问），让读者知道结论可靠度。

---

## 安装

```bash
npx skills add jialuolian-netizen/bili-youtube-summary
```

---

## 使用

```
总结这个视频 https://www.bilibili.com/video/BVxxxx
深度分析 https://www.youtube.com/watch?v=xxxx
从策划视角拆解这个竞品视频 https://www.bilibili.com/video/BVxxxx
```

---

## 前置依赖

| 依赖 | 安装 | 备注 |
|------|------|------|
| `yt-dlp` | `pip install yt-dlp` | 字幕/视频下载 |
| `ffmpeg` | 通常已预装 | 场景检测 + 抽帧 |
| `python3` | 标准库即可 | 清洗/调度 |
| B站 cookies | Chrome 扩展 "Get cookies.txt LOCALLY" → 导出到 `~/bilibili_cookies.txt` | ~30 天过期 |

---

## API Key 配置

| Key | 获取 | 费用 | 用途 |
|-----|------|------|------|
| `GEMINI_API_KEY` | https://aistudio.google.com/apikey（免费注册） | 免费 1500次/天 | 帧描述 |
| `LEIHUO_API_KEY` | 雷火大模型网关控制台 | ~$0.18/次 | L2/L2.1 GPT-5.5 |

```bash
# Windows
setx GEMINI_API_KEY "你的key"
setx LEIHUO_API_KEY "你的key"
setx LEIHUO_API_BASE "https://ai.leihuo.netease.com/v1/chat/completions"
```

无 Key → 仅 L1 文本模式可用（有字幕的视频）。

---

## 输出

- `{标题}.md` — 带时间戳转录 + 弹幕密度表（B站）
- `{标题}-summary.md` — 结构化总结（L1/L2/L2.1）

---

## License

[MIT](LICENSE)

## Credits

Forked from [xiapuyang/youtube-summary](https://github.com/xiapuyang/youtube-summary). Extended with Windows support, 3-tier output, evidence grading, visual mode (ffmpeg + Gemini + GPT-5.5), danmaku analysis, and 10-dim design checklist.
