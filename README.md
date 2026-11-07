# MoneyMoney — 2026 AI 蓝海项目探索与可行性评估仓库

本仓库用于系统化沉淀 2026 年高毛利、强防御壁垒的 AI 蓝海商业方向，并通过跨模型协同（Antigravity + Remote Claude Opus 5.5）进行定量可行性评估、立项筛选与 MVP 架构设计。

## Route A codebase (English)

Three Pay-Per-Event Apify actors backed by one shared, dependency-free core
(`permit_intel_core/`), built from `docs/CLAUDE_ROUTE_A_EXECUTION_PLAN_EN.md`.

| Path | What it is |
|---|---|
| [`permit_intel_core/`](./permit_intel_core/) | Shared engine (Python standard library only; the `apify` SDK is optional). It contains the source registry (`sources/registry.yaml`), the Socrata client, normalizers (E.164 phones, address, v1 record), the spec extractor, license joins, delta state, and JSON Schemas. |
| [`actors/permits-contractor-leads/`](./actors/permits-contractor-leads/) | Actor #1, Building Permits & Contractor Leads. PPE events `permit-record` and `contact-enriched-record`. |
| [`actors/contractor-license-radar/`](./actors/contractor-license-radar/) | Actor #2, Contractor License, Insurance & Bond Expiration Leads (WA L&I). PPE events `contractor-record` and `expiring-alert-record`. |
| [`actors/permit-history-lookup/`](./actors/permit-history-lookup/) | Actor #3, Permit History API. Runs in batch mode or as a Standby HTTP `/lookup` + MCP `/mcp` server. PPE event `lookup`, charged on matches only. |
| [`scripts/probe_sources.py`](./scripts/probe_sources.py) | Live endpoint, field and freshness probe. Writes [`probe_report.md`](./probe_report.md). |
| [`scripts/coverage_report.py`](./scripts/coverage_report.py) | Live fill-rate measurement and the Day-2 gate. Writes [`COVERAGE.md`](./COVERAGE.md) and refreshes the actor READMEs. |
| [`scripts/sync_core.sh`](./scripts/sync_core.sh) | Vendors the core into each actor before `apify push`. `--check` verifies the copies are current. |
| [`scripts/record_fixtures.py`](./scripts/record_fixtures.py) | Re-records the privacy-scrubbed offline fixtures. |
| [`assets/store_metadata.json`](./assets/store_metadata.json) | Store `title`/`description`/`seoTitle`/`seoDescription` for all 6 Actors, with the search-ranking rationale. `tests/test_store_assets.py` keeps it in sync with every `actor.json`. |
| [`scripts/generate_actor_icons.py`](./scripts/generate_actor_icons.py) | Deterministic Pillow generator for the 6 Store icons in [`assets/icons/`](./assets/icons/) (512x512 RGBA, 1024 supersampled). `--preview out.png` renders a dark/light contact sheet; `--check` only verifies. |
| [`PRICING.md`](./PRICING.md) | Competitor benchmark (live Store API) and the exact PPE configuration. |
| [`templates/`](./templates/) | n8n workflow, Make blueprint, Clay + MCP setup, and 5 community posts. |
| [`tests/`](./tests/) | Golden-set spec precision, normalization/schema/client/delta tests, and local runs of all 3 actors. |

```bash
python3 -m unittest discover -s tests -v      # offline: fixtures only
python3 scripts/probe_sources.py              # live: writes probe_report.md
python3 scripts/coverage_report.py            # live: writes COVERAGE.md
./scripts/sync_core.sh && cd actors/permits-contractor-leads && apify push
```

Optional: `export SOCRATA_APP_TOKEN=...`. The client then sends it as `X-App-Token` for higher rate limits. On Apify, set it as a secret environment variable in the Console.

## Apify Store PPE actors #4-#6 (English)

Three self-contained Pay-Per-Event actors, built from `docs/CLAUDE_APIFY_TOP5_EVAL_REPORT_EN.md`. Unlike the permit actors, they have no vendoring step: each `actors/<name>/` directory is its own Docker build context.

| Path | Apify name | PPE events |
|---|---|---|
| [`actors/ats-jobs-hiring-signals/`](./actors/ats-jobs-hiring-signals/) | `ats-jobs-hiring-signals-api`: jobs and company hiring signals from Greenhouse, Ashby, Lever, SmartRecruiters, Recruitee and Workday public board APIs | `job-record`, `company-hiring-summary` |
| [`actors/google-ads-transparency/`](./actors/google-ads-transparency/) | `google-ads-transparency-advertiser-leads`: advertiser checks and ad creatives from the Google Ads Transparency Center RPCs, with a labeled, never-charged verified-snapshot fallback | `advertiser-check`, `ad-creative` |
| [`actors/keyword-volume-seo-intel/`](./actors/keyword-volume-seo-intel/) | `keyword-search-volume-cpc-difficulty-api`: modeled volume, CPC, difficulty and intent from live Google/Bing/YouTube autocomplete, with optional bring-your-own DataForSEO for exact data | `keyword-metric` |

`src/runtime.py` is shared: each actor ships an identical copy, and a test fails if the copies drift.

```bash
pip install -r actors/ats-jobs-hiring-signals/requirements.txt pytest jsonschema
python -m pytest tests                       # offline: recorded real responses, Apify meta-schema checks, real-SDK PPE runs
LIVE_TESTS=1 python -m pytest tests          # plus live integration tests against the real upstreams
cd actors/ats-jobs-hiring-signals && python -m src.main --local-input '{"companies": ["greenhouse:stripe"]}'
python scripts/record_ads_snapshot.py        # re-record the Ads Transparency verified snapshot (live, paced)
python scripts/record_new_actor_fixtures.py  # re-record the offline ATS/keyword fixtures (live)
```

## 目录结构

- [`docs/BLUE_OCEAN_PROPOSALS_ZH.md`](./docs/BLUE_OCEAN_PROPOSALS_ZH.md)：7 大 AI 蓝海项目合集结构化总结（中文完整版，含痛点、产品形态、定价模型、壁垒与 2 周 MVP 切入点）。
- [`docs/CLAUDE_FEASIBILITY_PROMPT_EN.md`](./docs/CLAUDE_FEASIBILITY_PROMPT_EN.md)：发往远端 Claude Opus 5.5 的全英文可行性分析任务书与定量评分卡标准（100% English Prompt）。
- [`docs/CLAUDE_FEASIBILITY_ANALYSIS_EN.md`](./docs/CLAUDE_FEASIBILITY_ANALYSIS_EN.md)：远端 Claude Opus 5.5 (`--effort high`) 输出的 443 行完整英文可行性、单元经济模型与防坑分析报告。
- [`docs/CLAUDE_FEASIBILITY_ANALYSIS_ZH.md`](./docs/CLAUDE_FEASIBILITY_ANALYSIS_ZH.md)：Claude 深度可行性报告的中文精译、避坑拆解与 14 天 MVP 落地作战指南。
- [`docs/P1_P6_OPERATIONAL_PLAYBOOK_ZH.md`](./docs/P1_P6_OPERATIONAL_PLAYBOOK_ZH.md)：P1 与 P6 深度落地实操手册（做什么、如何做、卖给谁、地域群体、如何对接、公司注册与跨境收款全流程）。
- [`docs/SOLO_ZERO_TOUCH_BLUE_OCEANS_ZH.md`](./docs/SOLO_ZERO_TOUCH_BLUE_OCEANS_ZH.md)：单人可做、免商务对接、免注册公司的 5 大 AI 蓝海变现方向（平台托管、MoR 个人收款与投币式自助变现）。
- [`docs/CLAUDE_SOLO_ZERO_TOUCH_PROMPT_EN.md`](./docs/CLAUDE_SOLO_ZERO_TOUCH_PROMPT_EN.md)：发往远端 Claude Opus 5.5 的 S1–S5（单人·免对接·免开公司）可行性评估英文任务书。
- [`docs/CLAUDE_SOLO_ZERO_TOUCH_ANALYSIS_EN.md`](./docs/CLAUDE_SOLO_ZERO_TOUCH_ANALYSIS_EN.md)：Claude Opus 5.5 (`--effort high`) 对 S1–S5 模式的完整英文评估报告（含 Schema 设计与 14 天排期）。
- [`docs/CLAUDE_SOLO_ZERO_TOUCH_ANALYSIS_ZH.md`](./docs/CLAUDE_SOLO_ZERO_TOUCH_ANALYSIS_ZH.md)：Claude 对 S1–S5 评估报告的中文精译、避坑拆解与 `permit-core` 黄金技术栈指南。
- [`docs/CLAUDE_S1_PERMITS_DEEP_DIVE_PROMPT_EN.md`](./docs/CLAUDE_S1_PERMITS_DEEP_DIVE_PROMPT_EN.md)：发往远端 Claude Opus 5.5 的 S1（美国施工许可与承包商情报节点）专项研究英文任务书。
- [`docs/CLAUDE_S1_PERMITS_DEEP_DIVE_EN.md`](./docs/CLAUDE_S1_PERMITS_DEEP_DIVE_EN.md)：Claude Opus 5.5 (`--effort high`) 输出的 662 行 S1 专项深度研究英文原版报告。
- [`docs/CLAUDE_S1_PERMITS_DEEP_DIVE_ZH.md`](./docs/CLAUDE_S1_PERMITS_DEEP_DIVE_ZH.md)：S1 专项深度研究中文精译 + Apify Store 官方 API 实时查验数据交叉核对报告。
- [`docs/CLAUDE_ROUTE_A_EXECUTION_PLAN_EN.md`](./docs/CLAUDE_ROUTE_A_EXECUTION_PLAN_EN.md)：路线 A 终极落地作战蓝图英文原版（Claude Opus 5.5 + Antigravity 联合制定）。
- [`docs/CLAUDE_ROUTE_A_EXECUTION_PLAN_ZH.md`](./docs/CLAUDE_ROUTE_A_EXECUTION_PLAN_ZH.md)：路线 A 终极落地作战蓝图中文完整版（含 3 台售货机矩阵、分工表、90 天里程碑）。
- [`scripts/run_kr_claude_feasibility.py`](./scripts/run_kr_claude_feasibility.py)：第一轮（P1–P7）远端 Claude 评估启动脚本。
- [`scripts/run_kr_claude_solo_eval.py`](./scripts/run_kr_claude_solo_eval.py)：第二轮（S1–S5 单人免对接模式）远端 Claude 评估启动脚本。
- [`scripts/run_kr_claude_s1_deep_dive.py`](./scripts/run_kr_claude_s1_deep_dive.py)：第三轮（S1 专项深挖）远端 Claude 评估启动脚本。
- `.remote-tasks/`：存放远端 Claude 评估输出结果与运行日志。
