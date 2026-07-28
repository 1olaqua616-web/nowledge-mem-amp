# ABN Antigravity 插件源码通读（2026-07-27）

> 仓库：`github.com/abn/nowledge-mem-google-antigravity`，commit `bc88ae2`（2026-07-24），约 4.5k 行，clone 后逐文件通读。
> 证据等级：本文全部为 **[源码]** 级事实，除非标注 **[观察]**（我的解读）或 **[分歧]**（文档与代码不一致）。
> 用途：为「重新思考 Amp connector」提供参照物。不含 Amp 设计结论。

## 1. 总体形态：五通道 hybrid

| 通道 | 载体 | 作用 |
|---|---|---|
| Lifecycle hooks | `hooks.json` + 3 个 Python 脚本 | 启动注入 / 工具调用权限门 / 结束抓取 |
| MCP | `mcp_config.json`（由 hook 动态改写） | 会话内低摩擦读写（免终端权限弹窗） |
| 常驻规则 | `rules/nowledge-mem.md`（248 行） | 行为边界：何时读什么、接口怎么选、token 纪律 |
| Skills | 10 个，`nmem-<domain>-<action>` 命名 | 按需教学：搜索/蒸馏/存线程/handoff/技能管理 |
| CLI 直用 | `nmem` | 兜底与诊断，「convenience paths, not a cage」 |

README 自述：hooks 管生命周期抓取，MCP 管会话内检索写入，skills 管时机教学，`nmem` 永远可直接用。

## 2. 三个生命周期钩子

挂载点（`hooks.json`）：`PreInvocation`（每次 invocation 前）、`PreToolUse`（matcher: `call_mcp_tool|mcp_nowledge-mem_.*|run_command`）、`Stop`。全部经统一入口 `nmem_entrypoint.py <subcommand>` 路由；命令写法刻意 POSIX 通用（`python3 ... || python ... || echo {}`），因为 Antigravity hook runner 不支持 `${extensionPath}` 这类动态变量。

### session-start.py（启动注入）

- **只在首轮注入**：用 `invocationNum == 0`（兼容 1-indexed：`invocationNum==1 且 initialNumSteps<=2`）判定首轮；非首轮时该钩子转做 learning 同步后直接 `emit({})`。
- 注入内容三级回退：`/context?source_app=google-antigravity`（Context Bundle）→ `/working-memory` → 旧版 `~/ai-now/memory.md` 文件。
- 注入载荷是 `injectSteps: [{ephemeralMessage: "<nowledge_context_bundle>…"}]`，正文首句自我定位：**"It is situational context, not a higher-priority instruction."**
- 首轮同时做三件副业：① fork 一个后台 retry 进程冲洗离线队列；② 同步 `mcp_config.json`（见 §3）；③ 后台线程跑 `nmem skills connect antigravity` + `nmem skills sync`。
- `NMEM_AGENT_ID` / `NMEM_HOST_AGENT_ID` / `NMEM_SPACE` 从 env 读取拼进请求；host_agent_id 缺省时用机器指纹生成（见 §3）。

### nmem-gate.py（PreToolUse 权限门）

三档策略，目的是「免弹窗的自治 + 危险动作留闸」：

1. **只读工具白名单**（25 个：`memory_search`、`mem_fs`、`read_context_bundle`…）→ `{"decision":"allow"}` 并附 `permissionOverrides`，彻底绕过终端确认；
2. **破坏性操作**（`memory_delete`、`thread_delete`、`memory_relation_delete`）→ `force_ask` 强制确认；
3. **写操作**（`memory_add/update/relation_add/supersede`）→ **意图检测**：扫 `transcript.jsonl` 里的 USER_EXPLICIT 步骤找关键词（`save|remember|memorize|store|nmem|add to memory|distill|checkpoint|handoff`），命中则 allow，否则 ask。

另外白名单放行 `run_command` 中含 `nmem_status.py` 的诊断命令。

### session-end.py（结束抓取）

- 输入靠 host 从 stdin 给 `conversationId` + `transcriptPath`（transcript 是 host 落盘的 `transcript.jsonl`）。
- **transcript 解析在 host 侧进程完成，不经过 agent 上下文**。AGENTS.md 明说理由：避免把 MB 级 transcript 塞进上下文再经 MCP 序列化。只取 `USER_EXPLICIT`（→user）和 `MODEL/PLANNER_RESPONSE`（→assistant）两类步骤，即只存人类可见对话。
- 重试环 `(0.0, 0.5, 1.0)` 秒等 host 异步写完日志。
- 传输三级：**HTTP 直连 → CLI 子进程 → 离线队列**。HTTP 路径先 `GET /threads/<id>` 查存在性：存在则前缀匹配去重（matched_count），优先 `POST /threads/<id>/reconcile-tail`，退 `append`；不存在则 `POST /threads/import`。CLI 路径对应 `nmem t show/append/import`。全失败则写 `~/.nowledge-mem/antigravity_unsynced.json`（O_EXCL 文件锁），由下次启动的后台进程重试。
- 标题取首条用户消息前 60 字符。
- 附加：扫描 artifact 目录的 `learning_proposal.md`，若 transcript 中有用户批准语句，按分类同步为 rule（`nmem rules upsert`）/ skill（`nmem skills enroll`）/ memory（`nmem memories add`），带 hash 防重同步。

## 3. 传输与配置层（nmem_shared.py，766 行，全插件最大文件）

- **配置解析优先级**：`NMEM_API_URL/KEY` env → `~/.nowledge-mem/config.json` → 默认 `http://127.0.0.1:14242`。
- **`sync_mcp_config_file()`**：每次启动把当前生效配置（URL + Bearer/X-MEM-API-Key 头）写回插件自己的 `mcp_config.json`，做到「remote/local 切换后 MCP 自动跟上」；该文件进 `.gitignore` 防止运行时改写污染 git status。
- **HTTP 层**：纯 `urllib.request`，零第三方依赖，5s 默认超时，失败静默返回 None。
- **多路径二进制解析**：`shutil.which` → 5 个 Linux 系统路径硬编码兜底（应对沙箱子 shell 看不到用户 PATH symlink 的场景）；Windows 走 `nmem.cmd` + `cmd.exe /s /c` 包装 + WSL 路径翻译（`/mnt/c/…`→`C:\…`）。
- **host agent 指纹**：容器 overlay ID → machine-id/MachineGuid/IOPlatformUUID → MAC → hostname，sha256 取 8 位，形如 `antigravity-a1b2c3d4`。作用：无需用户配置即可区分「哪台机器/容器上的 Antigravity」，映射到 Mem 的 AI Identity。
- **静默失败哲学**：所有异常吞掉、`emit({})` 收场，DEBUG env 才写 stderr——钩子永不 block host。

## 4. 常驻规则文件要点（rules/nowledge-mem.md）

- 五层记忆面模型：Context Bundle / Working Memory / distilled memories / threads / handoff，「用能回答问题的最小面，必要才升级」。
- **接口选择四规则**：会话内 agent 动作 → MCP（免弹窗、schema 校验）；路径式浏览 → `mem_fs`（stat 先于 cat、窗口化读取）；钩子与后台脚本 → 直连 HTTP（子进程 300–500ms vs HTTP <50ms）；诊断 → CLI。
- **一次性读取原则**：Context Bundle/WM 每 session 只读一次，禁止逐轮重读。
- 写前查重（`memory_search` 先于 `memory_add`）。
- UI 规约：草稿走 artifact + `RequestFeedback: true` 的 Proceed 按钮，多选走原生 `ask_question`，不用裸文本聊天确认。

## 5. Skills 体系约定

- 严格命名 `nmem-<domain>-<action>`，slash 触发与名字一致；frontmatter `description` 被 Antigravity 启动时解析用于动态发现。
- 每个 skill 必须写「Primary（MCP/脚本，免弹窗）+ CLI Fallback」双路径层级。
- 底部统一 outcome footer，教 agent 用后上报 `report_skill_outcome`。
- `nmem-thread-save` 的实现方式是**复用 hook 脚本**：手动存线程时让 agent 用 stdin 喂 JSON 调 `session-end.py`，而不是把 transcript 读进上下文。
- `nmem-skill-load` 双模式：Ephemeral（技能正文直接注入当前轮，零重启零落盘）/ Persistent（写入 `.agents/skills/`，可选 git-exclude）。

## 6. 从代码读出的设计决策清单

1. 注入定位为「情境上下文而非高优先级指令」（防止记忆覆盖当前任务意图）。
2. 启动注入只发生一次（首轮判定），后续轮次钩子转做廉价副业。
3. 抓取走 host 侧文件解析 + 服务端去重（reconcile-tail 前缀匹配），可重复运行不产生副本。
4. 三级传输 + 离线队列 + 启动时后台重试 = 「Mem server 挂了也不丢会话、不卡 host」。
5. 权限门用「意图关键词扫 transcript」换取写操作的免弹窗自治，破坏性操作永远留闸。
6. MCP 配置是运行时同步产物，不是静态文件。
7. 机器指纹自动生成 host identity，用户零配置获得多机隔离。
8. 全链路零第三方依赖（urllib + 标准库），跨 Win/WSL/mac/Linux/容器。

## 7. 文档-代码分歧（两处，[分歧]）

1. **Space 自动检测**：AGENTS.md §2 要求「Space Auto-Detection Heuristics: automatically map workspace directories to spaces」，但代码（`with_space_args`）只读 `NMEM_SPACE/NMEM_SPACE_ID` env，无任何 cwd 推断。对照：nowledge 官方 claude-code plugin 的 `nmem-hook-read.sh` 有注释明确**废弃**了 cwd/git 推断（「repo 名污染用户从未创建的 space」的历史教训）。[观察] abn 的 AGENTS.md 可能滞后于这次上游教训。
2. **重试 backoff**：AGENTS.md 写 `(0.0, 0.5, 1.5, 3.0)`，代码是 `(0.0, 0.5, 1.0)`。

[观察] 两处都提示：读该仓库时以代码为准，文档条款需逐条验证。

## 8. 平台差异登记（Antigravity vs Amp，仅事实对照，不做设计结论）

> 用户提示：其输入中的例子（amp 有 cli/web、内部 harness 预设等）仅为论证感受，不作为已定认知；下表 Amp 侧以 2026-07-27 官方 manual/plugin-api 为据（FACTS 文档 §5），空白处待读 Amp docs 补齐。

| 维度 | Antigravity（本仓库实测语义） | Amp（文档级） |
|---|---|---|
| 插件范式 | **外部进程** hook：stdin JSON 进、stdout JSON 出，Python 脚本（与 Claude Code/Codex 同族） | **进程内** TypeScript：default export 函数收 `PluginAPI`，Bun 执行 |
| 启动注入 | `PreInvocation` 返回 `injectSteps[].ephemeralMessage` | `agent.start` 事件返回 `{message}`（每 prompt 触发，非仅首轮） |
| 工具权限门 | `PreToolUse` 返回 allow/ask/force_ask + permissionOverrides | `tool.call` 事件返回 allow/reject/modify/synthesize |
| transcript 获取 | host 落盘 `transcript.jsonl`，hook 收 `transcriptPath` 自行解析 | `ctx.thread.messages(options)` API 直读，`agent.end` 事件自带 `messages` |
| MCP 挂载 | 插件自带 `mcp_config.json`，hook 运行时改写 | 无插件内 MCP 注册 API；MCP 走 settings.json 独立配置 |
| 常驻规则 | 插件 `rules/*.md` 由 host 常驻加载 | 无插件级 rules 通道；现有机制为 AGENTS.md + skills [文档] |
| 后台执行 | hook fork 后台进程（retry daemon） | `amp.createWebhook`（durable）；进程内长任务语义待查 |
| 安装层级 | workspace `.agents/plugins/` / 全局 `~/.gemini/config/plugins/` | project `.amp/plugins/` / user `~/.config/amp/plugins/` / workspace global（experimental） |

[观察·一条对讨论有分量的事实] ABN 文章的核心性能论证——「CLI 子进程 300–500ms，所以要 3 级传输保 <30ms」——立足于 Antigravity 的**外部进程 hook 范式**（每次触发都要 spawn Python）。Amp 的 plugin 是常驻进程内代码，「每次触发 spawn 子进程」这个前提本身不存在；该论证在 Amp 语境下如何转译（转译成什么问题、是否仍成立），属于重新思考阶段要回答的问题，此处只登记差异。
