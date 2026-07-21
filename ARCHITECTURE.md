# OpenClaw 股票分析架构

> **项目代号**：龙虾（OpenClaw）· Stock_Research
> **文档版本**：v1.0 · 2026-07-21
> **代码规模**：约 10 万行 Python · 1882 个测试用例 · 76 个测试文件
> **本文档**：纯架构说明，不含源代码

---

## 目录

1. [项目概述](#1-项目概述)
2. [设计哲学](#2-设计哲学)
3. [整体架构](#3-整体架构)
4. [目录结构](#4-目录结构)
5. [核心分层抽象](#5-核心分层抽象)
6. [数据采集层](#6-数据采集层)
7. [跨市场统一数据契约](#7-跨市场统一数据契约)
8. [行情与监控层](#8-行情与监控层)
9. [研究体系](#9-研究体系)
10. [策略与回测层](#10-策略与回测层)
11. [通知与推送层](#11-通知与推送层)
12. [报告渲染层](#12-报告渲染层)
13. [命令体系](#13-命令体系)
14. [Skills 体系](#14-skills-体系)
15. [Cron 调度](#15-cron-调度)
16. [测试体系](#16-测试体系)
17. [4 道门禁](#17-4-道门禁)
18. [配置文件](#18-配置文件)
19. [数据库 Schema](#19-数据库-schema)
20. [跨市场能力](#20-跨市场能力)
21. [已知限制与未来规划](#21-已知限制与未来规划)
22. [附录](#22-附录)

---

## 1. 项目概述

### 1.1 项目定位

OpenClaw（龙虾）是一套**个人投资者级别的股票分析框架**，部署在 NAS 上 7×24 运行，覆盖 A 股、港股、美股三大市场。它不是一个交易系统，而是一个**研究 + 监控 + 通知**系统，目标是：

- **盘前**：生成早报 PDF 推送到微信，包含隔夜全球市场、政策事件、持仓成本盈亏
- **盘中**：15 分钟一次扫描异动，触发条件时推送告警
- **盘后**：生成晚报 PDF、批量跑研究、生成 T 计划
- **按需**：手动触发深度研究（13 章节基本面 / 68 章节深度研究 / 跨市场关联）

### 1.2 核心特征

| 特征 | 说明 |
|---|---|
| **多市场覆盖** | A 股（sh/sz/bj）、港股、美股，统一数据契约 |
| **多源数据** | 腾讯 / 新浪 / 东财 / mootdx / Tavily / SEC EDGAR / Yahoo |
| **多通道通知** | 微信推送（cron announce）+ JSON Lines 日志 + 文件落盘 |
| **多研究模式** | 盘中快评 / 深度研究 / 关联研究 / 跨市场研究 |
| **多策略回测** | 5 个内置策略（趋势/网格/均线/动量/反转）+ QuantStats tearsheet |
| **多报告格式** | PDF（Bento 风格）/ Markdown / JSON / HTML |
| **强类型** | Pydantic v2 模型 + typing.Protocol 协议 + Pandera DataFrame schema |
| **强门禁** | ruff / pyright / py_compile / pytest 四道门禁 |
| **强审计** | 数据落库带 source/fetched_at/confidence 三字段，操作走 JSON Lines 审计日志 |

### 1.3 部署形态

```
┌─────────────────────────────────────────────────────────┐
│                    NAS (Synology)                       │
│  /vol1/@apphome/trim.openclaw/data/workspace/           │
│  └── Stock_Research/                                    │
│      ├── scripts/         ← 主代码（10 万行）            │
│      ├── data/ohlcv.db    ← 主数据库（SQLite）           │
│      ├── data/duckdb/     ← 列式存储（DuckDB）           │
│      ├── docs/            ← 设计文档                     │
│      ├── config/          ← YAML 配置                    │
│      ├── var/log/         ← 审计日志                     │
│      ├── venv/            ← Python 3.11 虚拟环境         │
│      └── pyproject.toml   ← 项目配置                     │
│                                                         │
│  OpenClaw cron 调度器（19 个任务）                       │
│  └── 调用 `python main.py <command>`                    │
│      └── 输出 PDF / JSON / 落库 / 微信推送               │
└─────────────────────────────────────────────────────────┘
                          ↓
                   微信 / 邮件 / 文件
```

### 1.4 运行用户

- **代码所有者**：`17673945870`（UID 1000）
- **运行用户**：`trim.openclaw`（UID 993）
- **权限策略**：关键目录 `chmod 777`，关键文件 `chmod 666`，避免跨用户写入问题
- **Python 环境**：venv 路径 `/vol1/@apphome/trim.openclaw/data/workspace/Stock_Research/venv/`

---

## 2. 设计哲学

### 2.1 五条核心原则

1. **分层解耦**：Config → Repositories → Services → Renderers → CLI，任何一层可独立替换
2. **薄壳 CLI**：`main.py` 只做命令分发，业务逻辑在 `cli/` 子目录，便于独立测试
3. **旁路开发**：新功能一律新建文件，不修改生产代码（生产 = A 股盯盘 + 微信 cron + 持仓账本 + ResearchPipeline）
4. **默认 dry-run**：所有写库操作默认不写，需 `--write` 显式开启
5. **强类型 + 强门禁**：Pydantic v2 + typing.Protocol + 4 道门禁，保证代码质量

### 2.2 三条安全边界

1. **统一接口和参数，但不强行统一交易规则** — 各市场交易规则独立处理
2. **新模块全部旁路开发** — 验收前不修改生产路径
3. **默认 off** — 任何主动建议（T 操作 / 加减仓）默认关闭，需 `--on` 显式开启

### 2.3 三条命名规范

1. **产品名优先**：CLI 命令用产品名（`quick-research`），保留开发名为 alias（`basic-deep-research`）
2. **时间戳备份**：修改文件前先备份（`.pre_<purpose>_<YYYYMMDD>_<HHMMSS>`）
3. **JSON Lines 日志**：所有审计 / 操作日志用 JSONL 格式，便于 `jq` 处理

---

## 3. 整体架构

### 3.1 分层架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     CLI 入口层（main.py）                    │
│   38+ 个子命令 · 薄壳分发 · self_contained_commands          │
└────────────────────────────┬────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  研究子系统     │ │  监控子系统     │ │  报告子系统     │
│  cli/*_research │ │  cli/intraday_* │ │  cli/*_report   │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│                     Services 业务层                         │
│  research_service · alert_service · t_plan_service          │
│  buy_plan_service · event_service · watchlist_service       │
│  backtesting_service · tech_indicator_service · ...         │
└────────────────────────────┬────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  Repositories   │ │   Providers     │ │   Renderers     │
│  数据访问层     │ │  跨市场契约     │ │  PDF/MD/JSON    │
│  kline_repo     │ │  adapters/      │ │  pdf_renderer   │
│  quote_repo     │ │  sources/       │ │  full_renderer  │
│  search_repo    │ │  calendars/     │ │  watch_renderer │
└────────┬────────┘ └────────┬────────┘ └─────────────────┘
         │                   │
         ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│                     数据存储层                              │
│  ohlcv.db (SQLite) · duckdb/ · config/*.yaml · var/log/     │
└─────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                     外部数据源                              │
│  腾讯 / 新浪 / 东财 / mootdx / Tavily / SEC EDGAR / Yahoo   │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 数据流（典型场景）

**场景 1：盘后晚报告生成**

```
cron (00:00) → main.py late-report
  → morning_data_service.fetch_all()       # 拉 5 只持仓数据
  → repositories.quote_repo.get_quote()    # 从 ohlcv.db 读
  → services.cost_tracking_service.calc()  # 算浮盈
  → renderers.pdf_renderer.render()        # 生成 PDF
  → 微信推送（cron announce → weixin）
```

**场景 2：盘中异动告警**

```
cron (15min) → main.py watch
  → quote_engine.fetch_realtime()         # 实时行情
  → services/alert_service.check()         # 阈值判断
  → services/notify/notifier.send()       # 去重 + 推送
  → 微信 + JSONL 日志
```

**场景 3：深度研究**

```
手动 → main.py fundamental-research 600362 江西铜业
  → cli/fundamental_research.py
  → services/research_service.run()       # 4 维度研究
  → services/tech_indicator_service.calc() # 22 项技术指标
  → render_research_to_pdf.py             # 68 章节 PDF
  → 落盘 daily/fundamental_research/<date>/
```

**场景 4：跨市场关联**

```
手动/cron → main.py relation-research 600362
  → cli/relation_research.py
  → services/relation_research/agents/    # M3-M6 多 agent
  → providers/adapters/us_quote.py        # 美股行情
  → repositories/search_repo.search_global()  # Tavily
  → render_research_to_pdf.py
```

---

## 4. 目录结构

```
Stock_Research/
├── scripts/                    # 主代码（约 10 万行）
│   ├── main.py                 # 统一入口（53KB · 38+ 命令）
│   ├── cli/                    # CLI 子命令（51 文件）
│   │   ├── quick_research.py
│   │   ├── fundamental_research.py
│   │   ├── relation_research.py
│   │   ├── global_research.py
│   │   ├── intraday_*.py       # 8 个盘中相关 CLI
│   │   ├── data_fetcher.py     # 数据抓取主类
│   │   ├── data_fetcher_cli.py # 数据抓取 CLI
│   │   ├── global_check.py     # 跨市场快评
│   │   ├── backtest_cli.py
│   │   ├── dashboard.py
│   │   └── ...
│   ├── services/               # 业务服务层（111 文件）
│   │   ├── research_service.py
│   │   ├── research_framework.py
│   │   ├── alert_service.py
│   │   ├── t_plan_service.py
│   │   ├── buy_plan_service.py
│   │   ├── event_service.py
│   │   ├── watchlist_service.py
│   │   ├── market_hours_service.py
│   │   ├── cost_tracking_service.py
│   │   ├── backtesting_service.py
│   │   ├── strategy_lib.py
│   │   ├── tech_indicator_service.py
│   │   ├── tearsheet_service.py
│   │   ├── serenity_alpha.py
│   │   ├── duckdb_store.py
│   │   ├── parquet_partitioner.py
│   │   ├── schemas.py          # Pandera DataFrame schema
│   │   ├── data_quality.py     # DataResult 抽象
│   │   ├── global_snapshot_builder.py
│   │   ├── research_snapshot.py
│   │   ├── batch_research.py
│   │   ├── morning_data_service.py
│   │   ├── jiangtong_intraday.py
│   │   ├── intraday/           # 盘中子模块
│   │   │   ├── advice_lifecycle.py
│   │   │   ├── feature_engine.py
│   │   │   ├── lifecycle_store.py
│   │   │   ├── risk_engine.py
│   │   │   ├── rule_engine.py
│   │   │   └── snapshot_builder.py
│   │   ├── notify/             # 通知子模块
│   │   │   ├── notifier.py
│   │   │   ├── dedup.py
│   │   │   ├── fingerprint.py
│   │   │   └── channels/
│   │   ├── research_v2/        # Research v2 (U3-R)
│   │   │   ├── data_bundle.py
│   │   │   ├── evidence_adapter.py
│   │   │   ├── models.py
│   │   │   ├── registry.py
│   │   │   ├── snapshot_factory.py
│   │   │   ├── storage.py
│   │   │   └── diff.py
│   │   ├── relation_research/  # 关联研究 (M3-M6)
│   │   │   ├── adapters/
│   │   │   ├── agents/
│   │   │   ├── backtest/
│   │   │   ├── contracts/
│   │   │   ├── orchestration/
│   │   │   ├── reporting/
│   │   │   └── research/
│   │   └── wechat_verify/      # 微信发送验证
│   ├── models/                 # Pydantic 数据模型
│   │   ├── account.py
│   │   ├── advice.py
│   │   ├── global_snapshot.py
│   │   ├── intraday.py
│   │   ├── market_data.py      # 跨市场统一模型
│   │   ├── market_financials.py
│   │   ├── market_requests.py
│   │   └── trading_constraints.py
│   ├── providers/              # 跨市场数据 Provider
│   │   ├── contracts.py        # 4 个 Protocol
│   │   ├── router.py           # ProviderRouter
│   │   ├── calendar_router.py
│   │   ├── financial_mapping.py
│   │   ├── account_provider.py
│   │   ├── intraday_quote_provider.py
│   │   ├── market_provider.py
│   │   ├── minute_kline_provider.py
│   │   ├── sector_provider.py
│   │   ├── trading_calendar_provider.py
│   │   ├── adapters/           # 9 个 Adapter
│   │   │   ├── a_stock_quote.py
│   │   │   ├── a_stock_kline.py
│   │   │   ├── a_stock_financial.py
│   │   │   ├── hk_quote.py
│   │   │   ├── hk_kline.py
│   │   │   ├── us_quote.py
│   │   │   ├── us_kline.py
│   │   │   └── sec_financial.py
│   │   ├── sources/            # 数据源
│   │   │   ├── sina_us.py
│   │   │   ├── sina_us_kline.py
│   │   │   ├── tencent_hk.py
│   │   │   ├── tencent_us.py
│   │   │   └── sec_xbrl.py
│   │   └── calendars/          # 交易日历
│   │       ├── a_stock.py
│   │       ├── us.py
│   │       └── hk.py
│   ├── repositories/           # 数据访问层
│   │   ├── config_repo.py
│   │   ├── kline_repo.py
│   │   ├── quote_repo.py
│   │   ├── search_repo.py     # Tavily 搜索
│   │   ├── search_key_pool.py # 多 key 池
│   │   ├── unified_quote_repo.py
│   │   └── watchlist_repo.py
│   ├── renderers/              # 渲染层
│   │   ├── full_renderer.py
│   │   ├── json_renderer.py
│   │   ├── pdf_renderer.py
│   │   ├── watch_renderer.py
│   │   ├── late_report_v5.py
│   │   └── md_to_pdf.py
│   ├── config/                 # YAML 配置
│   │   ├── events.yaml         # 突发事件
│   │   ├── positions.json      # 持仓
│   │   ├── stock_knowledge.yaml # 个股认知库
│   │   └── watchlist.yaml      # 自选股
│   ├── tests/                  # 测试（79 文件 · 1882 用例）
│   │   ├── market_data/        # A-J 契约测试
│   │   ├── research_v2/
│   │   ├── intraday/
│   │   ├── test_*.py
│   │   └── conftest.py
│   ├── diagnostics/            # 诊断
│   ├── _archive/               # 归档
│   └── cron_logs/              # cron 日志
├── data/                       # 数据
│   ├── ohlcv.db                # 主 SQLite 数据库
│   ├── duckdb/                 # DuckDB 列式存储
│   ├── all_stocks.json         # 全市场股票列表
│   └── fetch_progress.json
├── docs/                       # 设计文档
│   ├── SKILLS_INDEX.md
│   ├── CLI_SKILL_MAP.md
│   ├── Data_Fetcher_集成文档_v1.0_20260721.md
│   ├── 数据自动抓取与落库机制_详细设计_v1.0_20260721.md
│   ├── 盘中T策略与异动监控_详细设计_v1.0_20260721.md
│   └── design/
│       ├── futu_api_integration_design.md
│       └── pdf_architecture_design.md
├── analyses/                   # 分析输出
├── reports/                    # PDF 报告
├── output/                     # 通用输出
├── macro_analysis/             # 宏观分析
├── stock_analysis/             # 旧版股票分析
├── stock_analysis_v2/          # v2
├── Stock_Watcher/              # 盯盘器
├── var/log/                    # 审计日志
│   ├── data_fetcher/audit.jsonl
│   └── intraday/
├── work/                       # 工作目录
├── venv/                       # Python 虚拟环境
├── pyproject.toml              # 项目配置
├── monitor.sh                  # 监控脚本
├── monitor_parallel.sh         # 并行监控
└── send_all.sh                 # 批量推送
```

---

## 5. 核心分层抽象

### 5.1 Config 层（YAML 配置）

所有可变配置走 YAML，代码不硬编码：

| 文件 | 内容 | 用途 |
|---|---|---|
| `config/watchlist.yaml` | 自选股清单 | 盯盘 / 研究 / 报告 |
| `config/positions.json` | 持仓（code/qty/cost/source） | 成本盈亏 + T 计划 |
| `config/events.yaml` | 突发事件（伊朗冲突 / 关税 / 政策） | 事件关联研究 |
| `config/stock_knowledge.yaml` | 个股认知库（护城河 / 业务结构） | 研究报告引用 |

### 5.2 Repositories 层（数据访问）

封装所有数据访问，对外提供领域语义接口：

| Repository | 职责 | 数据源 |
|---|---|---|
| `QuoteRepository` | 行情读取 | ohlcv.db + 实时 API |
| `KlineRepo` | K 线读取 | ohlcv.db |
| `WatchlistRepository` | 自选股 CRUD | watchlist.yaml |
| `ConfigRepo` | 配置读取 | YAML |
| `SearchRepo` | Tavily 搜索（国内/国外） | Tavily API |
| `SearchKeyPool` | 多 key 池轮转 | 内存 |

### 5.3 Services 层（业务逻辑）

封装业务规则，不直接接触数据源：

| Service | 职责 |
|---|---|
| `AlertService` | 告警阈值判断 |
| `BuyPlanService` | 建仓计划 |
| `CostTrackingService` | 持仓成本盈亏 |
| `EventService` | 突发事件管理 |
| `MarketHoursService` | A 股交易时间判断（所有 cron 共用） |
| `TPlanService` | 跨日 T 计划 |
| `WatchlistService` | 自选股业务 |
| `ResearchService` | 研究三步骤（GF-DMA + TAM-Adj-PEG） |
| `ResearchFramework` | 研究框架（部分 not_implemented） |
| `BacktestingService` | 回测封装 |
| `StrategyLib` | 5 个策略（趋势/网格/均线/动量/反转） |
| `TechIndicatorService` | 22 项技术指标 |
| `TearsheetService` | QuantStats tearsheet 生成 |
| `SerenityAlpha` | Serenity Alpha 评估 |
| `DuckDBStore` | DuckDB 列式存储 |
| `ParquetPartitioner` | Parquet 分区写入 |
| `MorningDataService` | 早报数据汇总 |
| `JiangtongIntraday` | 江铜专属盘中逻辑 |
| `BatchResearch` | 批量研究 |
| `ResearchSnapshot` | 研究快照 |
| `GlobalSnapshotBuilder` | 跨市场 Snapshot 构建器 |
| `DataQuality` | DataResult 抽象（ok/failed/is_ok） |
| `Schemas` | Pandera DataFrame schema |

### 5.4 Renderers 层（输出渲染）

将业务数据渲染为各种格式：

| Renderer | 输出 | 用途 |
|---|---|---|
| `FullRenderer` | Markdown + PDF | 完整扫描报告 |
| `JsonRenderer` | JSON | API 输出 / 程序消费 |
| `WatchRenderer` | Markdown | 盘中手表版（精简） |
| `PdfRenderer` | PDF（Bento 风格） | 早报 / 晚报 / 研究报告 |
| `LateReportV5` | PDF | v5 晚报模板 |
| `MdToPdf` | PDF | Markdown → PDF 转换 |

### 5.5 CLI 层（命令入口）

`main.py` 是唯一入口，38+ 个子命令薄壳分发到 `cli/` 子目录：

```python
# main.py 命令注册模式
COMMANDS = {
    "watch": cmd_watch,           # 盘中异动（手表版）
    "full": cmd_full,             # 完整扫描
    "late-report": cmd_late_report,  # 晚报 PDF
    "morning-report": cmd_morning_report,  # 早报 PDF
    "research": cmd_research,     # 研究三步骤
    "quick-research": cmd_quick_research,  # 盘中快评
    "fundamental-research": cmd_fundamental_research,  # 深度研究
    "relation-research": cmd_relation_research,  # 关联研究
    "global-check": cmd_global_check,  # 跨市场快评
    "data-fetch": cmd_data_fetch,  # 数据抓取
    # ... 38+ 个
}

# 薄壳模式（cmd_xxx 只做分发）
def cmd_quick_research():
    from cli.quick_research import main as quick_main
    raise SystemExit(quick_main(sys.argv[2:]))
```

---

## 6. 数据采集层

### 6.1 多源数据策略

| 数据类型 | 主源 | 备源 | 容错策略 |
|---|---|---|---|
| A 股行情 | 腾讯 `qt.gtimg.cn` | 东财 | 主源失败降级 secondary |
| A 股 K 线 | mootdx | ohlcv.db 缓存 | 优先读库，无则拉网 |
| A 股财务 | 东财 / 新浪 | mootdx | 多源对比 |
| 港股行情 | 腾讯港股 | 新浪港股 | 单源失败抛 DataResult.failed |
| 美股行情 | 新浪美股 | 腾讯美股 | 双源 fallback + HTTP 403 降级 |
| 美股 K 线 | 新浪日 K | Yahoo | 优先新浪 |
| 美股财务 | SEC EDGAR XBRL | Yahoo | SEC 优先 |
| 新闻（国内） | Tavily search_china | - | 多 key 池轮转 |
| 新闻（国外） | Tavily search_global | - | 多 key 池轮转 |
| 宏观事件 | 手动维护 events.yaml | - | 人工录入 |

### 6.2 DataResult 统一返回类型

所有数据采集返回 `DataResult[T]`，统一成功/失败语义：

```python
class DataResult(BaseModel, Generic[T]):
    value: Any
    source: str
    quality: str  # primary / secondary / heuristic / default
    error: str | None = None
    fetched_at: datetime

    @classmethod
    def ok(cls, value, source, quality="primary"): ...
    @classmethod
    def failed(cls, source, error): ...
    def is_ok(self) -> bool: ...
    def is_failed(self) -> bool: ...
```

### 6.3 DataFetcher 子系统（v1.0 · 2026-07-21）

统一的多源抓取 + 落库 + 审计框架：

- **主类**：`cli/data_fetcher.py` · `DataFetcher`
- **CLI**：`cli/data_fetcher_cli.py`
- **委托模式**：`DataFetcher` 委托 A-J Provider（`AStockQuoteAdapter` / `HKQuoteAdapter` / `USQuoteAdapter`），不重新实现解析
- **TTL**：30 分钟内同 code+market 不重复抓（审计日志反向扫描）
- **置信度**：high（quote.primary + 3+ news）/ medium（部分缺失）/ low（全缺失）
- **审计日志**：`var/log/data_fetcher/audit.jsonl`（JSON Lines）
- **默认 dry-run**：需 `--write` 显式开启写库
- **接入点 flag**：4 个研究 CLI 加 `--auto-fetch` flag（默认 False）

### 6.4 多 key 池（Tavily）

`repositories/search_key_pool.py` 管理多个 Tavily key，避免单 key 配额耗尽：

- 轮转策略：按 key 使用量 + 配额上限选下一个
- 配额状态：`key_pool_status()` 查询
- 添加 key：`add_search_key(key, alias, monthly_limit)`

---

## 7. 跨市场统一数据契约

### 7.1 背景

2026-07-21 完成 A-J 十个工作包，统一 A 股 / 港股 / 美股的数据模型与 Provider 协议。

### 7.2 核心抽象（Agent A · 领域模型）

| 模型 | 文件 | 字段 |
|---|---|---|
| `GlobalSecurityId` | `models/market_data.py` | code / market / exchange / currency / canonical_key |
| `QuoteSnapshot` | `models/market_data.py` | security / as_of / price / previous_close / open / high / low / change_pct / volume / amount / currency / source / quality |
| `Bar` | `models/market_data.py` | security / timestamp / interval / open / high / low / close / volume / amount |
| `FinancialMetric` | `models/market_data.py` | security / name / period / value / unit / filed_at / source |
| `MarketDataRequest` | `models/market_requests.py` | security / as_of |
| `QuoteRequest` | `models/market_requests.py` | security / as_of |
| `QuoteBatchRequest` | `models/market_requests.py` | securities（去重） / as_of |
| `KlineRequest` | `models/market_requests.py` | security / interval / limit / as_of |
| `FinancialRequest` | `models/market_requests.py` | security / metrics / as_of |

**模型特性**：
- Pydantic v2 + `ConfigDict(extra="forbid", frozen=True)`（严格 + 不可变）
- 时区感知（所有 datetime 必须带 tzinfo）
- OHLC 一致性校验用 `model_validator(mode="after")`（避免 field_validator 顺序问题）
- 货币用 ISO 4217（CNY / HKD / USD）

### 7.3 Provider 协议（Agent B · Protocol + Router）

4 个 `typing.Protocol` + `@runtime_checkable`：

| Protocol | 方法 | Adapter |
|---|---|---|
| `QuoteProvider` | `fetch(request: QuoteRequest) -> DataResult[QuoteSnapshot]` | AStockQuoteAdapter / HKQuoteAdapter / USQuoteAdapter |
| `KlineProvider` | `fetch(request: KlineRequest) -> DataResult[list[Bar]]` | AStockKlineAdapter / HKKlineAdapter / USKlineAdapter |
| `FinancialProvider` | `fetch(request: FinancialRequest) -> DataResult[list[FinancialMetric]]` | AStockFinancialAdapter / SECFinancialAdapter |
| `CalendarProvider` | `is_trading_day(date, market) -> bool` / `next_trading_day(...)` | AStockCalendar / USCalendar / HKCalendar |

**ProviderRouter** 按 `kind → market → interval → priority` 路由：
- `route_quote(market)` → 选最高 priority 的 QuoteProvider
- `route_kline(market, interval)` → 选匹配 market+interval 的最高 priority
- unsupported 时区分 market/interval 分别抛 `UnsupportedMarketError` / `UnsupportedIntervalError`

### 7.4 Adapter 实现（Agent C-G）

| Adapter | 市场 | 数据类型 | 源 |
|---|---|---|---|
| `AStockQuoteAdapter` | sh/sz/bj | 行情 | 腾讯 |
| `AStockKlineAdapter` | sh/sz/bj | K 线 | mootdx / ohlcv.db |
| `AStockFinancialAdapter` | sh/sz/bj | 财务 | 占位（永远 missing） |
| `HKQuoteAdapter` | hk | 行情 | 腾讯港股 |
| `HKKlineAdapter` | hk | K 线 | 占位 |
| `USQuoteAdapter` | us | 行情 | 新浪 + 腾讯双源 |
| `USKlineAdapter` | us | K 线 | 新浪日 K |
| `SECFinancialAdapter` | us | 财务 | SEC EDGAR XBRL |

### 7.5 交易日历（Agent G）

| Calendar | 市场 | 节假日来源 |
|---|---|---|
| `AStockCalendar` | sh/sz/bj | 手动维护（节假日表） |
| `USCalendar` | us | 手动维护 |
| `HKCalendar` | hk | 手动维护 |

`CalendarRouter` 按 market 路由到对应 Calendar。

### 7.6 GlobalCheck CLI（Agent H）

`cli/global_check.py` + `services/global_snapshot_builder.py` + `models/global_snapshot.py`：

- 输入：code + market
- 输出：`MarketDataSnapshot`（quote + kline + financial + calendar 一站式快照）
- 用途：跨市场手动验证

### 7.7 Research Bundle（Agent I）

`services/research_v2/data_bundle.py` + `evidence_adapter.py`：

- 输入：code + market
- 输出：`ResearchDataBundle`（行情 + K 线 + 财务 + 新闻 + 事件）
- 用途：研究子系统的统一数据入口

### 7.8 集成验收（Agent J）

`tests/market_data/test_contract_integration.py`（28 用例）：

- A-J 全链路集成测试
- 跨市场数据流验证
- Provider fallback 链验证

---

## 8. 行情与监控层

### 8.1 quote_engine（行情引擎）

`scripts/quote_engine.py`（18KB）：

- 封装腾讯实时行情 API
- 输出 `IntradaySnapshot`（兼容旧 A 股生产盯盘）
- 5 只持仓批量拉取
- 失败重试 + 降级

### 8.2 IntradaySnapshot（盘中快照）

`models/intraday.py`：

- 单只股票的盘中快照
- 包含价格 / 涨跌幅 / 成交量 / 五档
- 与 `QuoteSnapshot`（A-J 契约）兼容：`MarketDataSnapshot.quote` 字段用 `QuoteSnapshot`

### 8.3 盘中子系统（services/intraday/）

| 模块 | 职责 |
|---|---|
| `advice_lifecycle.py` | 建议生命周期管理（生成 → 推送 → 过期） |
| `feature_engine.py` | 特征工程（涨跌幅 / 量比 / 换手率） |
| `lifecycle_store.py` | 建议存储（JSONL） |
| `risk_engine.py` | 风险评估（止损 / 异常波动） |
| `rule_engine.py` | 规则引擎（异动触发条件） |
| `snapshot_builder.py` | 快照构建器 |

### 8.4 盘中 CLI（8 个）

| CLI | 命令 | 职责 |
|---|---|---|
| `intraday_snapshot.py` | `intraday-snapshot` | 盘中快照 |
| `intraday_advice.py` | `intraday-advice` | 盘中建议 |
| `intraday_scan.py` | `intraday-scan` | 盘中 live 扫描 |
| `intraday_notify.py` | `intraday-notify` | 盘中通知（支持 `--all-positions` / `--dry-run` / `--channels`） |
| `intraday_audit.py` | `intraday-audit` | 当日推送审计 |
| `intraday_diff.py` | `intraday-diff` | 新旧推送差异（15min bucket 对齐） |
| `intraday_summary.py` | `intraday-summary` | 当日汇总（T 信号 / BLOCKED / dedup_rate） |
| `intraday_replay` | `intraday-replay` | 盘中回放 |

### 8.5 监控脚本

| 脚本 | 用途 |
|---|---|
| `monitor.sh` | 单进程监控 |
| `monitor_parallel.sh` | 并行监控（5 只持仓同时） |
| `send_all.sh` | 批量推送 |
| `policy_monitor.py` | 政策监控 |

---

## 9. 研究体系

### 9.1 研究体系全景

| 类型 | 命令 | 字数 | 章节数 | 耗时 | cron | 触发方式 |
|---|---|---|---|---|---|---|
| 盘中快评 | `quick-research` | 5.5K | 14 | 2.4s | 40 15 * * 1-5 | cron + 手动 |
| 深度研究 | `fundamental-research` | 16K | 68 | 3-5s（inline）/ 3-8min（subagent） | - | 手动 |
| 关联研究 | `relation-research` | - | M3-M6 | - | 35 15 * * 1-5 / 0 16 * * 0 | cron + 手动 |
| 全球研究 | `global-research` | - | - | - | - | 手动 |
| 跨市场快评 | `global-check` | - | - | - | - | 手动 |
| 研究三步骤 | `research` | - | GF-DMA + TAM-Adj-PEG | - | - | 手动 |
| 批量研究 | `batch-research` | - | 简版 5 只 | - | - | 手动 |
| 屏幕筛选 | `screen` | - | 行业粗筛 | - | - | 手动 |
| 深度框架 | `deep-research` | - | 13 章节（step 5-12 not_implemented） | - | - | 手动 |

### 9.2 quick-research（v3 · 盘中快评）

**定位**：日常盘后用，5.5K 字，14 章节

**数据源**：
- 本地 `ohlcv.db`（K 线 + 22 项技术指标）
- 腾讯实时行情（PE / PB / 市值）
- `config/stock_knowledge.yaml`（公司护城河 / 业务结构）

**14 章节**：① 公司概况 ② 财务摘要 ③ 估值 ④ 技术面 ⑤ 行业地位 ⑥ 资金流 ⑦ 新闻 ⑧ 事件 ⑨ 风险 ⑩ T 计划 ⑪ 加减仓建议 ⑫ 投资哲学 ⑬ 综合评分 ⑭ 操作建议

**模式**：
- 默认 off（不主动做 T / 加减仓）
- `--on` 开关启用主动建议

**改名记录**（2026-07-21）：v3 开发术语 `basic-deep-research` → 产品名 `quick-research`，旧名保留为 alias

### 9.3 fundamental-research（v4 · 深度研究）

**定位**：重大决策前用，16K 字，68 章节

**新增 5 个扩展维度**（vs v3）：
- 杜邦分解：净利率 × 周转 × 杠杆 = ROE
- 现金流质量：OCF/净利 五年走势
- 卡脖子 / 稀缺性（专项评估）
- 支撑阻力位：60 日高低价 / 季度多空 / 止损位
- 简化 DCF：FCF × 折现率 = 企业价值

**双模式**：
- `inline`（默认 · 30-60s）：CLI 内部直接调模板拼报告
- `subagent`（慢 · 3-8 分钟）：spawn `openclaw agent` 走完整 SOP
- `auto`（默认）：openclaw 不可用时自动降级 inline

**改名记录**（2026-07-21）：v4 开发术语 `deep-research-v4` → 产品名 `fundamental-research`

### 9.4 relation-research（M3-M6 · 跨市场关联）

**定位**：跨市场关联研究（A 股 ↔ 美股 ↔ 港股 ↔ 商品）

**M3-M6 四阶段**：
- M3：跨市场数据拉取（mootdx / 腾讯 / Tavily）
- M4：关联矩阵 + 相关性分析
- M5：场景模拟（油价 ±5% / 美元 ±2% / LME 库存变化）
- M6：报告渲染（Markdown + 关联图）

**子模块**（`services/relation_research/`）：
- `adapters/`：数据适配器
- `agents/`：多 agent 协作
- `backtest/`：回测
- `contracts/`：契约
- `orchestration/`：编排
- `reporting/`：报告
- `research/`：研究逻辑

**已配 cron**：
- `35 15 * * 1-5` 关联研究-盘后汇总-1535
- `0 16 * * 0` 关联研究-周日周报-1600

**覆盖 5 只持仓**：002156 通富 / 603993 洛钼 / 002038 双鹭 / 512880 券商 ETF / 600362 江铜

### 9.5 ResearchPipeline（v3 · 研究三步骤）

`research <code>` 默认走 ResearchPipeline：

- **GF-DMA**（Growth-Frame DMA）：增长框架
- **TAM-Adj-PEG**：调整后 PEG 估值
- 三步骤：① 数据拉取 ② 框架计算 ③ 报告生成

`--legacy` flag 降级走旧 `run_research_report`（兼容）

### 9.6 Research v2（U3-R · 研究快照）

`services/research_v2/` 子模块：

| 模块 | 职责 |
|---|---|
| `models.py` | ResearchSnapshot 数据模型 |
| `registry.py` | 研究类型注册 |
| `snapshot_factory.py` | 快照工厂 |
| `storage.py` | 快照存储 |
| `diff.py` | 快照差异（新旧对比） |
| `data_bundle.py` | ResearchDataBundle（Agent I） |
| `evidence_adapter.py` | Evidence 适配 |

### 9.7 13 章节深度框架（stock-analyzer skill）

通过子 agent 触发，不走 main.py：

1. 公司概况 2. 行业地位 3. 护城河 4. 财务深度 5. 业务分析 6. 技术分析 7. 市场情绪 8. 竞争格局 9. 投资风险 10. 估值判断 11. 投资哲学 12. 赔率胜率 13. 综合评分

**为什么不做 cron**：
- 内容太多（13 章 × 5 只票 = 80K 字/次）
- 跑全持仓每次 30 分钟+
- 适合"按需触发"

### 9.8 研究决策树

```
用户说"分析 XXX" / "研究 XXX" / "深度看 XXX"
   ↓
【首次】stock-analyzer skill（子 agent，16K 字报告）
【复踏 / 跳路径】fundamental-research（main.py 命令，68 章 16K 字，3-5s）

用户说"XXX 怎么样" / "盘中快评" / "快速看下"
   ↓
quick-research（main.py 命令，14 章 5.5K 字，盘后 cron）

用户说"XXX 跟 XX 关联" / "LME 怎么看" / "海外对 XXX 影响"
   ↓
relation-research（M3-M6）→ 如需美股关联 → global-stock-data

用户说"海外对 XXX 影响"（涉及美股/港股具体标的）
   ↓
global-stock-data skill → global-research / global-check

默认日报 / 早晚报 / 盘中异动
   ↓
a-stock-data + 自建 renderers（main.py 已落地）
```

---

## 10. 策略与回测层

### 10.1 5 个内置策略

`services/strategy_lib.py` 基于 `backtesting.Strategy`：

| 策略 | 类名 | 逻辑 |
|---|---|---|
| 趋势跟随 | `TrendStrategy` | 均线多头排列 + 量能确认 |
| 网格交易 | `GridStrategy` | 价格区间分档买卖 |
| 均线交叉 | `MaCrossStrategy` | MA5 / MA10 / MA20 交叉 |
| 动量 | `MomentumStrategy` | 涨幅排名 + 量比 |
| 反转 | `ReversalStrategy` | 超跌反弹 + RSI 背离 |

### 10.2 BacktestingService

`services/backtesting_service.py`：

- 输入：code + strategy + period
- 输出：回测报告（收益 / 最大回撤 / 夏普 / 胜率）
- 数据源：ohlcv.db 历史数据

### 10.3 TearsheetService

`services/tearsheet_service.py`：

- 基于 QuantStats 生成 tearsheet
- 包含：收益曲线 / 月度热力图 / 滚动夏普 / 回撤分析
- 输出 HTML + PNG

### 10.4 backtest CLI

```bash
python main.py backtest --code 000977 --strategy trend --period 8y
```

### 10.5 多周期回测

`scripts/backtest_multiperiod.py`（17KB）：
- 多周期（1y / 3y / 5y / 8y）对比
- 多策略对比
- 输出汇总 PDF

### 10.6 Serenity Alpha

`services/serenity_alpha.py`：
- Alpha 评估（vs 基准）
- Serenity 指标（风险调整后收益）

---

## 11. 通知与推送层

### 11.1 Notifier 抽象

`services/notify/notifier.py`：

```python
class NotifyChannel(ABC):
    @abstractmethod
    def send(self, message: str, **kwargs) -> bool: ...

class Notifier:
    def __init__(self, channels: list[NotifyChannel]): ...
    def send(self, message: str, **kwargs) -> dict[str, bool]: ...
```

### 11.2 通道实现

`services/notify/channels/`：

| Channel | 用途 | 实现 |
|---|---|---|
| `LogChannel` | JSON Lines 日志 | 写入 `var/log/intraday/` |
| `WeChatChannel` | 微信推送 | stdout + 日志备份（cron announce → weixin） |

### 11.3 去重机制

`services/notify/dedup.py`：

- TTL 去重：同内容 30 分钟内不重复推送
- 指纹去重：6 字段指纹（code / type / direction / price / qty / time_bucket）

`services/notify/fingerprint.py`：

```python
def make_fingerprint(advice: Advice) -> str:
    # 6 字段: code + action_type + direction + price_bucket + qty + time_bucket
    # SHA256 取前 16 字符
```

### 11.4 推送流程

```
advice 生成
  → fingerprint 计算
  → dedup 检查（TTL + 指纹）
  → 如果未去重：
      → Notifier.send()
      → LogChannel 写 JSONL
      → WeChatChannel 推送（通过 cron announce → weixin）
  → 记录推送结果
```

### 11.5 微信推送验证

`services/wechat_verify/wechat_send_with_verify.py`：

- 发送前校验（key / channel）
- 发送后验证（落盘 + 回执检查）
- 失败重试（3 次）

---

## 12. 报告渲染层

### 12.1 PDF 渲染（Bento 风格）

`renderers/pdf_renderer.py`（v5.5.3 Bento 风格）：

- 基于 ReportLab + 自定义 Bento 布局
- 支持中文（思源黑体）
- 模块化：每个 Bento 块独立渲染
- 输出：PDF 文件

### 12.2 报告类型

| 报告 | 命令 | 模板 | 输出 |
|---|---|---|---|
| 早报 | `morning-report` | `morning_data_service` | PDF + 微信推送 |
| 晚报 | `late-report` | `late_report_v5.py` | PDF + 微信推送 |
| 盘中异动 | `watch` | `watch_renderer` | Markdown + 微信 |
| 完整扫描 | `full` | `full_renderer` | Markdown + PDF |
| 快评 | `quick-research` | 自定义 14 章 | PDF 落盘 |
| 深度研究 | `fundamental-research` | 自定义 68 章 | PDF 落盘 |
| 关联研究 | `relation-research` | M3-M6 | PDF + 关联图 |
| 持仓仪表盘 | - | `render_dashboard_pdf.py` | PDF |
| 江铜策略 | `jiangtong` | `render_jt_strategy_pdf.py` | PDF |
| AI 外网言论 | - | `render_ai_foreign_report.py` | PDF |
| AI 记忆 | - | `render_ai_memory_report.py` | PDF |

### 12.3 Markdown → PDF

`renderers/md_to_pdf.py`：

- 标准 Markdown → PDF 转换
- 支持代码块 / 表格 / 图片
- 中文友好

### 12.4 研究报告渲染

`render_research_to_pdf.py`（22KB）：

- 通用研究报告渲染
- 支持自定义章节模板
- 输出 PDF + Markdown 双格式

### 12.5 江铜专属

- `render_jt_strategy_pdf.py`：4 策略回测报告
- `render_jt_supplement.py`：补充材料
- `render_copper_tariff_watch.py`：铜关税盯盘

### 12.6 双鹭 / 长岳 / 双鹭补充

- `render_shuanglu_supplement.py`
- `render_zhangyue_supplement.py`

---

## 13. 命令体系

### 13.1 命令总览（38+ 个）

`main.py` 注册的 COMMANDS：

| # | 命令 | 入口 | 数据源 | cron |
|---|---|---|---|---|
| 1 | `watch` | `cmd_watch` | 腾讯 / 东财 | Stock-Watcher 15min |
| 2 | `full` | `cmd_full` | 腾讯 / 东财 | - |
| 3 | `alerts` | `cmd_alerts` | 腾讯 | - |
| 4 | `pnl` | `cmd_pnl` | ohlcv.db | - |
| 5 | `jiangtong` | `cmd_jiangtong` | 行情 | - |
| 6 | `reload` | `cmd_reload` | yaml | - |
| 7 | `check` | `cmd_check` | 腾讯 | - |
| 8 | `check-iran` | `cmd_check_iran` | Tavily | */5 9-11,13-14 * * 1-5 |
| 9 | `tplan` | `cmd_tplan` | 行情 | - |
| 10 | `tplan-json` | `cmd_tplan_json` | 行情 | - |
| 11 | `buyplan` | `cmd_buyplan` | 行情 | - |
| 12 | `buyplan-json` | `cmd_buyplan_json` | 行情 | - |
| 13 | `events` | `cmd_events` | 配置 | - |
| 14 | `events-json` | `cmd_events_json` | 配置 | - |
| 15 | `late-report` | `cmd_late_report` | 全数据 | 0 0 * * * |
| 16 | `morning-report` | `cmd_morning_report` | 全数据 | 50 8 * * * |
| 17 | `test-pdf` | `cmd_test_pdf` | - | - |
| 18 | `research` | `cmd_research` | a-stock-data | - |
| 19 | `batch-research` | `cmd_batch_research` | a-stock-data | - |
| 20 | `screen` | `cmd_screen` | a-stock-data | - |
| 21 | `deep-research` | `cmd_deep_research` | a-stock-data | - |
| 22 | `watchlist` | `cmd_watchlist` | yaml | - |
| 23 | `backtest` | `cmd_backtest` | ohlcv.db | - |
| 24 | `migrate-duckdb` | `cmd_migrate_duckdb` | SQLite → DuckDB | - |
| 25 | `position-import` | `cmd_position_import` | 同花顺 OCR | - |
| 26 | `intraday-replay` | `cmd_intraday_replay` | 内部 | - |
| 27 | `intraday-notify` | `cmd_intraday_notify` | 内部 | - |
| 28 | `global-check` | `cmd_global_check` | 美港股 | - |
| 29 | `data-fetch` | `cmd_data_fetch` | 多源 | - |
| 30 | `intraday-snapshot` | `cmd_intraday_snapshot` | 内部 | - |
| 31 | `intraday-advice` | `cmd_intraday_advice` | 内部 | - |
| 32 | `intraday-scan` | `cmd_intraday_scan` | 内部 | - |
| 33 | `intraday-audit` | `cmd_intraday_audit` | 内部 | - |
| 34 | `intraday-diff` | `cmd_intraday_diff` | 内部 | - |
| 35 | `intraday-summary` | `cmd_intraday_summary` | 内部 | - |
| 36 | `research-v2` | `cmd_research_v2_query` | ohlcv.db | - |
| 37 | `relation-research` | `cmd_relation_research` | 跨市场 | 35 15 * * 1-5 / 0 16 * * 0 |
| 38 | `quick-research` | `cmd_quick_research` | 本地 | 40 15 * * 1-5 |
| 39 | `fundamental-research` | `cmd_fundamental_research` | 本地 | -（手动） |
| 40 | `global-research` | `cmd_global_research` | 全球 | - |

### 13.2 self_contained_commands

部分命令标记为 `self_contained_commands`，表示它们直接透传到 `cli/*.py`，不走 main.py 的业务逻辑：

- `global-check`
- `data-fetch`
- `research-v2`
- `relation-research`
- `quick-research`
- `fundamental-research`
- `global-research`
- 等等

### 13.3 CLI 独立命令（不走 main.py）

`cli/` 目录下还有一批可独立运行的脚本：

| 命令 | 用途 |
|---|---|
| `cli/bulk_fetch_a_share.py` | 批量拉 A 股行情 |
| `cli/import_all_stocks.py` | 全市场股票导入 |
| `cli/incremental_update.py` | 增量更新 |
| `cli/db_backup.py` | 数据库备份 |
| `cli/dashboard.py` | Dashboard |
| `cli/extended_mappings.py` | 扩展映射 |
| `cli/profile_fetch.py` | 个股画像 |
| `cli/strategy_backtest.py` | 策略回测 |
| `cli/tech_bulk.py` | 技术指标批量 |
| `cli/watchlist_migrate.py` | watchlist 迁移 |
| `cli/watchlist_sync.py` | watchlist 同步 |

---

## 14. Skills 体系

### 14.1 Skills 总览

项目接入 6 个 skill（含自建）：

| Skill | 类型 | 大小 | 集成方式 | 用途 |
|---|---|---|---|---|
| `a-stock-data` | 数据 | 128K · 43 端点 | 直接 import | A 股全栈数据 |
| `global-stock-data` | 数据 | 53K · 8 层 | ❌ 未接入 main.py | 美港股全栈数据 |
| `stock-analyzer` | 研究 | 60K · 13 章节 | 子 agent 触发 | 一次性深度研究 |
| `uzi-skill` | 渲染 | - | ❌ 未接入 | Bloomberg HTML 风格（用户明示不接入） |
| `relation_research` | 自建 | 1910 行 | ✅ main.py | 跨市场关联 |
| `quick-research` | 自建 | - | ✅ main.py | 盘中快评 |
| `fundamental-research` | 自建 | - | ✅ main.py | 基本面深度 |

### 14.2 a-stock-data（A 股全栈数据）

**43 个端点，15 个通过实测**：

| 类别 | 端点 | 函数 |
|---|---|---|
| 行情 | 1.1 mootdx K线 + 五档 | `kline()` / `snapshot()` |
| 行情 | 1.2 腾讯 PE/PB/市值 | `tencent_quote()` |
| 行情 | 1.3 百度 K线 + MA | `baidu_kline()` |
| 研报 | 2.1 东财个股/行业研报 | `eastmoney_reports()` |
| 研报 | 2.2 同花顺一致预期 EPS | `ths_consensus_eps()` |
| 信号 | 3.1 同花顺热点 | `ths_hot_reason()` |
| 信号 | 3.2 同花顺北向资金 | `ths_hsgt()` |
| 信号 | 3.3 东财板块/概念 | `em_slist()` |
| 信号 | 3.4 东财个股资金流 | `em_push2_fund_flow()` |
| 信号 | 3.5 龙虎榜 | `dragon_tiger()` |
| 信号 | 3.6 解禁日历 | `limit_unlock()` |
| 信号 | 3.7 行业板块排名 | `industry_rank()` |
| 资金 | 4.1 融资融券 | `margin_detail()` |
| 资金 | 4.2 大宗交易 | `bulk_trade()` |
| 资金 | 4.3 股东户数 | `shareholder_count()` |
| 资金 | 4.4 分红送转 | `dividend_history()` |
| 资金 | 4.5 个股资金流 | `stock_fund_flow()` |
| 新闻 | 5.1 东财新闻 | `eastmoney_news()` |
| 新闻 | 5.3 全球资讯 7x24 | `global_news_724()` |
| 基础 | 6.1 mootdx 财务 | `mootdx_finance_snapshot()` |
| 基础 | 6.2 mootdx F10 | `mootdx_f10()` |
| 基础 | 6.3 东财基本面 | `em_push2_basic()` |
| 基础 | 6.4 新浪财报三表 | `sina_balance_sheet()` / `sina_income()` / `sina_cashflow()` |
| 公告 | 7.x 巨潮 | `cninfo_announcement()` |
| 打板 | 8.x 涨停池/连板 | `zt_pool()` / `limit_up_stats()` |
| 期权 | 9.x T 型报价 | `etf_option_chain()` |

### 14.3 global-stock-data（美港股全栈数据）

**8 层数据源**：

| 层 | 内容 | 端点示例 |
|---|---|---|
| 行情 | 美股 + 港股实时 | `us_quote_sina()` / `hk_quote_sina()` |
| K线 | 新浪 + Yahoo | `us_kline_sina()` / `us_kline_yahoo()` |
| 技术指标 | MA/MACD/RSI/KDJ/布林带/ATR | `tech_ma()` / `tech_macd()` |
| 基本面 | 东财 + Yahoo + SEC XBRL | `us_basic_em()` / `yahoo_quote_summary()` / `sec_xbrl()` |
| 资金流 | 东财日级 | `us_fund_flow_em()` |
| 期权 | Yahoo options | `yahoo_options()` |
| SEC Filing | EDGAR | `sec_filing()` |
| 搜索 | 东财 + Yahoo + SEC CIK | `search_global_em()` / `yahoo_search()` / `sec_cik_lookup()` |

### 14.4 stock-analyzer（13 章节深度研究）

**13 章节**：① 公司概况 ② 行业地位 ③ 护城河 ④ 财务深度 ⑤ 业务分析 ⑥ 技术分析 ⑦ 市场情绪 ⑧ 竞争格局 ⑨ 投资风险 ⑩ 估值判断 ⑪ 投资哲学 ⑫ 赔率胜率 ⑬ 综合评分

**触发方式**：手动触发子 agent → 16K 字报告

**不做 cron 的原因**：
- 内容太多（13 章 × 5 只票 = 80K 字/次）
- 跑全持仓每次 30 分钟+
- 适合"按需触发"

### 14.5 红线（不要动）

- 不动 skill 自身（a-stock-data / global-stock-data / stock-analyzer）
- 不动 main.py 主体（只新增薄壳函数 + COMMANDS 注册）
- 不动 services/research_framework.py（保持 not_implemented 标记）
- 不动现有 CLI 文件（仅新增）
- 不动 uzi-skill（按用户明示不接入）

---

## 15. Cron 调度

### 15.1 cron 总览（19 个任务）

OpenClaw cron 调度器管理 19 个任务：

| ID | 名称 | Schedule | Delivery | 命令 |
|---|---|---|---|---|
| `bef9c0db` | 早报-PDF-0850 | `50 8 * * *` | announce→weixin | `morning-report` |
| `aabb631d` | 晚报-PDF-0000 | `0 0 * * *` | announce→weixin | `late-report` |
| `5b0898bf` | 伊朗事件-8-23扫描 | `*/5 9-11,13-14 * * 1-5` | none | `check-iran` |
| `6648d7db` | 关联研究-盘后汇总-1535 | `35 15 * * 1-5` | none | `relation-research` |
| `01402086` | 关联研究-周日周报-1600 | `0 16 * * 0` | none | `relation-research` |
| `bdr1540-*` | 基本深度-盘后-1540 | `40 15 * * 1-5` | none | `quick-research --all-positions` |
| `4d69d6bb` | Stock-Watcher | `*/15 9-15 * * 1-5` | announce→weixin | `watch` |
| `5dffd09c` | Policy-Monitor | `30 8,12,18 * * *` | announce→weixin | 政策监控 |
| ... | （其他 11 个 cron） | - | - | - |

### 15.2 Delivery 模式

| 模式 | 含义 |
|---|---|
| `announce→weixin` | 推送到微信 |
| `none` | 仅落盘 / 日志，不推送 |

### 15.3 时间窗口

- **交易时段**：9:30-11:30 / 13:00-15:00（A 股）
- **盘前**：8:50（早报）
- **盘后**：15:35（关联研究）/ 15:40（快评）/ 16:00（周报）
- **晚报**：00:00（盘后汇总）

---

## 16. 测试体系

### 16.1 测试规模

- **测试文件**：76 个
- **测试用例**：1882 个
- **测试目录**：`scripts/tests/`

### 16.2 测试组织

```
scripts/tests/
├── conftest.py                 # pytest 配置
├── market_data/                # A-J 契约测试（384 用例）
│   ├── test_models.py          # Agent A
│   ├── test_router.py          # Agent B
│   ├── test_a_stock_adapters.py # Agent C
│   ├── test_us_adapters.py     # Agent D
│   ├── test_hk_adapters.py     # Agent E
│   ├── test_sec_financial.py   # Agent F
│   ├── test_calendars.py       # Agent G
│   ├── test_global_check.py    # Agent H
│   ├── test_contract_integration.py  # Agent J
├── research_v2/                # Research v2 测试
├── intraday/                   # 盘中子系统测试
├── test_data_fetcher.py        # DataFetcher 测试（40 用例）
├── test_backtest_cli.py
├── test_backtesting_service.py
├── test_basic_deep_research.py
├── test_batch_research.py
├── test_cli_routing.py
├── test_data_result.py
├── test_deep_research_v4.py
├── test_fetch_failure_hypothesis.py
├── test_kline_boundary.py
├── test_kline_hypothesis.py
├── test_kline_repo.py
├── test_m3_integration_smoke.py
├── test_m4_domain_agents.py
├── test_m5_model_adapters.py
├── test_m6_reporting.py
├── test_main_integration.py
├── test_orchestration.py
├── test_pandera_schemas.py
├── test_pdf_pipeline.py
├── test_relation_research_contracts.py
├── test_render_research_to_pdf.py
├── test_research_calendar_targets.py
├── test_research_event_rolling.py
├── test_research_features.py
├── test_research_framework.py
├── test_research_gating.py
├── test_research_probability_calibration.py
├── test_research_scoring_lifecycle.py
├── test_research_service.py
├── test_research_snapshot.py
├── test_search_key_pool.py
├── test_serenity_alpha.py
├── test_serenity_gating.py
├── test_tam_real_data.py
├── test_tearsheet_service.py
├── test_u1_public_contract.py
└── ...
```

### 16.3 测试策略

- **单元测试**：覆盖每个 Service / Repository / Provider
- **集成测试**：跨模块流程（如 test_contract_integration.py 覆盖 A-J 全链路）
- **端到端测试**：CLI 入口（如 test_cli_routing.py）
- **Mock 策略**：所有外部 HTTP 全 mock，用 `tmp_path` 替换 db_path / log_path

### 16.4 测试标记

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = [
    "network: tests requiring network access (deselect with '-m \"not network\"')",
]
```

跑无网络测试：`pytest -m "not network"`

### 16.5 coverage 配置

```toml
[tool.coverage.run]
source = ["scripts/services", "scripts/repositories"]
omit = ["*/tests/*", "*/venv/*", "*/__pycache__/*"]
```

---

## 17. 4 道门禁

### 17.1 ruff（lint + format）

```toml
[tool.ruff]
target-version = "py311"
line-length = 120

[tool.ruff.lint]
# E   pycodestyle errors
# F   pyflakes
# I   isort
# B   bugbear
# UP  pyupgrade
# SIM simplify
```

命令：`ruff check --fix <file>`

### 17.2 pyright（类型检查）

命令：`pyright <file>`

要求：0 errors（warnings 可接受）

### 17.3 py_compile（语法检查）

命令：`python -m py_compile <file>`

### 17.4 pytest（单元测试）

命令：`pytest tests/ -q`

要求：全绿

### 17.5 全量回归命令

```bash
source venv/bin/activate
cd scripts
ruff check cli/ services/ providers/ models/ repositories/ renderers/
pyright cli/ services/ providers/ models/ repositories/ renderers/
python -m py_compile $(find . -name "*.py" -not -path "*/__pycache__/*")
pytest tests/ -q
```

---

## 18. 配置文件

### 18.1 配置文件清单

| 文件 | 格式 | 内容 |
|---|---|---|
| `pyproject.toml` | TOML | pytest / ruff / coverage 配置 |
| `scripts/config/watchlist.yaml` | YAML | 自选股清单 |
| `scripts/config/positions.json` | JSON | 持仓（code/qty/cost/source/reconciliation_status） |
| `scripts/config/events.yaml` | YAML | 突发事件（伊朗冲突 / 关税 / 政策） |
| `scripts/config/stock_knowledge.yaml` | YAML | 个股认知库（护城河 / 业务结构） |
| `scripts/.env` | ENV | 环境变量（密钥等） |
| `scripts/.env.example` | ENV | 环境变量示例 |

### 18.2 watchlist.yaml 结构

```yaml
# 自选股清单
watchlist:
  - code: "600362"
    name: "江西铜业"
    market: "sh"
    role: "core"  # core / satellite
    tags: ["铜", "周期", "有色"]
  - code: "002156"
    name: "通富微电"
    market: "sz"
    role: "core"
    tags: ["半导体", "封测", "AMD关联"]
  # ... 5 只持仓 + 卫星股
```

### 18.3 positions.json 结构

```json
{
  "as_of": "2026-07-21",
  "reconciliation_status": "partial",
  "positions": [
    {
      "code": "600362",
      "name": "江西铜业",
      "market": "sh",
      "quantity": 0,
      "cost_price": 0,
      "source": "manual",
      "quality": "secondary"
    }
  ]
}
```

### 18.4 events.yaml 结构

```yaml
# 突发事件
events:
  - id: "iran_conflict_2026"
    name: "伊朗冲突"
    start_date: "2026-07-19"
    end_date: null
    impact_markets: ["oil", "copper", "a_stock"]
    related_codes: ["600362", "603993"]
    description: "中东局势升级，原油/铜价波动"
```

### 18.5 stock_knowledge.yaml 结构

```yaml
# 个股认知库
stocks:
  "600362":
    name: "江西铜业"
    moat: "铜矿资源 + 海外布局"
    business_structure:
      - segment: "铜矿开采"
        weight: 0.4
      - segment: "铜冶炼"
        weight: 0.5
      - segment: "其他"
        weight: 0.1
    key_drivers: ["LME铜价", "美元指数", "库存变化"]
```

---

## 19. 数据库 Schema

### 19.1 ohlcv.db（主 SQLite 数据库）

路径：`data/ohlcv.db`

#### 19.1.1 `stock` 表（元数据）

| 字段 | 类型 | 说明 |
|---|---|---|
| `code` | TEXT | 股票代码（主键） |
| `name` | TEXT | 股票名称 |
| `market` | TEXT | 市场（sh/sz/bj） |
| `role` | TEXT | 角色（core/satellite） |
| `last_updated` | TEXT | 最后更新时间 |
| `row_count` | INTEGER | K 线行数 |
| `first_date` | TEXT | 最早日期 |
| `last_date` | TEXT | 最新日期 |

#### 19.1.2 `fin_report` 表（财报长表）

| 字段 | 类型 | 说明 |
|---|---|---|
| `code` | TEXT | 股票代码 |
| `report_type` | TEXT | 报告类型（balance/income/cashflow） |
| `report_date` | TEXT | 报告日期 |
| `line_idx` | INTEGER | 行号 |
| `item_name` | TEXT | 科目名 |
| `item_value` | TEXT | 科目值 |
| `source` | TEXT | 数据源（审计字段） |
| `fetched_at` | TEXT | 抓取时间（审计字段） |
| `confidence` | TEXT | 置信度（审计字段） |

#### 19.1.3 `finance_snapshot` 表（财务快照长表）

| 字段 | 类型 | 说明 |
|---|---|---|
| `code` | TEXT | 股票代码 |
| `report_date` | TEXT | 报告日期 |
| `field_name` | TEXT | 字段名（如 latest_price / previous_close） |
| `field_value` | TEXT | 字段值 |
| `source` | TEXT | 数据源（审计字段） |
| `fetched_at` | TEXT | 抓取时间（审计字段） |
| `confidence` | TEXT | 置信度（审计字段） |

#### 19.1.4 `tech_indicator` 表（技术指标）

| 字段 | 类型 | 说明 |
|---|---|---|
| `code` | TEXT | 股票代码 |
| `date` | TEXT | 日期 |
| `ma5` / `ma10` / `ma20` / `ma60` | REAL | 均线 |
| `macd` / `macd_signal` / `macd_hist` | REAL | MACD |
| `rsi_6` / `rsi_12` / `rsi_24` | REAL | RSI |
| `kdj_k` / `kdj_d` / `kdj_j` | REAL | KDJ |
| `boll_upper` / `boll_middle` / `boll_lower` | REAL | 布林带 |
| `atr_14` | REAL | ATR |
| `volume_ratio_5d` | REAL | 量比 |
| ... | | 共 23 个指标 |
| `source` | TEXT | 数据源（审计字段） |
| `fetched_at` | TEXT | 抓取时间（审计字段） |
| `confidence` | TEXT | 置信度（审计字段） |

### 19.2 DuckDB（列式存储）

路径：`data/duckdb/`

用途：
- 大规模 K 线扫描
- 跨股票聚合分析
- 列式存储加速查询

迁移命令：`python main.py migrate-duckdb --dry-run`

### 19.3 Parquet 分区

`services/parquet_partitioner.py`：
- 按 year/month 分区
- 存储路径：`data/parquet/`
- 用于 Pandas / DuckDB 高效读取

---

## 20. 跨市场能力

### 20.1 市场覆盖

| 市场 | 代码格式 | 行情 | K 线 | 财务 | 交易日历 |
|---|---|---|---|---|---|
| A 股（sh/sz/bj） | 6 位数字 | ✅ 腾讯 | ✅ mootdx + ohlcv.db | ⚠️ 占位 | ✅ |
| 港股（hk） | 5 位数字 | ✅ 腾讯港股 | ⚠️ 占位 | - | ✅ |
| 美股（us） | 字母 | ✅ 新浪 + 腾讯双源 | ✅ 新浪日 K | ✅ SEC EDGAR | ✅ |

### 20.2 跨市场关联场景

| 场景 | A 股 | 海外关联 |
|---|---|---|
| 通富微电（002156） | 封测 | AMD / NVDA / TSM |
| 寒锐钴业（300618） | 钴业 | GLNCY / MP Materials |
| 江西铜业（600362） | 铜矿 | FCX / SCCO / LME 铜价 |
| 洛阳钼业（603993） | 钼业 | 海外钼矿 |

### 20.3 GlobalCheck 手动验证

```bash
python main.py global-check --code AAPL --market us --json
```

输出 `MarketDataSnapshot`（quote + kline + financial + calendar 一站式）

### 20.4 跨市场安全边界

1. **统一接口和参数，但不强行统一交易规则** — 各市场交易规则独立
2. **新模块全部旁路开发** — 验收前不修改 A 股生产路径
3. **旧 SecurityId 保留** — 27 个调用点不破坏，新建 `GlobalSecurityId`

---

## 21. 已知限制与未来规划

### 21.1 已知限制

| 限制 | 说明 | 后续 |
|---|---|---|
| 港股 K 线占位 | `HKKlineAdapter` 返回 missing | U6 阶段 |
| A 股财务占位 | `AStockFinancialAdapter` 永远 missing | U6 阶段 |
| Provider Health 完整实现 | 熔断 / 冲突仲裁未实现 | U6 阶段 |
| 双源一致性门禁 | 美股双源但未做价格对比 | U6 阶段 |
| 节假日表手动维护 | 未接 exchange_calendars | Phase 2 |
| `global-stock-data` skill 未接入 main.py | 仅子 agent 可用 | P2-4 集成 |
| `--auto-fetch` 未接入 run_* 函数 | flag 已注册但未调用 DataFetcher | 需手动接入 |
| 新闻不写库 | 只写审计日志 | Phase 2 |
| 审计日志无轮转 | 长期运行无限增长 | Phase 2 |
| Tavily 无配额预检查 | 耗尽时直接失败 | Phase 2 |

### 21.2 Phase 2 规划

- 接入 `exchange_calendars` 自动维护节假日
- 新闻表落库
- 审计日志 logrotate
- Tavily 配额预检查
- `global-stock-data` skill 集成到 main.py

### 21.3 Phase 3 规划

- 富途 API 集成（见 `docs/design/futu_api_integration_design.md`）
- 实时下单（当前仅研究 + 通知，不下单）
- 多组合管理（当前仅 5 只持仓）

### 21.4 U6 阶段规划

- Provider Health 熔断
- 双源一致性门禁（价格差 > 0.5% 降级 secondary）
- 港股 K 线真实接口
- A 股财务数据真实接口
- 完整 fallback 链

---

## 22. 附录

### 22.1 关键术语

| 术语 | 含义 |
|---|---|
| OpenClaw | 项目代号（龙虾） |
| A-J | 跨市场统一数据契约的 10 个工作包 |
| Agent A | 领域模型（GlobalSecurityId / QuoteSnapshot / Bar / FinancialMetric） |
| Agent B | Provider Protocol + Router |
| Agent C-G | 5 个 Adapter（A 股 / 美股 / 港股 / SEC / 交易日历） |
| Agent H | GlobalCheck CLI + MarketDataSnapshot |
| Agent I | Research Bundle + Evidence 适配 |
| Agent J | 集成验收 + 迁移报告 |
| U1-U5 | 历史迭代批次 |
| M3-M6 | 关联研究 4 阶段 |
| GF-DMA | Growth-Frame DMA（增长框架） |
| TAM-Adj-PEG | 调整后 PEG 估值 |
| DataResult | 统一数据返回类型（ok/failed） |
| IntradaySnapshot | 盘中快照（旧） |
| QuoteSnapshot | 行情快照（A-J 契约） |
| MarketDataSnapshot | 跨市场一站式快照 |
| ResearchDataBundle | 研究数据 Bundle |
| Bento 风格 | PDF 报告布局风格（v5.5.3） |

### 22.2 命名规范

| 规范 | 示例 |
|---|---|
| 产品名优先 | `quick-research`（产品名） > `basic-deep-research`（开发名 alias） |
| 时间戳备份 | `main.py.pre_data_fetch_20260721_203202` |
| JSONL 日志 | `audit.jsonl` / `cron_test.log` |
| 测试文件 | `test_<module>.py` / `test_<feature>.py` |
| 配置文件 | `*.yaml` / `*.json` |
| 审计字段 | `source` / `fetched_at` / `confidence` |

### 22.3 4 道门禁脚本

```bash
#!/bin/bash
# 4-gate-check.sh
source venv/bin/activate
cd scripts

echo "=== Gate 1: ruff ==="
ruff check cli/ services/ providers/ models/ repositories/ renderers/ tests/ || exit 1

echo "=== Gate 2: pyright ==="
pyright cli/ services/ providers/ models/ repositories/ renderers/ || exit 1

echo "=== Gate 3: py_compile ==="
python -m py_compile $(find . -name "*.py" -not -path "*/__pycache__/*") || exit 1

echo "=== Gate 4: pytest ==="
pytest tests/ -q || exit 1

echo "=== All gates passed ==="
```

### 22.4 典型 cron 配置示例

```json
// 早报 cron
{
  "id": "bef9c0db",
  "name": "早报-PDF-0850",
  "schedule": "50 8 * * *",
  "delivery": "announce→weixin",
  "command": "morning-report"
}

// 盘中扫描 cron
{
  "id": "4d69d6bb",
  "name": "Stock-Watcher",
  "schedule": "*/15 9-15 * * 1-5",
  "delivery": "announce→weixin",
  "command": "watch"
}

// 关联研究盘后 cron
{
  "id": "6648d7db",
  "name": "关联研究-盘后汇总-1535",
  "schedule": "35 15 * * 1-5",
  "delivery": "none",
  "command": "relation-research"
}
```

### 22.5 相关文档索引

| 文档 | 路径 | 说明 |
|---|---|---|
| SKILLS_INDEX.md | `docs/SKILLS_INDEX.md` | 6 个 skill 索引 |
| CLI_SKILL_MAP.md | `docs/CLI_SKILL_MAP.md` | 38+ 命令 ↔ skill 对照表 |
| 数据自动抓取与落库机制 | `docs/数据自动抓取与落库机制_详细设计_v1.0_20260721.md` | DataFetcher 设计文档 |
| Data Fetcher 集成文档 | `docs/Data_Fetcher_集成文档_v1.0_20260721.md` | DataFetcher 接入指南 |
| 盘中T策略与异动监控 | `docs/盘中T策略与异动监控_详细设计_v1.0_20260721.md` | 盘中子系统设计 |
| 跨市场统一数据契约设计 | `Stock_Research跨市场统一数据契约与多Agent并行实施设计_20260721.md` | A-J 工作包设计 |
| 跨市场统一数据契约迁移报告 | `跨市场统一数据契约_迁移报告_20260721.md` | A-J 验收报告 |
| 个股关联发现框架 | `个股关联发现与跨市场条件概率研究框架_详细设计_v1.3_20260721.md` | M3-M6 设计 |
| PDF 架构设计 | `docs/design/pdf_architecture_design.md` | PDF 渲染设计 |
| 富途 API 集成 | `docs/design/futu_api_integration_design.md` | 富途 API 设计 |
| U2 验收报告 | `U2_acceptance_report.md` | U2 迭代验收 |
| U3-U4 验收报告 | `U3_U4_acceptance_report.md` | U3-U4 迭代验收 |

### 22.6 模块文件数与代码规模

| 模块 | 文件数 | 说明 |
|---|---|---|
| `cli/` | 51 | CLI 子命令（含 helpers） |
| `services/` | 111 | 业务服务（含子模块） |
| `models/` | 9 | Pydantic 数据模型 |
| `providers/` | 30 | 跨市场 Provider（含 adapters/sources/calendars） |
| `repositories/` | 8 | 数据访问层 |
| `renderers/` | 7 | 渲染层 |
| `tests/` | 79 | 测试（76 文件 + conftest 等） |
| **合计** | **~295** | 约 10 万行 Python |

### 22.7 测试用例分布

| 测试类别 | 用例数 | 说明 |
|---|---|---|
| A-J 契约测试 | 384 | Agent A-J 验收 |
| DataFetcher 测试 | 40 | 数据抓取子系统 |
| 盘中子系统测试 | 64+ | intraday/notify/T3 CLI |
| 研究框架测试 | 100+ | research_service / framework / features |
| 回测测试 | 20+ | backtesting_service / strategy_lib |
| 渲染测试 | 20+ | pdf_pipeline / render_research |
| 集成测试 | 30+ | main_integration / orchestration |
| **合计** | **1882** | |

---

## 文档元信息

- **最后更新**：2026-07-21
- **文档版本**：v1.0
- **维护者**：OpenClaw Agent
- **许可**：本文档不含源代码，可自由分发
- **代码仓库**：本文档对应的代码为私有部署，不在本文档中包含

---

**文档结束**
