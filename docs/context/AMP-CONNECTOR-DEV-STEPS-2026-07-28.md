# Amp Connector 开发步骤路由（2026-07-28 定稿）

> S0（上下文供给）的收尾产物。步骤梯子与交接协议经用户 2026-07-28 裁决定稿。每个步骤在**新 thread** 中进行，本文档是所有后续 thread 的入口路由。

## 开发主仓

- **[实测]** `https://github.com/1olaqua616-web/nowledge-mem-amp` — 用户 fork 自 `abn/nowledge-mem-google-antigravity`，默认分支 `main`，fork 时无自有提交（HEAD 与上游 2026-07-24 一致）。**公开仓**。
- 整体任务在此仓开 worktree 完成。
- **2026-07-28 更新（用户指示推仓）**：上下文文档已入库至 `docs/context/`（PR #7，含索引 README），入库后以仓内副本为准；本目录保留工作站原稿。步骤跟踪走 Issues：S0=#1（由 PR #7 关闭）、S1=#2、S2=#3、S3=#4、S4=#5、S5=#6。步骤产出通过 PR 交付并关联对应 issue。

## 步骤梯子

| 步骤 | 内容 | 产出 | 关键裁决 |
|---|---|---|---|
| **S0** 上下文供给 | 事实基底 + 三份读仓/能力面报告 + 本路由 | 见「文档清单」 | 已收尾 |
| **S1** 「为什么为 Amp 设计原生 connector」（2026-07-28 S1 thread 改口径） | 以 **Amp 能力面为基底**（FACTS §5/§7.5 实测），abn 与 pi 均为**对照样本**（不是基底、不是分析对象）。论证形状仿 abn 四层：价值面 / host 开了什么口子 / 为什么必须原生 / 不做的基线。注意：abn 的性能主论证（子进程 300–500ms）在 Amp 进程内范式下失效，不得引用 | 「为什么为 Amp 设计原生 connector」文档 | **不产出初版架构**；不做三栏清单、不做分诊清单（2026-07-28 裁决）；「是否需要」已由用户决定为「做」，S1 只论证「为什么」 |
| **S2** Product Requirements（Dex 一） | 问题陈述 + 成功标准，从 **S1 产出与 FACTS 重新推导** | PR 决议文档 | 旧 D1–D5 与原 Goals/Rules（DESIGN-CONTEXT）**全部废弃**（同源，2026-07-28 裁决），不折入、不继承 |
| **S3** System Architecture（Dex 二） | 通道取舍（extension/skills/MCP/AGENTS.md 各承担什么）、注入与抓取机制、nmem HTTP 合同、session 内切换 | 架构文档（图 + 合同形状） | 输入：S1 前提清单 + pi 事件对照表（PI-…READING §7） |
| **S4** Program Design（Dex 三） | 代码形状：类型、函数签名、文件布局、调用栈树（伪代码可视化） | 程序设计文档 | — |
| **S5** Vertical Slices（Dex 四） | 切片序列，每片可独立触摸验证、100–200 行可 review，逐片实现 | 切片计划 + 代码（进 fork 仓） | 可能多个 thread |

## 交接协议（v2，2026-07-28 用户裁决，取代 /handoff skill 流程）

1. **不使用 /handoff skill。**
2. **接收方定义合同**：每个新 thread 开始后，**先定义本步骤的验收标准和 handoff（交付物格式与去向），经用户批准，再开始工作**。
3. 步骤产出 = 落盘的日期文档（上下文类归本目录，代码类归 fork 仓）。
4. 闸门 = 用户批准本步产出后才开下一 thread；每 thread 只做一个步骤（单阶段原则）。
5. 下游发现上游文档有误 → 回上游文档修正，不在下游 thread 内就地推翻。
6. **私密文本不入库**：开题 prompt、本地绝对路径、工作站布局等一律不得出现在推送到公开仓的任何文档/commit/issue/PR 中；仓内引用一律用 `docs/context/` 相对路径。

## 文档清单与状态

| 文档（本目录） | 状态 |
|---|---|
| `AMP-CONNECTOR-DESIGN-CONTEXT.md` | **废弃**（2026-07-28 裁决）— Goals/Rules 与 D1–D5 同源，一并废弃，不作为任何步骤输入。唯一保留物：Dex 四阶段流程（步骤梯子 S2–S5 依据），其原文见下方 Clippings 条目与 Dex Horthy 原始链接 |
| `AMP-CONNECTOR-FACTS-2026-07-27.md` | 有效 · 必读 — 事实基底（含 §7.5 实测增补与写路径修正） |
| `ABN-ANTIGRAVITY-PLUGIN-CODE-READING-2026-07-27.md` | 有效 · S1 必读 — abn 插件源码通读 |
| `PI-CONNECTOR-CODE-READING-2026-07-28.md` | 有效 · S1/S3 必读 — pi 官方 connector + 事件面对照 |
| `docs/context/amp-plugin-architecture.md` | 有效 — Amp 插件架构（用户产出，已实测核对） |
| `AMP-CONNECTOR-DISCUSSION-HANDOFF-2026-07-27.md` | **废弃** — D1–D5 全部废弃（2026-07-28 裁决），不折入任何步骤 |
| `Clippings/Why Software Factories Fail…`（仓库根 Clippings/） | 有效 — Dex 四阶段流程原文 |

## 新 thread 开题

开题 prompt 属私密文本，不入库；由工作站侧路由原稿提供。仓内工作以本文档 + Issues #2–#6 为入口。
