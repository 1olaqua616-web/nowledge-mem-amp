# Amp Connector 事实基底（2026-07-27）

> 阶段：前置对齐（Product Requirements 之前）。本文件只含事实与出处，不含架构结论、hook 选择、文件结构、实现计划。
>
> 证据分级：
> - **[实测]** 本机命令输出，2026-07-27 执行
> - **[文档]** 官方文档/仓库源码原文
> - **[二手]** WebFetch 小模型摘要，引用前需回源核对

## 1. 缺失确认：Amp 不在 Nowledge 官方连接矩阵中

- **[实测]** `https://mem.nowledge.co/SKILL.md` 全文 grep `\bamp\b` 零命中。Host 检测表覆盖 Claude Code / Grok Build / Codex / Cursor / Gemini CLI / Copilot CLI / OpenClaw / Hermes / Droid / Alma / Bub / Pi / OpenCode / Kimi Code / Kimi Work / ZCode / MiMo Code / OMP / Claude Desktop / Proma / Raft / Lody / Multica / Cumora / Paseo + Generic MCP client，无 Amp。
- **[实测]** `https://mem.nowledge.co/docs/integrations` 页面正文 grep 零命中。
- **[实测]** `nmem t sync --from amp` → `Invalid Client: 'amp' is not supported. Must be one of: claude-code, codebuddy, workbuddy, codex, cursor, grok, gemini-cli, kimi-code, kimi-work, mimo-code, omp, opencode, pi, craft-agent, hermes, paseo`
- **[实测]** `nmem config mcp show --host amp` → `error: Unsupported MCP host: amp`
- **[实测]** `nmem --json context --source-app amp` → **正常返回完整 context bundle**（owner/agent/space/rules/working memory）。即 nmem 的「读路径」对 amp 已可用，「transcript 导入路径」缺失。
- **[文档]** Hermes 安装说明中出现通用导入端点线索：setup 脚本应打印 `Thread import endpoint: /threads/import`。

## 2. 本机 Amp 现状

- **[实测]** `amp --version`：`0.0.1785111528-g7dd942`（2026-07-27 发布，检查时 4h 前）。
- **[实测]** `amp mcp doctor`：`nowledge-mem (user settings): connected (56 tools: memory_search, memory_add, thread_search, read_working_memory, read_context_bundle, mem_fs, find_skills, ...)`，配置在 `~/.config/amp/settings.json`，endpoint `http://127.0.0.1:14242/mcp/`。
- **[实测]** `amp skill list`（26 个）中 Nowledge 相关为 generic skills（`~/.agents/skills/`）：`check-integration`、`distill-memory`、`read-working-memory`、`search-memory`、`status`、`save-handoff`、`save-thread`。
- **[实测]** `save-thread` skill 自述：*"Deprecated compatibility skill. In generic npx skills environments this must degrade honestly to a resumable handoff"* —— 即 Amp 下今天没有真实 transcript capture，只有 handoff 摘要。
- **[实测]** `nmem status`：0.10.30，local mode，ok。
- 对照 Nowledge 官方 Autonomy Ladder（§3）：Amp 现状 = **Reusable package + Direct MCP 两层**，无 native connector 层能力。

## 3. Nowledge 官方口径：native connector 的定义与取舍

- **[文档]** Autonomy Ladder（docs/integrations 原文表格）：
  - **Native connector**：startup context *"Often automatic at session start or through host lifecycle"*；recall/distill *"Strongest path; may be hook-driven"*；threads *"Some hosts also add real automatic capture or real transcript save"*。
  - **Reusable package**：*"Usually guided"* / *"Guided by rules, skills, or prompts"* / *"Usually handoff-only or explicit-only"*。
  - **Direct MCP**：*"Guided only; MCP tools alone do not create autonomy"* / *"no local transcript import"*。
- **[文档]** 官方决策树：*"install the most specific Nowledge path your host supports. Only fall back to generic MCP when there is no better package."*
- **[文档]** Thread sync 三路径（desktop auto-sync / connector hooks / historical import），边界是 transcript 所在机器：*"If Mem runs on another machine, that server cannot scan your laptop's ~/.codex, ~/.claude ..."*——connector hook 是「Keeping new work current」的官方路径。
- **[文档]** SKILL.md 安装契约的验收（Definition of Done 六条）：host detected / install path / nmem status green / restart confirmed / Context Bundle 或 Working Memory 在新会话内可达 / 提示保存第一条记忆。

## 4. 参考实现事实（两个官方 plugin，源码已读）

### claude-code plugin（`nowledge-co/community/nowledge-mem-claude-code-plugin`）

- **[文档]** hooks.json 四类生命周期：
  - `SessionStart`（matcher `startup|resume|clear`）→ `nmem-hook-read.sh` 注入 Context Bundle；matcher `compact` → 重注入 + 提醒保存洞察；
  - `UserPromptSubmit` → 每个 prompt 注入一段 nmem 主动检索/保存/技能路由提醒；
  - `PreCompact` / `Stop` → `nmem-hook-save.py` 保存真实 transcript（timeout 35s）。
- **[文档]** `nmem-hook-read.sh` 回退链：context（带 space）→ context（默认）→ wm read（带 space）→ wm read（默认）→ `cat ~/ai-now/memory.md`。Space 只认显式 `NMEM_SPACE`，注释明确拒绝从 cwd/git 推断（历史教训：repo 名污染 space）。
- **[实测]** 本 Claude Code 会话即活样本：SessionStart 注入了 Context Bundle，UserPromptSubmit 注入了路由提醒。

### codex plugin（`nowledge-mem-codex-plugin`）

- **[文档]** 同样三钩子（SessionStart+compact 合并 matcher / UserPromptSubmit / Stop），另有 `install_hooks.py` 处理 host 版本差异与 feature gate；捆绑本地 MCP `.mcp.json`；`NMEM_AGENT_ID` / `NMEM_HOST_AGENT_ID` / `NMEM_SPACE` 经 env 映射为 Mem 的 AI Identity 与默认 Space。
- **[文档]** README 定位为 hybrid：*"plugin package for automatic ... startup context ... + bundled MCP for stronger retrieval ... + lifecycle hooks ... + project AGENTS.md for repo-specific follow-through"*。

## 5. Amp plugin 能力面（官方 manual + plugin-api）

- **[文档]** 分工原文：*"Use `AGENTS.md` for durable instructions, skills for task-specific agent guidance, plugins for custom tools or event-driven behavior, and MCP only when integrating an MCP server."*
- **[文档]** 形态：TypeScript 文件，Bun 执行；三层加载：project `.amp/plugins/*.ts`、user `~/.config/amp/plugins/*.ts`、workspace global（experimental）；改动后 `plugins: reload`。
- **[文档]** 事件与能力：
  - `session.start`（thread 建立时，返回 void）；
  - `agent.start`（**每次用户提交 prompt 触发**，可返回 `{ message: { content, display } }` 注入消息）；
  - `agent.end`（本轮结束，事件带 `messages`，可返回 `{ action: 'continue', userMessage }`）；
  - `tool.call` / `tool.result`（可 allow/reject/modify/synthesize）；
  - `ctx.thread.messages(options)` 读全 transcript，`append`、`waitForResponse`；
  - `amp.registerTool` / `amp.registerCommand`（command 可 setAvailability）；
  - `amp.createWebhook`（durable、at-least-once）；`amp.configuration.get/update/delete`；`amp.ai.ask/generate`；`amp.createAgent` / `registerAgentMode`。
  - **无 MCP 注册 API**（MCP 走 settings.json，与 plugin 并行，本机已配好）。
- **[文档]** Orbs：*"Orbs are remote machines where Amp agents can run without supervision"*；配置为 **project 级**（`.agents/setup` / `.agents/resume` 提交入库；env/secrets 在 project settings；orb size per project）。文档未提供 per-orb 差异化配置。
- **[实测·间接]** builtin skill `creating-webhooks` 自述 *"Creates durable webhook handlers in Amp plugins running in Orbs"* → plugin 在 Orb 内运行有官方措辞支撑，待实测确认。

## 6. ABN 文章要点 **[二手，回源核对后引用]**

- 三启动通道 + context economics：① PreInvocation hook 只注入当日 briefing + `nowledgemem://memory/<id>` 直链；② always-on rules 文件承载常驻行为边界；③ skills 索引只放轻指针，正文按需加载。理由：*"inlining an entire knowledge graph into system prompts causes severe context bloat and expensive KV cache penalties."*
- 原生 plugin 的必要性论证走性能/韧性：CLI 子进程 hook 300–500ms vs 直连 REST <30ms（*"ensuring session initialization never stalls developer flow"*），三级传输回退（native HTTP → CLI → 离线队列）。
- 注入时机原则：启动只给最小 briefing；skill 正文按需 ephemeral 注入（zero-restart）；workspace 级与 global 级 plugin 双扫描 + MCP 配置按会话动态解析 → bundle 可按 workspace/agent 切换。

## 7. 痛点 ↔ 事实对照（含最小推论，均已标注；非方向结论）

> **[废弃 2026-07-28]** 本节以已废弃的 Goals 原文为组织轴（Goals/Rules 与 D1–D5 同源，一并废弃）。表内 [文档]/[实测] 事实条目本身仍真，存活登记见 §5 与 §7.5；本节不再作为任何步骤的输入。

| 用户痛点（Goals 原文） | 相关事实 | 推论标注 |
|---|---|---|
| AGENTS.md「无法在 session 内更换」bundle | `agent.start` 每 prompt 触发且可注入消息 [文档]；`registerCommand` + `configuration` 可持有运行时状态 [文档] | 推论：session 内换 bundle 在 Amp plugin 能力面内可表达 |
| convention「触发依赖主动更多」 | claude-code plugin 用 UserPromptSubmit 每 prompt 确定性注入路由提醒 [文档+实测活样本]；Amp 对应事件为 `agent.start` [文档] | 推论：确定性触发在 Amp 能力面内可表达 |
| 按 customer agent / orb 更换（orb 待检查） | Orb 配置是 project 级，无 per-orb 差异化配置 [文档]；plugin 在 Orb 内运行有间接证据 [实测·间接] | 事实修正：「按 orb 换」在 Amp 当前模型下不存在 per-orb 配置位，需改述（见对齐问题） |
| （未在 Goals 中出现）transcript capture | Amp 下 save-thread 已声明降级 handoff-only [实测]；`ctx.thread.messages()` 可读全 transcript [文档]；nmem 无 amp 导入源 [实测]；存在 `/threads/import` 通用端点线索 [文档] | 范围问题，留给 product review 拍板 |

## 7.5 增补（2026-07-28）：本机 `amp plugins show-docs` 实测，升级 §5 若干条目为 [实测]

- **[实测]** 注入通道确认：`agent.start` 是 request 事件，返回 `AgentStartResult.message?: { content: string; display?: boolean }`。原文语义：*"A message to append after the user's content **in the user message**."* `display` 默认 false（默认不在 UI 显示）。即注入物是**拼在用户消息尾部**，不是独立消息，也不是 system 级。
- **[实测]** `session.start` 不在 `PluginRequestResultMap` 中（request 事件只有 tool.call / tool.result / agent.start / agent.end）→ session.start 是 fire-and-forget，**无注入能力**；Amp 的「启动注入」必然搭线程首个 agent.start。
- **[实测]** transcript 两条路径：① `agent.end` 事件自带 `messages: ThreadMessage[]`（*"All messages since the agent.start event"*，即逐轮增量）；② `thread.messages(options)` 默认 `{from:'end', limit:10}`，**单次上限 20 条**，需 offset 分页。
- **[实测]** `onDispose`：所有清理回调合计约 **3 秒预算**；**crash / SIGKILL 时不执行**，官方建议外部兜底（server-side idle timeout）。
- **[实测]** 插件进程模型原文：*"long-lived processes that may run for multiple threads concurrently"*——一个常驻进程服务多线程，事件全部 thread-scoped。
- **[实测]** 其余能力面确认：`createAgent`/`getBuiltinAgent`/`registerAgentMode`/`activeThread`/`threads.get` 已一等化（experimental 下保留兼容别名）；另有 `appendUserMessage(…, {steer:true})` 转向注入、`createStatusItem`（experimental）、`amp.attachments`、`helpers.toolCallsInMessages` 等。`createWebhook` 标注 @experimental，绑定 Orb thread。
- **[实测]** API 出处自述：*"Amp's plugin API is inspired by pi's extension API"*（pi 是 nmem 已有 native connector 的 host 之一，见 SKILL.md 安装矩阵）。
- **[源码·2026-07-28]** §1「nmem 无 amp transcript 导入源」的风险评估修正：官方 `nowledge-mem-pi` 连接器的活同步不走 CLI importer，走通用 HTTP 合同（`POST /threads` + `POST /threads/<id>/append` 带 `deduplicate: true` + `idempotency_key`），`source` 为任意字符串。写路径对 amp 无服务端阻塞；CLI importer 只影响历史回填。详见 `PI-CONNECTOR-CODE-READING-2026-07-28.md` §4。

## 8. 命令留痕

```bash
amp --version                          # 0.0.1785111528-g7dd942
amp skill list                         # 26 skills，Nowledge generic 7 个
amp mcp list && amp mcp doctor         # nowledge-mem connected, 56 tools
nmem --json status                     # 0.10.30 local ok
nmem --json context --source-app amp   # 返回完整 bundle（读路径可用）
nmem t sync --from amp --limit 1       # Invalid Client: 'amp' is not supported
nmem config mcp show --host amp        # Unsupported MCP host: amp
amp plugins show-docs                  # 2026-07-28 增补：完整 @ampcode/plugin 类型声明（§7.5 依据）
```
