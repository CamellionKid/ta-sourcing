---
name: ta-sourcing
description: "JD sourcing for recruiting benchmarking. Use when the user asks to source JDs, build a competitor hiring board, compare salary bands, or map a role across companies. Triggers on sourcing, JD 对标, 招聘看板, 岗位调研, 帮我找 XX 岗位在 XX 行业的 JD, TA-Sourcing. Two entry modes: by role (ask industry, then guided sub-direction refinement like 整机/灵巧手/关节/供应链, then region, then role) or by company (ask company name, then full-scrape all open positions). Default sources are 猎聘 and 官网; Boss 直聘 is anti-crawled and only a manual supplement. Outputs a Feishu Base kanban plus narrative doc plus local Obsidian note."
---

# TA-Sourcing

JD sourcing pipeline for recruiting and benchmarking work. Turns a role or company target into a structured Feishu Base kanban, a narrative doc, and a local Obsidian note.

## When to use

- The user says "sourcing", "JD 对标", "招聘看板", "岗位调研", "帮我找 XX 岗位在 XX 行业的 JD".
- The user asks to compare salary bands, job responsibilities, or requirements across companies.
- The user wants a repeatable way to build a competitor hiring board.

## Hard constraints

- **Boss 直聘 is anti-crawled.** Do not attempt automated Boss scraping. Treat Boss as a manual supplement only; if the user needs Boss data, ask them to export or screenshot manually.
- **Default sources: 猎聘 + 官网.** Use 猎聘 for salary bands and JD text; use 官网 (company career page / ATS) to confirm whether the role is officially posted and to capture official JD text when available.
- **No candidate names.** Only public job postings. Never write candidate names, recruiter names, or personal contact info into output.
- **Salary honesty.** Only compute 年包 when 月薪 and 年薪月数 are both present. If 月数 is missing, leave 年包 empty and write "月数未披露" in 备注. If the source has no salary, write "官网未披露" or "未披露".
- **Read-only public info.** Do not bypass CAPTCHA, do not log in, do not use stealthy_fetch on Boss.

## Entry flow — always ask first

Before any scraping, ask the user to pick one of two modes. Use the AskQuestion tool with 2 options plus an "Other" free-text fallback.

### Mode A: 按岗位 sourcing

Ask in this order, one question per AskQuestion call:

1. **行业** — recommend 1-2 options based on context (e.g. 具身智能, 智能硬件), plus Other for manual input.
2. **行业细分方向** — never search the top-level industry directly; it will miss component and supply-chain companies. After the user picks an industry, ask whether to break it down, offering the known sub-directions as multi-select plus Other:
   - 具身智能 → 整机本体 / 灵巧手 / 关节模组 / 减速器 / 电动夹爪与末端 / 视触觉传感器 / 微型电缸 / 制造 ODM 供应链
   - 智能硬件 → 影像 / 音频 / 整机品牌 / 零部件 / ODM 代工
   - 其他行业 → propose 3-5 sub-directions from market knowledge, then ask
   Default recommendation: 全选（整机 + 零部件 + 供应链都覆盖）. Companies like 舞肌科技、大寰机器人 only appear when 灵巧手/部件 is explicitly included — a top-level 具身智能 search returns mostly 整机厂.
3. **地域** — recommend 1-2 options (e.g. 深圳, 北京), plus Other.
4. **岗位** — recommend 1-2 options (e.g. 机械结构工程师, 结构工程师), plus Other.
5. **细化要求** (optional) — e.g. 学历, 年限. Only ask if the user wants filters.

Then build the company list:
- First read the user's local recruiting notes folder (ask for the path if unknown) for known companies in that industry.
- Build the list **per sub-direction**: for each selected sub-direction, list known companies from local notes, then supplement each with web search (e.g. "灵巧手 公司 深圳", "关节模组 厂商"). Do not stop at整机厂.
- Present the grouped company list (按细分方向分组) and confirm with the user before scraping. Ask explicitly: "这些细分方向的公司齐不齐？有没有要加的？"

### Mode B: 按公司 sourcing

Ask one question:

1. **公司** — recommend 1-2 options based on recent context (e.g. 自变量机器人, 银河通用), plus Other for manual input.

Then do a **full scrape of every open position at that company** — all job families (研发/算法/硬件/机械/产品/销售/职能等), not just 机械/结构. For each posting capture every field the source exposes: 岗位名称, 月薪, 年薪月数, 学历, 经验, 工作职责, 任职资格, 工作地, 发布时间, 招聘人数, 部门, 岗位类别.

- 猎聘: search `site:liepin.com <公司>` and open the company page (`liepin.com/company/<id>/`) to enumerate all postings, then open each JD page.
- 官网 Feishu ATS: call `POST /api/v1/search/job/posts` with empty keyword `{"keyword":"","limit":50,"offset":0}` and paginate (`count` field gives total) until all jobs are pulled.
- For non-机械 roles, leave 岗位族/赛道分层/部件类型 empty (those fields are mechanical-benchmark specific) and put the source's own job category into 备注, e.g. "岗位类别: 算法 / 感知".

## Sourcing workflow

1. **猎聘** — 按岗位模式: search `site:liepin.com <公司> <岗位> <城市>` via WebSearch, open the top JD pages with WebFetch. 按公司模式: search `site:liepin.com <公司>`, open the company page to enumerate all postings, then open each JD page. Extract: 岗位名称, 月薪, 年薪月数, 学历, 经验, 工作职责, 任职资格, 工作地, 发布时间.
2. **官网** — find the company career page. If it is a static page, read it directly. If it is a Feishu ATS (`*.jobs.feishu.cn`), use the public API `POST /api/v1/search/job/posts`; 按岗位模式 pass `{"keyword":"<岗位>","limit":50,"offset":0}`, 按公司模式 pass empty keyword and paginate via `count` until all jobs are pulled. If it is Moka (`jobs.*` or `app.mokahr.com`), try the public job list; if encrypted, note "官网列表加密" and fall back to 猎聘. If it is 智联 zhiye (`*.zhiye.com`), note that HTML does not expose job names; use 猎聘 as primary.
3. **Cross-check** — if 官网 and 猎聘 both have the same role, keep both records but mark 来源平台 accordingly. Use 官网 for official JD text, 猎聘 for salary.
4. **Normalize** — map to the field schema below. Compute 年包 only when 月薪 × 年薪月数 is known.

## Field schema

Write these fields to Feishu Base. Use the exact field names.

| 字段 | 类型 | 说明 |
|---|---|---|
| 岗位名称 | text | Official job title |
| 公司 | select | Company name; add option if missing |
| 城市 | select | 深圳 / 北京 / etc.; add option if missing |
| 工作地 | text | Detailed address or district |
| 岗位族 | select | 整机结构 / 机械臂 / 灵巧手/末端 / 关节/传动 / 结构仿真 / 外壳量产 / 线束结构；仅机械类岗位填，其他岗位留空 |
| 赛道分层 | select | 整机 / 部件 / 供应链；仅机械类岗位填，其他岗位留空 |
| 部件类型 | select | 整机本体 / 灵巧手 / 电动夹爪/末端 / 关节模组 / 减速器 / 视触觉传感器 / 制造ODM / 微型电缸；仅机械类岗位填，其他岗位留空 |
| 学历 | select | 大专 / 本科 / 统招本科 / 硕士 / 博士 |
| 经验要求 | text | e.g. "3年以上", "5-10年" |
| 工具/技能 | text | CAD, simulation, domain tools |
| 加分项 | text | Preferred qualifications |
| 工作职责 | text | Numbered list, one per line |
| 任职资格 | text | Numbered list, one per line |
| 月薪口径 | text | e.g. "25-50K", "官网未披露" |
| 年薪月数 | number | e.g. 14, 15, 16; empty if unknown |
| 年包下限(万) | number | 月薪下限 × 月数 / 10; empty if unknown |
| 年包上限(万) | number | 月薪上限 × 月数 / 10; empty if unknown |
| 来源平台 | select | 猎聘 / 官网 / Boss公司页 / 全职招聘网 / 一览英才网 |
| 来源链接 | url | Direct JD URL |
| 招聘状态 | select | 在招 / 暂停 / 仅卡片 |
| 抓取日期 | date | YYYY-MM-DD |
| 备注 | text | Data quality notes, e.g. "月数未披露", "官网列表加密" |

## Output artifacts

Always produce three artifacts:

1. **Feishu Base kanban** — create or reuse a Base in the user's Feishu folder. Table name `结构机械岗JD` or `<岗位>JD`. Create views: 按公司看板, 按岗位族看板, 按赛道分层看板, 按部件类型看板. Use `lark-cli base +record-batch-create` to write records. If a select option is missing, add it with `+field-update` before writing.
2. **Feishu narrative doc** — one doc per sourcing run. Structure: 背景 → 数据源边界 → 公司圈 → 读数 (salary bands, experience, education) → 增量说明. Use `lark-cli docs +update --command append`.
3. **Local Obsidian note** — write to the user's Obsidian recruiting folder as `YYYY-MM-DD <岗位>对标看板.md`. Include frontmatter `type: work-asset` and `tags: [招聘运营, 竞对对标]`, and link to the Feishu Base and doc. Append a line to today's daily note.

## Data quality rules

- Mark 招聘状态 = 暂停 if the posting date is older than 90 days or the page says 暂停.
- Mark 招聘状态 = 仅卡片 if only a search-result card is available, no JD body.
- If 官网 returns 0 jobs but the brand page mentions the role, note "官网有入口无正文" in 备注.
- If 猎聘 and 官网 JD text diverge, prefer 官网 for 工作职责/任职资格, prefer 猎聘 for 月薪/年薪月数.
- Never invent salary, headcount, or reporting line. Write "未披露" when absent.

## Boss supplement (manual only)

If the user later provides Boss screenshots or exports, treat them as a manual supplement source. Add a record with 来源平台 = Boss公司页, 来源链接 = the Boss company page URL, and 备注 = "Boss 手动补充". Do not automate Boss scraping.

## Example: 机械结构工程师 in 具身智能, 深圳

This is the reference run from 2026-09-01. See `references/example-embodied-structural.md` for the full company list, field values, and lessons learned.
