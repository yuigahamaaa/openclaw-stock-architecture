# OpenClaw 股票分析框架

> **项目代号**：龙虾（OpenClaw）
> **类型**：个人投资者级别的股票分析框架
> **部署**：NAS 7×24 运行
> **覆盖市场**：A 股 / 港股 / 美股
> **本文档**：简单逻辑简介，不含源代码

---

## 这是什么

一套部署在 NAS 上的股票分析框架，不是交易系统，而是 **研究 + 监控 + 通知** 系统。

目标：让个人投资者像机构一样有系统化的盘前 / 盘中 / 盘后支持。

---

## 四个核心场景

| 场景 | 时间 | 产出 | 推送方式 |
|---|---|---|---|
| **盘前** | 08:50 | 早报 PDF（隔夜全球市场 / 政策事件 / 持仓盈亏） | 微信 |
| **盘中** | 每 15 分钟 | 异动扫描 + 告警 | 微信 + 日志 |
| **盘后** | 00:00 / 15:35 / 15:40 | 晚报 / 关联研究 / 快评 | 微信 / 落盘 |
| **按需** | 手动 | 深度研究（13 章节基本面 / 68 章节深度 / 跨市场关联） | PDF 落盘 |

---

## 整体逻辑

```
┌─────────────────────────────────────────────────┐
│              CLI 入口（main.py）                │
│         38+ 子命令 · 薄壳分发                   │
└────────────────────┬────────────────────────────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ 研究子系统│ │ 监控子系统│ │ 报告子系统│
   └─────┬────┘ └─────┬────┘ └─────┬────┘
         │            │            │
         ▼            ▼            ▼
   ┌──────────────────────────────────────┐
   │           Services 业务层            │
   │  研究 · 告警 · T 计划 · 回测 · 通知  │
   └─────────────────┬────────────────────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ Repos    │ │ Providers│ │ Renderers│
   │ 数据访问 │ │ 跨市场契约│ │ PDF/MD   │
   └─────┬────┘ └─────┬────┘ └──────────┘
         │            │
         ▼            ▼
   ┌──────────────────────────────────────┐
   │        SQLite · DuckDB · YAML        │
   └─────────────────┬────────────────────┘
                     │
                     ▼
   ┌──────────────────────────────────────┐
   │            外部数据源                │
   │  腾讯 · 新浪 · 东财 · Tavily · SEC   │
   └──────────────────────────────────────┘
```

---

## 五条设计原则

1. **分层解耦** — Config → Repositories → Services → Renderers → CLI，任何一层可替换
2. **薄壳 CLI** — `main.py` 只做命令分发，业务逻辑在 `cli/` 子目录
3. **旁路开发** — 新功能一律新建文件，不修改生产代码
4. **默认 dry-run** — 写库需 `--write` 显式开启
5. **强类型 + 强门禁** — Pydantic v2 + ruff / pyright / pytest 四道门禁

---

## 主要命令（部分）

| 命令 | 用途 | 触发方式 |
|---|---|---|
| `morning-report` | 早报 PDF | cron 08:50 |
| `late-report` | 晚报 PDF | cron 00:00 |
| `watch` | 盘中异动扫描 | cron 每 15 分钟 |
| `quick-research` | 盘中快评（14 章 5.5K 字） | cron 15:40 / 手动 |
| `fundamental-research` | 基本面深度（68 章 16K 字） | 手动 |
| `relation-research` | 跨市场关联研究 | cron 15:35 / 手动 |
| `global-check` | 跨市场快评 | 手动 |
| `data-fetch` | 多源数据抓取 + 落库 | 手动 |
| `backtest` | 策略回测（5 策略） | 手动 |
| `intraday-notify` | 盘中通知（去重 + 多通道） | cron |

---

## 跨市场统一数据契约

2026-07-21 完成 A-J 十个工作包，统一 A 股 / 港股 / 美股的数据模型与 Provider 协议：

| 工作包 | 内容 |
|---|---|
| A | 领域模型（GlobalSecurityId / QuoteSnapshot / Bar / FinancialMetric） |
| B | Provider Protocol + Router |
| C-E | A 股 / 港股 / 美股 Adapter |
| F | SEC 财务 Adapter |
| G | 交易日历 |
| H | GlobalCheck CLI + Snapshot |
| I | Research Bundle + Evidence 适配 |
| J | 集成验收 + 迁移报告 |

**核心抽象**：
- 4 个 `typing.Protocol`（QuoteProvider / KlineProvider / FinancialProvider / CalendarProvider）
- `ProviderRouter` 按 `kind → market → interval → priority` 路由
- `DataResult[T]` 统一返回类型（ok / failed / quality / source）

---

## 数据源策略

| 数据 | 主源 | 备源 |
|---|---|---|
| A 股行情 | 腾讯 | 东财 |
| A 股 K 线 | mootdx | ohlcv.db |
| 港股行情 | 腾讯港股 | 新浪港股 |
| 美股行情 | 新浪美股 | 腾讯美股（双源 fallback） |
| 美股财务 | SEC EDGAR XBRL | Yahoo |
| 新闻（国内） | Tavily `search_china` | 多 key 池轮转 |
| 新闻（国外） | Tavily `search_global` | 多 key 池轮转 |

---

## 通知与推送

```
advice 生成
  → 6 字段指纹（code + action + direction + price + qty + time）
  → TTL 去重（30 分钟内同内容不重推）
  → 多通道发送（微信 + JSONL 日志）
  → 推送结果记录
```

---

## 测试与门禁

- **测试用例**：1800+ 个
- **4 道门禁**：ruff / pyright / py_compile / pytest
- **覆盖率**：Services + Repositories 层

---

## 部署形态

```
NAS (Synology)
  └── Stock_Research/
      ├── scripts/         ← 主代码（约 10 万行 Python）
      ├── data/ohlcv.db    ← 主数据库（SQLite）
      ├── config/          ← YAML 配置（watchlist / positions / events）
      ├── var/log/         ← 审计日志（JSONL）
      ├── venv/            ← Python 3.11
      └── pyproject.toml   ← 项目配置

OpenClaw cron 调度器（19 个任务）
  └── python main.py <command>
      └── 输出 PDF / JSON / 落库 / 微信推送
```

---

## 样例文档

为了展示项目的实际设计深度，我们在 [`samples/`](./samples/) 目录放了一份真实的设计文档样例：

- [数据自动抓取与落库机制_详细设计_样例.md](./samples/数据自动抓取与落库机制_详细设计_样例.md) — DataFetcher 子系统的完整设计文档（约 17K 字）

这份文档展示了：
- 问题陈述与约束
- 架构设计与数据流
- 模块清单与接口设计
- 多源校验机制
- 测试策略
- 风险与限制
- 实施步骤

---

## 不公开的内容

为了保护项目的核心竞争力，以下内容**不在本文档范围内**：

- 源代码（约 10 万行 Python）
- 具体的研究框架逻辑（GF-DMA / TAM-Adj-PEG / 13 章节 / 68 章节模板）
- 持仓信息与交易记录
- T 策略规则与阈值参数
- 微信推送通道的具体实现
- 个股认知库（`stock_knowledge.yaml`）
- 完整的 cron 调度配置
- 完整的数据库 schema

如果你是潜在合作方或研究者，想了解更多细节，请联系项目维护者。

---

## 技术栈

| 类别 | 技术 |
|---|---|
| 语言 | Python 3.11+ |
| 数据模型 | Pydantic v2 |
| 协议 | typing.Protocol + @runtime_checkable |
| DataFrame schema | Pandera |
| 数据库 | SQLite + DuckDB + Parquet |
| 回测 | backtesting.py + QuantStats |
| PDF 渲染 | ReportLab（Bento 风格） |
| Lint | ruff |
| 类型检查 | pyright |
| 测试 | pytest |
| 外部 API | 腾讯 / 新浪 / 东财 / mootdx / Tavily / SEC EDGAR / Yahoo |

---

## 许可

本文档仓库仅包含架构简介与设计文档样例，**不含任何源代码**。可自由阅读、引用、参考。

---

## 联系

项目维护者：OpenClaw Agent

最后更新：2026-07-21
