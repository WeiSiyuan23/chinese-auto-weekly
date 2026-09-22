---
name: chinese-auto-weekly
description: Produce a Chinese automotive industry intelligence weekly report (图文周报) — six-section layout covering competitor moves, new car launches, policy & subsidies, tariffs, earnings, and industry events — with vehicle-accurate images, HTML+PNG rendering, and vision-based image verification. Use when the user asks for 汽车行业周报, 车市周报, automotive weekly intel, competitor monitoring for a Chinese car brand, or any recurring Chinese-market industry digest with photos. Works with any watchlist brand (default reference implementation monitors BYD YangWang competitors).
license: MIT
metadata:
  version: "1.0.0"
  author: weisiyuan
  tags: [automotive, weekly-report, chinese, infographic, monitoring, html-render]
  category: research
---

# Chinese Auto Weekly（汽车行业图文周报管线）

Generate a publication-ready Chinese automotive industry weekly: a vertical 1080px HTML report + PNG long-screenshot, six fixed sections, every item backed by a source link, and every photo **verified to actually show the car being discussed**.

> 本 skill 是一条经过 10+ 周实战打磨的完整管线：信息采集 → 配图三原则 → 视觉终审 → 渲染截图。它把"周报"从"AI 写的新闻摘要"提升到"可直接发布的图文产品"。

## Why this skill exists（为什么需要专门的方法论）

LLM 写汽车周报有三个通病，本 skill 逐一解决：

| 通病 | 后果 | 本 skill 的对策 |
|---|---|---|
| 配图张冠李戴 | 长城条目配了别家车、财报通稿配红底标语横幅 | 按车型独立搜图 + PIL 规则初筛 + 视觉模型终审（§4） |
| 一句话新闻 | 读者没有任何收获 | 每条强制 3~5 条要点，含具体数字（§5） |
| Markdown 交付 | 手机上没法看、没法转发 | HTML 模板 + headless Chrome 出 PNG 长图（§6） |

## 0. Watchlist（关注清单）

Ask the user for their brand watchlist on first run, or use this battle-tested default (monitoring BYD 仰望 YangWang's competitors, excluding YangWang itself):

- **Direct — off-road/luxury SUV**: Tank 700/800, M-Hero 917, Land Rover Defender/Range Rover, Mercedes G-Class/EQG, Hummer EV, Land Cruiser EV
- **Direct — executive sedans**: NIO ET9, Zunjie S800 (Huawei), Hongqi Guoya, Mercedes EQS, BMW i7, Porsche Panamera
- **Direct — performance**: Zeekr 001 FR, Xiaomi SU7 Ultra, Lotus, Porsche Taycan
- **Indirect — premium NEV**: AITO M9, Li Auto L9/MEGA, NIO ES8, Zeekr 009, Denza Z9, Fangchengbao, Xiaomi SU7
- **Group-level signals**: BYD, Great Wall, Geely, Chery, NIO, XPeng, Li Auto, Seres, Tesla, Mercedes, BMW, Porsche

Store the watchlist in a config file so it survives across weeks.

## 1. Collection（三层采集）

**Layer 1 — RSS** (reliable, mostly English): Reuters Auto, Electrek, Autoblog, Car and Driver, CNBC Auto. Chinese auto media (autohome, dongchedi, gasgoo, d1ev) almost never publish RSS — don't fight it, use layers 2-3. Verify each feed with curl (browser User-Agent) before subscribing; many sites block bare curl.

**Layer 2 — Government/agency pages** (fetch weekly, extract last-7-day items):
- 工信部 miit.gov.cn（准入、产业政策）· 商务部 mofcom.gov.cn（以旧换新）· 财政部 mof.gov.cn（补贴、购置税）
- 国务院关税税则委 gacc.mof.gov.cn · 海关总署 customs.gov.cn（进出口）· 乘联会 cpca.org.cn（周度销量）· 中汽协 caam.org.cn

**Layer 3 — Keyword search** (fills the gaps): new launches / policy & subsidies / tariffs & anti-subsidy / earnings / CPCA sales / per-brand keywords. This layer does 70% of the work in practice.

## 2. Image sourcing — the three principles（抓图三原则，用户强约束）

> 配图必须准确贴合消息主体车型。准确 > 有图。

1. **Search per vehicle**: search the specific model by name (`"魏牌蓝山 2026"`), take images only from that model's dedicated coverage or model-gallery page. **Never** pull images from auto-show roundups, earnings press releases, or group announcements — their photos are routinely other models or slogan banners (battle-tested: one auto-show article contributed the same image to 4 different-brand items).
2. **Accuracy beats presence**: cannot confirm what's in the photo → leave the field empty and render a "暂无配图" placeholder. Never guess.
3. **Global dedup**: one image (by URL or md5) may serve exactly one item across the whole issue. After writing data.json, run an md5 self-check.

Image source priority: model gallery pages (Sohu 车系库 `db.auto.sohu.com/model_<id>/` serves official 360° photos) > official press releases > dedicated article body images.

**Download-time checks (PIL, zero extra deps)**: delete on fail — file <3KB or >8MB; width <300 or height <200 (thumbnail/logo); aspect ratio >5:1 or <1:5 (ad banner); quantized colors <100 (pure logo); white background >65% with <1200 colors; **red background >40% (red slogan banner — political-propaganda style images measured at 73% red coverage; a red bulk-carrier hull measures 29%, so the threshold separates them)**. Compress keepers to longest edge 1280 + JPEG q82 (the HTML inlines images as base64 data URIs, so oversized images bloat the file).

## 3. Vision verification（视觉语义终审）

Rules alone still let wrong-model photos through (real catches: a GLE story carrying a YangWang U8 photo, an ID. AURA T6 card carrying a BMW i3, a Li i9 story carrying a Li MEGA). Route every candidate image through a vision model:

- For vehicle items: keep only when `is_target_vehicle=true` with confidence ≥0.90; 0.80–0.89 → mark as backup; <0.80 → drop.
- For non-vehicle items (policy/earnings/industry): only drop slogan banners, logos, and AI-generated images.
- Cache verdicts by image md5 so re-runs don't re-bill. On API failure, keep the image conservatively and warn — never auto-delete on uncertainty.
- Budget: ~6s per image. Run `--dry-run` first, present the report, then apply.

## 4. Writing standards（写作标准）

- **Six fixed sections** (order and accent colors): 01 竞品动态 blue → 02 新车上市 green (card layout) → 03 政策法规与补贴 red → 04 关税与国际贸易 orange → 05 企业财报 purple → 06 行业大事件 cyan.
- Every item: date badge + title + one-line lead + **3–5 detailed bullets with concrete numbers** (dimensions, powertrain, price, range, sales) + source name + link. One-sentence news is a defect, not a style.
- Group competitor items: 直接竞品 (red tag) / 间接竞品 (blue tag) / 重点 (gold tag).
- New-car section uses product cards: image, name, price, launch date, one-line take.
- All-Chinese output. Numbers stay exact — "42.98 万元起", not "40 多万".

## 5. data.json contract

```json
{"title":"汽车行业周报","period":"2026.08.24 — 2026.08.30","generated_at":"ISO-8601",
 "stats":[{"label":"竞品动态","value":"8"}],
 "sections":[{"id":"competitors","num":"01","title":"竞品动态","accent":"blue",
   "groups":[{"group":"直接竞品 · 硬派越野","items":[
     {"date":"08-25","title":"...","summary":"导语","details":["要点1","要点2"],
      "source":"来源","url":"https://...","tag":"直接竞品","imgs":["images/xx.jpg"]}]}]},
  {"num":"02","title":"新车上市","accent":"green","cards":[
     {"img":"images/car1.jpg","date":"08-25","title":"车名","price":"42.98 万起",
      "summary":"...","details":["..."],"source":"...","url":"..."}]}]}
```

See `examples/data.sample.json` for a real (trimmed) issue and `examples/report.sample.html` for rendered output.

## 6. Rendering（HTML + PNG 长图）

For dense Chinese text, **always** HTML template + headless Chromium screenshot. Do **not** use AI image generation for the report body: CJK text renders unreliably (错字/乱码), style drifts week to week, and prompts aren't reusable. (This was a user-corrected decision — trust it.)

1. Render HTML from data.json with a light business template (dark tech variant optional). Inline local images as data URIs.
2. Screenshot at 1080px width, full height, via Playwright Chromium. **Chinese font trap**: systems often have zero CJK fonts (`fc-list :lang=zh` empty) → PNG is all tofu boxes. Install a **static OTF** (NotoSansCJKsc-Regular.otf into `~/.local/share/fonts/` + `fc-cache -f`). Variable-font TTFs are NOT parsed by headless Chrome — must be static OTF. If the template names PingFang SC/Microsoft YaHei, add a fontconfig alias mapping them to installed families.
3. **Playwright install trap**: `playwright install chromium` can hang on the chrome-headless-shell download. If the full chromium dir has `INSTALLATION_COMPLETE`, launch with explicit `executable_path` + `args=["--no-sandbox","--disable-dev-shm-usage"]` instead.
4. `loading="lazy"` is banned in the template — off-viewport images never load and the long screenshot ships with holes.
5. **Visual final check is mandatory**: inspect the PNG (thumbnail + full) for tofu text, broken images, content overflow. Real catch: source URLs overflowing card borders → fix with `.meta{flex-wrap:wrap}` + `.url{word-break:break-all}`.
6. Archive to `~/weekly-reports/<period>/`: `report.md` + `data.json` + HTML + PNG + `images/`.

## 7. New-car config deep-dive（新车配置全量采集，可选进阶）

After a launch event, capture the car's full config table (CSV+JSON+MD triple, with source URL):

1. Find the official config page (HIMA 系: `hima.auto/<brand>/<model>/configuration/`).
2. `web_extract` first, but the header row (model names + prices) is often missing — don't conclude from that; fetch the static HTML with urllib + browser UA (most official pages are server-rendered).
3. **The config table is NOT a `<table>`** — it's div structures (`line`/`line-head`/`mc-col`). Parse with per-column div-depth counting; regexing the whole block merges all 10 model columns into the first one. Header rows have 12 columns, data rows 10 — disambiguate by count.
4. Cross-validate 5–8 key specs (range/battery/power/0-100/ADAS hardware) against prose and media reports. Keep footnote markers (整备质量（kg）2) for traceability.

## 8. Operational traps（运维踩坑速查）

- **Tokenless/restricted sessions**: terminal may be hard-blocked (NOT_READY). Run everything via code-execution (`execute_code`) by importing script modules — don't retry terminal.
- Web backends flake (`Unrecognized MCP response shape`, CRAWL_TIMEOUT): vary the query and retry; if `web_extract` fails, pull HTML with urllib and extract `og:image` + `<img>` + **`data-src` lazy-load attributes**.
- Bing image search is anti-scraped (murl extraction returns empty) — don't use it; the three-principles sources above are better anyway.
- Tencent image CDN requires `Referer: https://news.qq.com/` or 403s.
- Government portal layouts change often and homepage links lack dates — treat Layer 2 as auxiliary; Layer 3 search is the backbone.
- Cron schedules: run in full-toolset sessions (file tools included); `HERMES_CRON_TIMEOUT` extends the default 600s; gateway must be running (`hermes gateway start`); a paused job must be resumed before `run`.

## 9. Quality gate（交付前自检清单）

- [ ] Six sections present, accents correct, no YangWang-self items in a YangWang watchlist
- [ ] Every item ≥3 bullets with numbers, every item has source + URL
- [ ] md5 dedup passed; no image serves two items
- [ ] Vision audit applied; all kept vehicle images ≥0.90 confidence
- [ ] PNG: no tofu text, no broken images, no overflow; width exactly 1080
- [ ] Archive complete (report.md / data.json / html / png / images/)
