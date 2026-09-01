# ta-sourcing

JD sourcing skill for AI agents (Claude Code / Cursor / Codex / Gemini). Turns a role or company target into a structured hiring-benchmark kanban: Feishu Base board + narrative doc + local Obsidian note.

Generalizable to any JD benchmarking task.

## What it does

Given a sourcing target, the skill:

1. **Asks the entry mode first** — by role or by company.
   - **By role**: asks industry → **guided sub-direction refinement** → region → role → optional filters. The sub-direction step is mandatory: a top-level industry search (e.g. 具身智能) returns mostly 整机厂 and misses component makers (灵巧手/关节模组/减速器/供应链). The skill offers known sub-directions as multi-select (具身智能 → 整机本体/灵巧手/关节模组/减速器/夹爪末端/视触觉传感器/微型电缸/制造 ODM; 智能硬件 → 影像/音频/整机品牌/零部件/ODM), then builds the company list per sub-direction and confirms coverage before scraping.
   - **By company**: asks the company name, then **full-scrapes every open position** at that company (all job families, salary, JD, and every field the source exposes) — not just mechanical roles.
2. **Sources JDs from public data** — 猎聘 (Liepin) for salary bands and JD text; company official career sites (brand pages, Feishu ATS, Moka, Zhiye portals) for official postings.
3. **Normalizes into a 22-field schema** — job title, company, city, job family, track layer, component type, education, experience, skills, responsibilities, requirements, monthly salary, months per year, annual package floor/ceiling, source platform, source URL, hiring status, scrape date, notes.
4. **Delivers three artifacts** — a Feishu Base kanban (with company / job-family / track / component views), a Feishu narrative doc, and a local Obsidian note linked from the daily log.

## Hard constraints

- **Boss 直聘 is anti-crawled.** The skill never automates Boss scraping; Boss data is a manual supplement only.
- **Salary honesty.** Annual package is computed only when both monthly salary and months-per-year are known. Otherwise the field stays empty with a note.
- **No candidate data.** Only public job postings; no candidate names, recruiter names, or personal contacts.
- **Read-only public info.** No CAPTCHA bypass, no login, no stealth scraping of anti-bot sites.

## Source coverage notes (from the reference run)

| Source type | Behavior |
|---|---|
| 猎聘 | Primary source for salary bands and JD text |
| Feishu ATS (`*.jobs.feishu.cn`) | Public API `POST /api/v1/search/job/posts` works without login; returns JD body but no salary |
| Moka (`jobs.*`, `app.mokahr.com`) | Job list API is encrypted; mark "官网列表加密" and fall back to 猎聘 |
| 智联 zhiye (`*.zhiye.com`) | HTML does not expose job names; use 猎聘 as primary |
| Brand career pages | Often only an email or a form; note "官网有入口无正文" |

## Install

This is an agent skill (SKILL.md format). Drop it into your agent's skills directory:

```bash
# Claude Code
ln -s /path/to/ta-sourcing ~/.claude/skills/ta-sourcing

# Cursor
ln -s /path/to/ta-sourcing ~/.cursor/skills/ta-sourcing

# Codex
ln -s /path/to/ta-sourcing ~/.codex/skills/ta-sourcing
```

Or import `ta-sourcing.skill` (zip format) via your agent's skill installer.

## Usage

Trigger phrases: `sourcing`, `JD 对标`, `招聘看板`, `岗位调研`, `帮我找 XX 岗位在 XX 行业的 JD`, or `ta-sourcing`.

Example session:

```
User: 帮我 sourcing 机械结构工程师
Agent: 按岗位还是按公司？
User: 按岗位
Agent: 哪个行业？（具身智能 / 智能硬件 / 其他）
User: 具身智能
Agent: 哪个地域？（深圳 / 北京 / 其他）
User: 深圳
Agent: 具体岗位？（机械结构工程师 / 结构工程师 / 其他）
...
Agent: → Feishu Base kanban + narrative doc + Obsidian note
```

## Repo layout

```
ta-sourcing/
├── SKILL.md                                  # entry flow, field schema, sourcing workflow, data quality rules
└── references/
    └── example-embodied-structural.md        # reference run: 机械结构工程师 in 具身智能/深圳, company list + salary bands + lessons
```

## Reference run

The bundled example documents a real run (2026-09-01): 55 JD records across 21 companies in the embodied-AI hardware space, three rounds (整机 → 部件/供应链 → 官网核查), with observed salary bands from 猎聘 and official JD text from Feishu ATS portals.

## License

MIT
