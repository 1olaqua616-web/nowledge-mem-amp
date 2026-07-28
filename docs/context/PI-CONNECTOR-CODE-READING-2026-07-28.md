# nowledge-mem-pi 源码 + pi extension API 通读（2026-07-28）

> 材料：npm `nowledge-mem-pi@0.8.5` tarball（官方 native connector）+ `badlogic/pi-mono` extensions.md（2961 行，Amp plugin API 的血统来源，Amp 官方 show-docs 自述 *"inspired by pi's extension API"*）。
> 证据等级：全部 **[源码]**，除标注 **[观察]**。
> 意义：这是「与 Amp 同范式（进程内 TS、单文件、事件驱动）的官方 connector 实现」，比 abn 的 Antigravity 插件（外部进程 hook 范式）更接近 Amp 的设计空间。

## 1. 包形态：比 abn 简得多

```
package/
├── extensions/nowledge-mem.ts   # 全部逻辑，815 行单文件
├── skills/                      # 5 个通用 skills（distill-memory / read-working-memory /
│                                #   save-thread / search-memory / status）
├── AGENTS.md / README.md
└── scripts/sync-history.mjs     # 历史回填脚本
```

**没有** MCP 捆绑、**没有**权限 gate、**没有** rules 目录、**没有**离线队列文件。对照 abn 的五通道，pi 官方连接器只有两通道：extension + skills。[观察] 两种范围都是官方认可的「native connector」——connector 的最小合同比 abn 案例展示的更小。

## 2. 事件接线（全部逻辑的骨架）

```ts
pi.on("session_start",          → 预热 startup context 缓存（按 session key）
pi.on("before_agent_start",     → 返回 { systemPrompt: 原 systemPrompt + Context Bundle + 行为指引 }
pi.on("agent_end",              → scheduleFlush（750ms 防抖增量同步）
pi.on("session_before_compact", → flush（压缩前抢救全量）
pi.on("session_compact",        → 重新预热 context（压缩后重注入准备）
pi.on("session_before_switch",  → flush + 清缓存
pi.on("session_shutdown",       → flush + 清缓存
```

## 3. 注入设计

- **落点是 system prompt**：`before_agent_start` 返回修改后的 `systemPrompt`（pi 每轮重建 system prompt，注入不在 transcript 里累积）。内容 = `## Nowledge Mem Context Bundle\n<正文>` + 固定行为指引段（何时 search/save、provenance、别重复读）。
- 读取回退链（与 claude-code hook 同构）：context(space) → context(default) → wm(space) → wm(default) → `~/ai-now/memory.md`（仅纯默认配置时）。
- 全链 8 秒总预算（多次尝试共享 deadline），单条消息 20k 字符截断；失败时降级为注入一行 `[startup context unavailable: <原因>]` + 指引段——**注入通道本身永不缺席**。
- 缓存按 session key，`session_start` 预热、`session_compact` 后刷新、切换/关闭时清除。

## 4. 抓取设计：连续增量同步，非「结束时一次性」

- 每次 `agent_end` 防抖 750ms 后，从 `ctx.sessionManager.getBranch()` **重读全量** session entries，规范化后整包发送；服务端负责去重。
- **服务端合同（关键发现）**：`POST /threads`（带 `thread_id`/`title`/`messages`/`source`/`project`/`metadata`）创建；409 视为已存在；`POST /threads/<id>/append` 带 **`deduplicate: true` + `idempotency_key`**（`source:sessionId:messageCount`）；404 时回退重建。thread id 确定性生成：`pi-<sessionId>`。
- **这条合同是通用 HTTP API，不依赖 nmem CLI 的 per-host importer**——直接修正 FACTS §1 的风险评估：「nmem 无 amp 导入源」不构成写路径阻塞，pi 官方连接器的活同步就没走 CLI importer。
- 无离线队列：靠「每次全量重发 + 服务端 dedupe」自愈（下次 flush 自动补上上次失败的）。in-flight 合并（同 thread 只保留最新 payload 排队）。
- 规范化细节：pi 特有 entry 类型都映射成 user/assistant 文本（bashExecution→user、compaction/branch summary→assistant）；**`role === "custom"`（extension 注入的上下文）明确排除**，注释原文：*"Extension-injected context is not user transcript. Keep it out of thread history."*——防止注入的记忆回流进 Mem。
- 只有「至少一条 user + 一条 assistant」才同步（防空 thread）。

## 5. 传输取舍：写走进程内 HTTP，读走 CLI 子进程

- Thread 同步 = 进程内原生 `fetch()`（30s 超时，`/remote-api` 路径自动降级尝试）。
- Startup context 读取 = spawn `nmem --json context/wm`（CLI 负责 space/agent 解析的一致性）。
- [观察] 同一个进程内插件，作者仍选择读路径走 CLI——换取与 nmem 客户端配置解析（space/identity/remote）的完全一致，代价是 spawn 延迟（有 8s 预算兜底）。abn 的「一切求 <30ms」在这里没有被采纳，说明官方对启动注入的延迟容忍度是「秒级预算 + 降级不缺席」，不是「毫秒级硬指标」。
- 配置解析：env（`NMEM_API_URL/KEY/SPACE/AGENT_ID/HOST_AGENT_ID`）→ `~/.nowledge-mem/config.json` → 默认 local。~100 行 Windows `.cmd` shim 引号/环境变量注入防御。

## 6. 一包服务两 host

`NMEM_PLUGIN_SOURCE_APP` / `NMEM_PLUGIN_HOST_LABEL` env 让同一包以 `pi` 或 `omp` 身份运行（OMP 内嵌 pi runtime）。[观察] 官方已有「近亲 host 复用同一 connector 包」的先例。

## 7. pi 事件面 vs Amp 事件面（对 Amp 设计最要命的一张表）

pi 的 extension API 事件远比 Amp 丰富（project_trust / resources_discover / session_* 全家 / before_agent_start / agent_settled / turn_* / message_* / tool_* / before_provider_* / model_select / user_bash / input …）。官方 pi connector 实际用到 7 个，其在 Amp（show-docs 实测面）的对应情况：

| pi connector 所用事件 | 用途 | Amp 对应物 | 差距 |
|---|---|---|---|
| `session_start` | 预热缓存 | `session.start` | ✓ 等价（都是 fire-and-forget，可做异步预热） |
| `before_agent_start` → 改 **systemPrompt** | 每轮注入 Context Bundle | `agent.start` → append 到**用户消息尾部** | **落点不同**：pi 注入进每轮重建的 system prompt（不进 transcript）；Amp 注入物拼进用户消息（**进 transcript、会累积**） |
| `agent_end` | 防抖增量同步 | `agent.end`（自带本轮 `messages`） | ✓ 有对应物，且 Amp 直接给增量；全量回读另有 `thread.messages()`（单次 ≤20 条需分页） |
| `session_before_compact` | 压缩前抢救 | **无** | Amp plugin API 无压缩事件 |
| `session_compact` | 压缩后重注入 | **无** | 同上 |
| `session_before_switch` | 终局 flush | **无直接对应** | Amp 事件均 thread-scoped，无切换语义 |
| `session_shutdown` | 终局 flush | 仅 `onDispose`（全插件 ~3s 预算、crash/SIGKILL 不执行、非 per-thread） | **可靠性弱一档** |

[观察] 缺口集中在「session 终局/压缩」一侧；pi 连接器对此的答案（连续增量同步 + 服务端幂等去重，把「结束时必须成功」弱化为「任一次 flush 成功即最终一致」）恰好**不依赖**终局事件的可靠性——这个模式在 Amp 的事件面上是完整可表达的。此为观察，不是设计结论。

## 8. 对既有材料的修正与增补

1. FACTS §1「nmem 无 amp transcript 导入源」→ 补充：活同步的官方路径是通用 HTTP 合同（§4），CLI importer 只服务历史回填（pi 包的 `scripts/sync-history.mjs` 干这个）。
2. abn 报告 §8 的差异表 → pi 案例填上了「与 Amp 同范式下官方怎么做」的空档：无离线队列、读走 CLI、写走 HTTP、连续同步、注入永不缺席（降级占位）。
3. 本机 Amp 已装的 `~/.agents/skills/` 5 件套与 pi 包捆绑的 skills 同源——Amp 侧「skills 通道」事实上已就位，connector 缺的是 extension 本体。
