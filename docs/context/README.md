# Amp Connector 项目上下文（S0 产物）

本目录是 Nowledge Mem × Amp connector 项目的上下文文档库，由 S0（上下文供给）阶段产出并入库。本仓 fork 自 [`abn/nowledge-mem-google-antigravity`](https://github.com/abn/nowledge-mem-google-antigravity)，上游源码为 S1 的对照样本（2026-07-28 口径：Amp 能力面为基底，abn 与 pi 均为对照）；后续按步骤将其改造为 `nowledge-mem-amp`。

## 阅读顺序

| 文档 | 作用 |
|---|---|
| [`AMP-CONNECTOR-DEV-STEPS-2026-07-28.md`](./AMP-CONNECTOR-DEV-STEPS-2026-07-28.md) | **入口路由**：步骤梯子 S1–S5、交接协议 v2。步骤跟踪见本仓 Issues #1–#6 |
| [`AMP-CONNECTOR-DESIGN-CONTEXT.md`](./AMP-CONNECTOR-DESIGN-CONTEXT.md) | **废弃**（2026-07-28）— Goals/Rules 与 D1–D5 同源废弃，不作为任何步骤输入；仅 Dex 流程出处保留。S2 输入改为 S1 产出 + FACTS |
| [`AMP-CONNECTOR-FACTS-2026-07-27.md`](./AMP-CONNECTOR-FACTS-2026-07-27.md) | 无推论事实基底，证据分级［实测/文档/二手］；§7.5 为 Amp plugin API 本机实测增补 |
| [`ABN-ANTIGRAVITY-PLUGIN-CODE-READING-2026-07-27.md`](./ABN-ANTIGRAVITY-PLUGIN-CODE-READING-2026-07-27.md) | 上游（abn Antigravity 插件）源码通读，S1 对照样本 |
| [`PI-CONNECTOR-CODE-READING-2026-07-28.md`](./PI-CONNECTOR-CODE-READING-2026-07-28.md) | 官方 `nowledge-mem-pi` connector 通读；§7 的 pi↔Amp 事件对照表是 S3 核心输入 |
| [`amp-plugin-architecture.md`](./amp-plugin-architecture.md) | Amp 插件架构（来自 amp docs，经本机 `amp plugins show-docs` 实测核对） |
| [`S1-WHY-AMP-NATIVE-CONNECTOR-2026-07-28.md`](./S1-WHY-AMP-NATIVE-CONNECTOR-2026-07-28.md) | **S1 产出**：为什么为 Amp 设计原生 connector（四层论证）；S2 主输入之一 |
| [`S2-PRODUCT-REQUIREMENTS-2026-07-28.md`](./S2-PRODUCT-REQUIREMENTS-2026-07-28.md) | **S2 产出**：问题陈述 P1–P3 + 成功标准 C1–C7 + 范围与待拍板项；C1–C7 为 S3 验收约束 |
| [`AMP-CAPABILITY-SURFACE-2026-07-28.md`](./AMP-CAPABILITY-SURFACE-2026-07-28.md) | 视图：Amp 能力面（connector 视角），八项能力 + 架构图 + C1–C7 映射 |
| [`S3-SYSTEM-ARCHITECTURE-2026-07-28.md`](./S3-SYSTEM-ARCHITECTURE-2026-07-28.md) | **S3 产出**：两通道架构（注入/抓取/HTTP 合同/切换/韧性），决策来源速查；前置实测见 FACTS §7.6 |

## 说明

- 文档中的本地绝对路径与开题 prompt 属私密文本，不入库；在本仓内工作时以 `docs/context/` 相对路径为准。
- Dex Horthy《Why Software Factories Fail: Turning the lights back on》为第三方文章，不随仓分发，见 [原文](https://x.com/dexhorthy/status/2081058573556306030)。
- 已废弃文档（旧 D1–D5 讨论 handoff）不入库，仅存于原工作站，状态记录见路由文档「文档清单」。
