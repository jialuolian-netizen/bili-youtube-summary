# CLAUDE.md

## Agent Permission

This skill is maintained by **LianJialuo (jialuolian-netizen)** . The agent has **full read-write-auto-commit permission** — no confirmation needed for edits, commits, or pushes.

## Commands

```bash
# 安装 pre-commit hooks（首次，可选）
pre-commit install

# 提交前自查
pre-commit run --all-files

# 手动冒烟测试（不经 Claude，直接跑底层下载）
yt-dlp --list-subs "<URL>"                       # 看有哪些字幕轨
yt-dlp --write-auto-subs --sub-langs zh-Hans \
  --skip-download -o "/tmp/test_%(id)s" "<URL>"  # 拉一条字幕验证
```

> 本 skill 是 **prompt 驱动**，没有可执行入口脚本——真正的流程编排在 `SKILL.md` 的 10 个 Step 里。要验证完整流程，把链接发给 Claude Code 触发 skill 即可。

## Architecture

整个 skill 就两个文件，逻辑全部内联：

```
SKILL.md                      # ← 真理来源：完整的 Python pipeline（内联 heredoc）
                              #   1. 元数据 · 2. 字幕（含时间校验+重试） · 3. 弹幕
                              #   4. 视频下载（yt-dlp → playurl API fallback）
                              #   5. 场景检测（ffmpeg → 固定间隔 fallback）
                              #   6. 帧描述（x.2→Gemini / x.3→GPT vision）
                              #   7. 总结（x.2→Gemini / x.3→GPT-5.5）
                              #   8. 保存 + 清理
references/
  cookies-setup.md            # B站 + YouTube cookies 配置指南
  output-formats.md           # 9 段总结的标题/模板 + frontmatter 模板
```

## Key design constraints

读这些**反直觉**约定，否则容易改坏：

- **没有 .py 文件可改**：所有处理逻辑都是 `SKILL.md` 里的内联 python heredoc。改行为 = 改 SKILL.md，别去找模块。
- **Bilibili 必须带 cookies，YouTube 推荐带**：B站 `--cookies bilibili_cookies.txt`；YouTube `--cookies youtube_cookies.txt`（可选，但无 cookie 高概率遇到 412 限流）。
- **B站字幕三重保障**：真 aid → aid=0 → yt-dlp，每层有时间范围校验（字幕时间戳超过视频时长 2x 则丢弃重试）。
- **B站视频双重下载**：yt-dlp 失败 → playurl API 直取 CDN（urllib，多画质降级 qn=32→64→16→80，flv 自动转 mp4）。
- **字幕语言优先级是写死的回退链**：Bilibili `ai-zh → zh-Hans → zh-CN → zh → en`；YouTube `zh-Hans → zh-CN → zh → en`。命中第一条即停。
- **场景检测有 fallback**：ffmpeg scene detection 返回 0 帧 → 自动降级为每 10% 时长固定抽帧。
- **x.2 全 Gemini（免费），x.3 全 GPT-5.5（付费）**：中度档帧描述+总结都用 Gemini；深度档都用 GPT-5.5。
- **文件名只剥 `/` 和 ASCII `:`**：保留全角标点（`：《》、`），截断 100 字符。
- **产物 frontmatter 是 Obsidian 风格**（`[[wikilink]]`、`type: source`）。这是设计选择不是遗留。
- **总结正文禁止归因前缀**：不要写「X 说 / X 认为 / 据 X」，说观点本身。

## Development Workflow

- 改动走 feature 分支 + PR（`.pre-commit-config.yaml` 里 `no-commit-to-branch` 守 main）。
- bootstrap / 首次 commit 场景可 `--no-verify`，但需审批。
- 改了流程后，用一条真实 YouTube + 一条真实 Bilibili 链接各跑一遍，确认两份产物正常落盘。

---

# Karpathy Guidelines

Behavioral guidelines to reduce common LLM coding mistakes, derived from [Andrej Karpathy's observations](https://x.com/karpathy/status/2015883857489522876) on LLM coding pitfalls.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- If you write 200 lines and it could be 50, rewrite it.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

- Don't "improve" adjacent code, comments, or formatting.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.
