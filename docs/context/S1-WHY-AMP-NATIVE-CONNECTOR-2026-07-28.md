# S1 · 为什么为 Amp 设计原生 connector（2026-07-28）

> S1 产出。口径（2026-07-28 S1 thread 裁决）：以 **Amp 能力面为基底**，abn（Antigravity 插件）与 pi（官方 connector）为**对照样本**；「是否需要」已由用户决定为「做」，本文只回答「为什么」。不含架构选型、通道取舍、hook 选择、文件结构、实现计划（S3/S4 领地）。
>
> 论证形状仿 abn 文章的四层：价值面 → host 开了什么口子 → 为什么必须原生 → 不做的基线。
>
> 证据分级：[实测] 本机命令输出 / [文档] 官方文档原文 / [源码] 仓库源码通读 / [观察] 本文推断 / [未测] 待验证。出处指向本目录三份 S0 报告（FACTS / ABN-READING / PI-READING）。

## 0. 一句话回答

nmem 对 Amp 的读路径已经通了，缺的不是接口，而是**自治**：今天没有任何记忆动作会自动发生，并且真实对话根本没有被保存。这两个缺口，加上「上下文运行时可变」，都只有插件层能补——MCP、skills、AGENTS.md 三者在原理上给不了。

## 1. 第一层：价值面（与 host 无关）

connector 必须兑现的价值，官方口径三项（FACTS §3，[文档]，Autonomy Ladder 的 native connector 档）：

1. **启动上下文**："Often automatic at session start or through host lifecycle"；
2. **会话内检索与写入**："Strongest path; may be hook-driven"；
3. **真实 thread 抓取**："Some hosts also add real automatic capture or real transcript save"。

官方决策树（[文档]）："install the most specific Nowledge path your host supports. Only fall back to generic MCP when there is no better package."

## 2. 第二层：Amp 开了什么口子（基底；FACTS §5/§7.5，[实测]/[文档]）

- 插件形态：进程内 TypeScript 常驻进程（"long-lived processes that may run for multiple threads concurrently"），Bun 执行，事件全部 thread-scoped；
- `agent.start`：**每次用户提交 prompt 触发**，可返回 message（拼在用户消息尾部，进 transcript；`display` 默认 false）；
- `agent.end`：自带本轮增量 `messages`；`thread.messages()` 可全量回读（单次 ≤20 条，分页）；
- `session.start`：fire-and-forget，**无注入能力**——启动注入必然搭线程首个 `agent.start`；
- `tool.call` / `tool.result`：allow / reject / modify / synthesize；
- `registerCommand` + `configuration`：插件可持有运行时状态；
- `onDispose`：全插件约 3 秒预算，crash/SIGKILL 不执行；
- 无插件级 MCP 注册 API（MCP 走 settings.json，本机已 connected [实测]）；无插件级 rules 通道；
- 血统："Amp's plugin API is inspired by pi's extension API"——与官方已有 native connector 的 pi 同族。

通道齐了（注入位、增量抓取位、工具拦截位、运行时状态位）——这回答「为什么是现在」。

## 3. 第三层：为什么必须原生——三条能力差

先排除一条不可用的理由：**abn 的性能论证在 Amp 失效**。abn 立论于外部进程 hook 每次 spawn 子进程的 300–500ms（ABN-READING §8 [源码]）；Amp 插件是常驻进程内代码，该前提不存在；pi 官方 connector 甚至让读路径主动走 CLI 子进程、给 8 秒总预算（PI-READING §5 [源码]）。原生的理由是**能力差**，不是速度差。

### 3.1 确定性触发：自治只能来自 host 无条件触发的事件

官方 Autonomy Ladder（FACTS §3 [文档]）：Direct MCP 档 "MCP tools alone do not create autonomy"；Reusable package 档 "Usually guided"。本机现状恰好停在这两档（FACTS §2 [实测]：MCP connected 56 tools + 7 个 generic skills）——一切记忆动作都要等 agent 自己想起来，或用户开口。

`agent.start` 由 host 每 prompt 无条件触发，不经过 agent 判断——这是 MCP（被动工具）、skills（按需教学）、AGENTS.md（静态文本）在原理上都给不了的东西。对照：官方 claude-code plugin 正是用 UserPromptSubmit 做确定性注入（FACTS §4 [文档+实测活样本]）。

### 3.2 真实 transcript 抓取：从零到有

今天 Amp 下的「存线程」是降级品：save-thread skill 自述 "Deprecated compatibility skill... must degrade honestly to a resumable handoff"（FACTS §2 [实测]）——存的是摘要，不是对话。

而抓取的两端都已就位：host 侧 `agent.end` 自带增量 messages、`thread.messages()` 可全量回读（[实测]）；服务端侧，pi 官方 connector 的活同步走通用 HTTP 合同（`POST /threads` + `POST /threads/<id>/append` 带 `deduplicate: true` + `idempotency_key`，PI-READING §4 [源码]），不依赖 per-host CLI importer——`nmem t sync --from amp` 不支持只影响历史回填（[实测]）。中间缺的那一段，恰好只有插件能补。[未测] `source:"amp"` 的服务端接收，见 §6。

### 3.3 上下文运行时可变

AGENTS.md 是静态文件，session 内不可变。插件持有运行时状态（`registerCommand` + `configuration` [文档]），且 `agent.start` 每轮触发——每一轮都可以重新决定注入什么。上下文从「启动时定死」变成「逐轮可决策」，这一能力只存在于插件层。

## 4. 第四层：不做的基线

不做原生 connector，Amp 永久停在 Autonomy Ladder 的 Reusable package + Direct MCP 两档（FACTS §2 对照 §3）：

- 启动无自动上下文（靠 agent 主动调 MCP 读）；
- 检索/保存 "Guided by rules, skills, or prompts"——依赖提醒与自觉；
- 线程 "Usually handoff-only or explicit-only"——真实对话持续丢失，不可追溯。

读路径已可用（`nmem --json context --source-app amp` 返回完整 bundle [实测]）反而加重反差：万事俱备，只差一个会自己动的东西。

## 5. 对照样本的分工记录

- **abn**：提供论证形状（价值面 → 口子 → 必要性 → 基线）与「测绘先于设计」的方法；其 Antigravity 方案细节不迁移，其性能论证不得引用（见 §3 开头）。
- **pi**：证明服务端写合同通用（§3.2 的关键一环），并示范与 Amp 同族 API 下官方 connector 的最小形态；其事件面与 Amp 的差异对照（PI-READING §7）是 S3 输入，不在本文展开。

## 6. 未测事实登记（2026-07-28 裁决：S1/S2 不做实测，S3 架构定稿前完成）

1. [未测] `POST /threads` 带 `source:"amp"` 服务端是否接收；
2. [未测] `agent.start` 注入物是否随 prompt 在 transcript 中累积（现为类型声明推断，FACTS §7.5 [实测·类型级]）；
3. [未测] plugin 在 Orb 内是否运行（现仅 builtin skill 措辞的间接证据，FACTS §5）。

## 7. 给 S2 的输入

问题表述口径（路由文档 S2 行）：Amp ↔ nmem 关系中缺三样——**缺自治**（§3.1）、**缺真实抓取**（§3.2）、**缺运行时可变**（§3.3）。S2 从本文 + FACTS 重新推导问题陈述与成功标准；成功标准须 shipping 后可读可判。本文不预设成功标准。
