# Amp 插件架构

Amp 将插件定义为一个由 Bun 执行的 TypeScript 程序。插件通过 `@ampcode/plugin` 提供的 `PluginAPI`，在代码中注册生命周期事件、工具、命令和 Agent。Amp 没有采用包含 manifest、hooks、rules 和 skills 子目录的插件包格式。

```text
<workspace-root>/
└── .amp/
    └── plugins/
        ├── nowledge-mem.ts      # 一个独立的 Amp 插件
        ├── permissions.ts       # 另一个独立插件
        └── architect-mode.ts    # 自定义 Agent Mode 插件
```

每个 `.ts` 文件本身就是一个插件入口：

```ts
import type { PluginAPI } from '@ampcode/plugin'

export default function (amp: PluginAPI) {
  amp.logger.log('Plugin initialized')

  // Lifecycle hooks
  amp.on('session.start', async (event, ctx) => {})
  amp.on('agent.start', async (event, ctx) => {})
  amp.on('tool.call', async (event, ctx) => {
    return { action: 'allow' }
  })
  amp.on('tool.result', async (event, ctx) => {})
  amp.on('agent.end', async (event, ctx) => {})

  // Agent-callable tool
  amp.registerTool({
    name: 'memory_search',
    description: 'Search stored memories',
    inputSchema: {
      type: 'object',
      properties: {
        query: { type: 'string' },
      },
      required: ['query'],
    },
    async execute(input, ctx) {
      return `Searching for: ${input.query}`
    },
  })

  // Command-palette command
  amp.registerCommand(
    'memory-status',
    {
      title: 'Show memory status',
      category: 'Nowledge Mem',
    },
    async (ctx) => {
      await ctx.ui.notify('Nowledge Mem is connected')
    },
  )

  // Cleanup on unload/reload
  amp.onDispose(async () => {
    amp.logger.log('Plugin disposing')
  })
}
```

## 插件加载位置

Amp 启动时会加载以下插件来源：

1. **项目级插件**

   ```text
   <workspace-root>/.amp/plugins/*.ts
   ```

   只在当前项目中生效。

2. **系统级插件**

   ```text
   ~/.config/amp/plugins/*.ts
   ```

   在当前用户自己的所有本机项目中生效。Windows 对应：

   ```text
   %USERPROFILE%\.config\amp\plugins\*.ts
   ```

3. **Workspace 全局插件**

   由 Amp Workspace 设置统一配置，对 Workspace 的所有成员生效。这项能力目前处于有限实验阶段，没有对应的普通本地扫描目录。

## 插件运行结构

```text
┌─────────────────────────────┐
│       Amp Plugin Host       │
│     Bun 长生命周期进程      │
└──────────────┬──────────────┘
               │ 加载 *.ts
               ▼
┌─────────────────────────────┐
│ default function(PluginAPI) │
└──────────────┬──────────────┘
               │
       ┌───────┼────────┬──────────┬───────────┐
       ▼       ▼        ▼          ▼           ▼
┌──────────┐ ┌─────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ Events   │ │Tools│ │Commands│ │ Agents │ │ UI/AI  │
│ amp.on() │ │ LLM │ │ Palette│ │ Modes  │ │ Config │
└──────────┘ └─────┘ └────────┘ └────────┘ └────────┘
```

## Plugin API 能力

| 能力 | Plugin API | 作用 |
|---|---|---|
| 生命周期事件 | `amp.on(...)` | 监听线程、Agent 和工具生命周期 |
| 自定义工具 | `amp.registerTool(...)` | 向模型暴露可调用工具 |
| 命令面板命令 | `amp.registerCommand(...)` | 添加用户主动执行的命令 |
| UI 交互 | `ctx.ui.*` | 通知、确认、文本输入和选项选择 |
| AI 调用 | `amp.ai.generate()` / `amp.ai.ask()` | 从插件中调用模型 |
| 自定义 Agent | `amp.createAgent(...)` | 创建专用 Agent 或子 Agent |
| Agent Mode | `amp.registerAgentMode(...)` | 在 Amp 模式选择器中添加自定义模式 |
| Thread 操作 | `amp.threads.get(...)` | 读取、控制或向其他 Thread 发送消息 |
| 配置管理 | `amp.configuration.*` | 读取和更新项目级或用户级配置 |
| Shell 执行 | `amp.$` / `ctx.$` | 使用 Bun Shell 执行命令 |
| 清理资源 | `amp.onDispose(...)` | 插件卸载、重载或正常关闭时清理资源 |
| Webhook | `amp.createWebhook(...)` | 接收与 Orb Thread 关联的持久 Webhook |

## 生命周期事件

```text
Plugin loaded
    │
    ├── default function(amp)       # 插件初始化
    │
    └── Thread session
          │
          ├── session.start         # Thread session 启动
          │
          └── 每轮用户消息
                ├── agent.start     # Agent 开始处理
                │
                ├── tool.call       # 工具执行之前
                ├── tool.result     # 工具产生结果之后
                └── agent.end       # Agent 本轮结束

Plugin reload / unload / shutdown
    └── onDispose                   # 释放外部资源
```

`tool.call` 可以决定工具调用如何继续：

```text
allow                  # 原样执行
modify                 # 修改输入后执行
reject-and-continue    # 拒绝该调用，但让 Agent 继续
synthesize             # 不执行工具，直接返回合成结果
error                  # 终止当前 Thread worker
```

## 与 Antigravity 架构的差异

```text
Antigravity                       Amp
──────────────────────────────    ──────────────────────────────
plugin.json manifest              不需要 manifest
mcp_config.json                   MCP 独立配置，不属于插件结构
hooks.json                        amp.on(...) 写在 TypeScript 中
rules/*.md                        AGENTS.md 是独立机制
skills/*                          Skills 是独立机制
目录表示一个插件包                每个顶层 *.ts 文件是一个插件
声明式 JSON/Markdown              编程式 TypeScript Plugin API
```

下面这种结构来自 Antigravity 的插件模型，不符合 Amp 官方的插件发现规则：

```text
.amp/plugins/nowledge-mem/
├── plugin.json
├── hooks.json
├── rules/
└── skills/
```

对应的 Amp 实现可以采用：

```text
<workspace-root>/
├── .amp/
│   └── plugins/
│       └── nowledge-mem.ts       # 工具、事件和命令
├── AGENTS.md                     # Always-on 项目规则，独立于插件
└── .agents/
    └── skills/
        ├── memory-search/
        │   └── SKILL.md
        └── thread-save/
            └── SKILL.md
```

`.agents/skills/` 与 `.amp/plugins/` 可以协同工作。两者由不同的机制发现和加载，不构成一个内置插件包。

## 加载与调试

修改插件后，在 Amp 命令面板中执行：

```text
plugins: reload
```

查看已经加载的插件及其事件、命令和工具：

```text
plugins: list
```

查看当前安装版本实际支持的 Plugin API：

```bash
amp plugins show-docs
```

查看自定义 Agent 可使用的模型和内置工具：

```bash
amp plugins show-agent-options
```

## 来源

- [Amp Plugins Guide](https://ampcode.com/manual#plugins)
- [Amp Plugin API Reference](https://ampcode.com/manual/plugin-api)
- 本机当前 Amp 版本导出的 `amp plugins show-docs`

Amp Plugin API 仍在快速演进。开发时应优先以当前安装版本的 `amp plugins show-docs` 为准。
