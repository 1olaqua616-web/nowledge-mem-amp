# Amp Connector Design Agent Context

> 本文件只路由原始目标、规则和范围。Design agent 必须读取列出的原文，不以本文件替代原文；不要继承此前 thread 中生成的 connector 设计或必要性结论。

## Goals

用户原文：

> 如果我的理解没错，我倾向于为 amp 设计并生成一个 connector 来解决缺失问题。我倾向于在此对话中明确我们的步骤（在具体执行前）和上下文。

> 我想让你看 abn 的文章是因为我以为 antigravity 原本是没有 plugins 的，abn 写了这个关于 plugins 的文章。我期待你从中找的设计思路。当然这可能和 claude，codex 的 plugins 想当。

> 我认为这个提前注入相比于 agents.md 更重要是提前汇入的 bundle 我能随时更换，根据 customer agent 或者orb 更换（orb 是否能换待检查）。

> 我还是建议去写 plugins，因为我不满意目前的 agents.md，一是无法随时更换，应该说无法在 session 内更换。二是我在其中为了设置哦 convention，但其触发依赖主动更多。

## Rules

用户对当前 thread 与 design agent 的分工原文：

> 你的任务不是设计生成思路。我们只是讨论，并得出一个预处理过的上下文交给设计 agent。

> 我不需要区分，我只需要把原文给他，从 goals，rules，scope 处理，不要自行区分。

用户指定的前置开发方式原文来自 `Why Software Factories Fail: Turning the lights back on`：

> We're gonna embrace that same thing we've been doing since before AI, which is to do a little bit of planning up front, to reduce the odds of a long and difficult review.

> We're gonna find leverage, and we're gonna use AI to help with this, across 4 phases:
>
> - Product Requirements
> - System Architecture
> - Program Design
> - Vertical Slices

> Everything starts with a product review: a short doc that pins down what we're building and why.

> First, we align on the problem to solve -- the actual user pain, in the user's terms. Second, what success looks like -- what can we read after shipping to decide the thing was worth building.

> We try to keep this pretty grounded in the product space, not the technical.

> But what I see working well is that before anyone (human or agent) writes the implementation, we go a level down from architecture into the shape of code: the types, the method signatures, the program layout, and the call stacks.

> Checking 100-200 lines and resteering is a lot cheaper here.

## Scope

### 用户指定的核心范围

用户原文：

> 当然最开始也是最重要的，我们真的需要一个原生 connector 吗。或许这个问题藏在 abn 的文章里。

> 我认为这个提前注入相比于 agents.md 更重要是提前汇入的 bundle 我能随时更换，根据 customer agent 或者orb 更换（orb 是否能换待检查）。

> 一是无法随时更换，应该说无法在 session 内更换。二是我在其中为了设置哦 convention，但其触发依赖主动更多。

### 必须读取的原文

1. ABN，`Bringing your (k)nowledge to Antigravity`：  
   <https://abn.is/void/bringing-your-k-nowledge-to-antigravity/#a-the-3-startup-channels-context-economics-rationale>
2. Dex Horthy，`Why Software Factories Fail: Turning the lights back on` 本地原文：  
   [`Clippings/Why Software Factories Fail Turning the lights back on.md`](../../../Clippings/Why%20Software%20Factories%20Fail%20Turning%20the%20lights%20back%20on.md)
3. Dex Horthy 原始来源：  
   <https://x.com/dexhorthy/status/2081058573556306030?s=20>
4. Nowledge Mem 安装契约：  
   <https://mem.nowledge.co/SKILL.md>
5. Nowledge Mem integrations：  
   <https://mem.nowledge.co/docs/integrations>
6. Amp Plugins Manual：  
   <https://ampcode.com/manual#plugins>
7. Amp Plugin API：  
   <https://ampcode.com/manual/plugin-api>
8. Nowledge Mem Claude Code plugin 原文：  
   <https://github.com/nowledge-co/community/tree/main/nowledge-mem-claude-code-plugin>
9. Nowledge Mem Codex plugin 原文：  
   <https://github.com/nowledge-co/community/tree/main/nowledge-mem-codex-plugin>

### 当前环境入口

- Amp 当前全局 skills：运行 `amp skill list` 检查。
- Amp 当前 MCP：运行 `amp mcp list` 与 `amp mcp doctor` 检查。
- Nowledge Mem：运行 `nmem --json status`、`nmem --json context --source-app amp` 检查。
- Amp 当前版本：运行 `amp --version` 检查。
- Amp 的 Orb、runner、local executor 能力必须从当前官方文档与实测检查，不从本文件预设。

### 本 context 不包含

- connector 架构结论；
- plugin 文件结构；
- lifecycle hook 选择；
- transcript capture 取舍；
- session 内切换实现；
- Orb 配置方案；
- 实现计划或代码。

这些内容由 design agent 在读取原文、与用户完成前置对齐后产生。
