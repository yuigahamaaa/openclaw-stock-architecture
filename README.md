# OpenClaw 股票分析架构

> **项目代号**：龙虾（OpenClaw）· Stock_Research
> **本文档**：纯架构说明，不含源代码
> **版本**：v1.0 · 2026-07-21

## 这是什么

这是一套部署在 NAS 上 7×24 运行的**个人投资者级别股票分析框架**，覆盖 A 股 / 港股 / 美股三大市场。它不是交易系统，而是**研究 + 监控 + 通知**系统。

## 核心能力

- **盘前**：早报 PDF 推送微信（隔夜全球市场 / 政策事件 / 持仓成本盈亏）
- **盘中**：15 分钟一次扫描异动，触发条件推送告警
- **盘后**：晚报 PDF / 批量研究 / T 计划生成
- **按需**：手动深度研究（13 章节基本面 / 68 章节深度研究 / 跨市场关联）

## 技术特征

| 特征 | 说明 |
|---|---|
| 多市场覆盖 | A 股 / 港股 / 美股统一数据契约 |
| 多源数据 | 腾讯 / 新浪 / 东财 / mootdx / Tavily / SEC EDGAR / Yahoo |
| 多通道通知 | 微信推送 + JSON Lines 日志 + 文件落盘 |
| 强类型 | Pydantic v2 + typing.Protocol + Pandera schema |
| 强门禁 | ruff / pyright / py_compile / pytest 四道门禁 |
| 强审计 | 数据落库带 source/fetched_at/confidence 三字段 |

## 规模

- 代码量：约 10 万行 Python
- 测试用例：1882 个
- 测试文件：76 个
- CLI 命令：38+ 个
- Cron 任务：19 个
- Skills：6 个（含自建）

## 文档

完整架构说明见 [ARCHITECTURE.md](./ARCHITECTURE.md)（22 章节 · 约 1500 行）。

涵盖：
1. 项目概述与设计哲学
2. 整体架构与数据流
3. 目录结构
4. 核心分层抽象（Config / Repositories / Services / Renderers / CLI）
5. 数据采集层（多源策略 / DataFetcher / Tavily 多 key 池）
6. 跨市场统一数据契约（A-J 十个工作包）
7. 行情与监控层（盘中子系统 / 8 个 intraday CLI）
8. 研究体系（quick / fundamental / relation / global）
9. 策略与回测层（5 策略 + QuantStats tearsheet）
10. 通知与推送层（Notifier 抽象 / 去重 / 微信验证）
11. 报告渲染层（Bento 风格 PDF）
12. 命令体系（38+ 命令完整对照表）
13. Skills 体系（6 skill 端点清单）
14. Cron 调度（19 任务）
15. 测试体系（1882 用例分布）
16. 4 道门禁
17. 配置文件（YAML / JSON schema）
18. 数据库 Schema（ohlcv.db / DuckDB / Parquet）
19. 跨市场能力
20. 已知限制与未来规划
21. 附录（术语表 / 命名规范 / 模块统计）

## 部署形态

```
NAS (Synology)
  └── /vol1/@apphome/trim.openclaw/data/workspace/Stock_Research/
      ├── scripts/         ← 主代码
      ├── data/ohlcv.db    ← 主数据库
      ├── config/          ← YAML 配置
      ├── var/log/         ← 审计日志
      └── venv/            ← Python 3.11

OpenClaw cron 调度器（19 个任务）
  └── python main.py <command>
      └── 输出 PDF / JSON / 落库 / 微信推送
```

## 设计哲学五原则

1. **分层解耦**：Config → Repositories → Services → Renderers → CLI
2. **薄壳 CLI**：main.py 只做命令分发
3. **旁路开发**：新功能不修改生产代码
4. **默认 dry-run**：写库需 `--write` 显式开启
5. **强类型 + 强门禁**

## 三条安全边界

1. 统一接口和参数，但不强行统一交易规则
2. 新模块全部旁路开发，验收前不修改生产路径
3. 任何主动建议默认关闭，需 `--on` 显式开启

## 许可

本文档仓库仅包含架构说明，**不含任何源代码**。可自由阅读、引用、参考。

## 相关文档

- [ARCHITECTURE.md](./ARCHITECTURE.md) — 完整架构文档（22 章节）
