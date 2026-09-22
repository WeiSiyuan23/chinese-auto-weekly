# chinese-auto-weekly 🚗📰

**A production-hardened Agent Skill that turns weekly Chinese automotive industry news into a publication-ready 图文周报 (photo-rich weekly report) — with vehicle-accurate images, vision-verified captions, and a one-command HTML+PNG render pipeline.**

[中文说明](#中文说明) · [Skill](skill/SKILL.md) · [Example output](examples/report.sample.html)

---

## What it does

Given a brand watchlist (default: BYD 仰望 YangWang competitor set), the agent produces every week:

```
┌─────────────────────────────────────────────┐
│  汽车行业周报  2026.08.24 — 2026.08.30        │
│  ── 01 竞品动态 ── 02 新车上市 ── 03 政策补贴 ──│
│  ── 04 关税贸易 ── 05 企业财报 ── 06 行业事件 ──│
│  ✔ 每条 3-5 个要点 + 具体数字 + 来源链接        │
│  ✔ 每张配图经视觉模型验证确系所述车型           │
│  ✔ 1080px 竖版 PNG 长图，手机直接转发          │
└─────────────────────────────────────────────┘
```

- **6 fixed sections**, color-accented: competitor moves · new launches (product cards) · policy & subsidies · tariffs & trade · earnings · industry events
- **Three-principles image sourcing**: per-model search, accuracy-over-presence, global md5 dedup — no more "Great Wall story carrying a competitor's car" or red-slogan banners
- **Two-stage image QA**: PIL rule filter (banner/logo/tofu detection) → vision-model semantic audit with confidence thresholds (≥0.90 keep / 0.80-0.89 backup / <0.80 drop)
- **Rendering that survives reality**: HTML template + headless Chromium long screenshot, with the CJK-font traps, Playwright install traps, and lazy-load traps already solved for you
- **New-car deep-dive**: full official config table scraping (div-structure parsing, not `<table>`) to CSV+JSON+MD

## Why not just ask ChatGPT?

| Common failure | This skill |
|---|---|
| Images show the wrong car (real case: a Mercedes GLE story with a YangWang U8 photo) | Vision audit drops it before publish |
| "Red slogan banner" political-propaganda images as "industry news" photos | PIL red-background rule (>40% coverage) |
| One-sentence news with no substance | Hard gate: 3-5 numbered bullets with concrete figures |
| Markdown dump nobody reads on a phone | 1080px PNG long-image, WeChat-ready |
| Same photo reused across 4 different-brand items | md5 dedup across the whole issue |

## Install

**Hermes / Claude Code / any Agent-Skills-compatible harness:**

```bash
# copy the skill directory into your skills path
cp -r skill/ ~/.hermes/skills/chinese-auto-weekly/   # Hermes
# or ~/.claude/skills/chinese-auto-weekly/            # Claude Code
```

No extra dependencies for the methodology; for the render pipeline you need `playwright` + Chromium and one static CJK OTF font (see SKILL.md §6 — the exact traps and fixes are documented).

## Files

```
skill/SKILL.md              ← the skill (drop into any skills directory)
examples/data.sample.json   ← real (trimmed) issue data
examples/report.sample.html ← rendered output (open in browser)
examples/images/            ← sample vehicle images
```

## Results from 10+ weeks of real runs

- Vision audit caught **6 wrong-model images out of 23** in a single issue (including an ID. AURA T6 card carrying a BMW i3)
- PIL rules auto-rejected red slogan banners (73% red coverage) while keeping red-hulled carrier-ship photos (29%) — the threshold is measured, not guessed
- PNG pipeline survived: zero-CJK-font systems, Playwright CDN hangs, lazy-load holes, and base64-bloat from uncompressed official 5-8MB images

## 中文说明

本仓库是一个可直接安装的 Agent Skill：让 AI 每周自动产出**可发布级**的汽车行业图文周报。

- **六段式结构**：竞品动态 / 新车上市 / 政策补贴 / 关税贸易 / 企业财报 / 行业大事件
- **抓图三原则**：按车型独立搜图、准确优先于有图、全局 md5 去重
- **图片双重质检**：PIL 规则初筛（横幅/logo/红底标语）+ 视觉模型语义终审（车型置信度 ≥0.90 才采用）
- **渲染管线**：HTML 模板 + headless Chromium 长截图，中文字体/懒加载/大图压缩等坑已全部趟平
- **新车配置采集**：官方配置页 div 结构解析，产出 CSV+JSON+MD 三件套

安装：把 `skill/` 目录拷入你的 skills 路径即可（Hermes、Claude Code 及兼容 Agent Skills 标准的工具均可用）。

实战数据：单期 23 张图视觉终审揪出 6 张错图；红底标语横幅 73% 红色占比 vs 红色滚装船 29%，阈值来自实测而非猜测。

## License

MIT — use it, fork it, sell reports made with it. A star is appreciated if it saves you hours.
