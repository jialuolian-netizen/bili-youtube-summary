# bili-youtube-summary · 视频深度分析 Skill

> 丢给它一条 YouTube 或 Bilibili 链接，自动下载字幕/弹幕/关键帧，**6 档输出**满足不同场景。任何支持 Skills 的 AI agent 都能用。

---

## 6 档输出

| 档位 | 触发词 | 帧描述 | 总结引擎 | 成本 | 适合 |
|------|--------|--------|---------|------|------|
| **1.1 速览** | `总结` `讲了什么` | 无 | Gemini | 免费 | 快速扫片 |
| **1.2 中度** | `深度` `详细` | Gemini | Gemini | 免费 | 单视频深入 |
| **1.3 深度** | `深度` `--deep` | GPT-5.5 | GPT-5.5 | ~$0.18 | 高精度分析 |
| **2.1 策划速览** | `设计` `拆解` | 无 | Gemini | 免费 | 策划快速评估 |
| **2.2 策划中度** | `灵感` `启发` | Gemini | Gemini | 免费 | 策划竞品分析 |
| **2.3 策划深度** | `落地` `竞品` | GPT-5.5 | GPT-5.5 | ~$0.18 | 策划深度拆解 |

- x.1 档：纯文本，秒出，不下载视频
- x.2 档：Gemini 全链路（帧描述 + 总结），免费（1500 次/天）
- x.3 档：GPT-5.5 全链路（帧描述 + 总结），付费

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
| YouTube cookies | 同上 → youtube.com → Export → `~/youtube_cookies.txt` | 防 412 限流（可选） |

---

## API Key 配置

| Key | 获取 | 费用 | 用途 |
|-----|------|------|------|
| `GEMINI_API_KEY` | https://aistudio.google.com/apikey | 免费 1500次/天 | x.2 帧描述 + 总结 |
| `LEIHUO_API_KEY` | 雷火大模型网关控制台 | ~$0.18/次 | x.3 帧描述 + 总结 |

```bash
# Windows
setx GEMINI_API_KEY "你的key"
setx LEIHUO_API_KEY "你的key"
setx LEIHUO_API_BASE "https://ai.leihuo.netease.com/v1/chat/completions"
```

无 Key → 仅 x.1 文本模式可用。

---

## 特性

- **B站 playurl API 备选下载**：yt-dlp 失败时自动切换 CDN 直链（urllib），多画质降级尝试
- **字幕时间范围校验**：检测 player/v2 字幕错配（时间戳远超视频时长自动重试）
- **真 aid 字幕路由**：使用 view API 返回的真实 aid 替代 aid=0，降低字幕错配率
- **Cookie 健康检查**：运行前检查 cookies 年龄，>25 天预警
- **场景检测 fallback**：ffmpeg 场景检测返回 0 帧时自动降级为固定间隔抽帧

---

## 输出

- `{标题}-summary.md` — 结构化总结

---

## License

[MIT](LICENSE)

## Credits

Forked from [xiapuyang/youtube-summary](https://github.com/xiapuyang/youtube-summary). Extended with 6-tier output, Gemini free tier, B站 playurl API download, subtitle validation, evidence grading, danmaku analysis, and 10-dim design checklist.
