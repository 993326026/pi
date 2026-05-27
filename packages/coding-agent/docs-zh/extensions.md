> pi 可以创建扩展。让它为你的用例构建一个吧。

# 扩展

扩展是扩展 pi 行为的 TypeScript 模块。它们可以订阅生命周期事件、注册 LLM 可调用的自定义工具、添加命令，等等。

> **/reload 放置位置：** 将扩展放在 `~/.pi/agent/extensions/`（全局）或 `.pi/extensions/`（项目本地）以实现自动发现。仅在快速测试时使用 `pi -e ./path.ts`。自动发现位置中的扩展可以通过 `/reload` 热重载。

**主要功能：**
- **自定义工具** - 通过 `pi.registerTool()` 注册 LLM 可调用的工具
- **事件拦截** - 阻止或修改工具调用、注入上下文、自定义压缩
- **用户交互** - 通过 `ctx.ui` 提示用户（选择、确认、输入、通知）
- **自定义 UI 组件** - 通过 `ctx.ui.custom()` 实现带键盘输入的完整 TUI 组件，用于复杂交互
- **自定义命令** - 通过 `pi.registerCommand()` 注册如 `/mycommand` 的命令
- **会话持久化** - 通过 `pi.appendEntry()` 存储可在重启后保留的状态
- **自定义渲染** - 控制工具调用/结果和消息在 TUI 中的显示方式

**示例用例：**
- 权限门控（在执行 `rm -rf`、`sudo` 等前确认）
- Git 检查点（在每轮保存，分支时恢复）
- 路径保护（阻止写入 `.env`、`node_modules/`）
- 自定义压缩（按你的方式总结对话）
- 对话摘要（参见 `summarize.ts` 示例）
- 交互式工具（问题、向导、自定义对话框）
- 有状态工具（待办列表、连接池）
- 外部集成（文件监视器、webhook、CI 触发器）
- 等待时的游戏（参见 `snake.ts` 示例）

参见 [examples/extensions/](../examples/extensions/) 查看可运行的实现。

## 目录

- [快速开始](#快速开始)
- [扩展位置](#扩展位置)
- [可用导入](#可用导入)
- [编写扩展](#编写扩展)
  - [扩展样式](#扩展样式)
- [事件](#事件)
  - [生命周期概览](#生命周期概览)
  - [资源事件](#资源事件)
  - [会话事件](#会话事件)
  - [代理事件](#代理事件)
  - [模型事件](#模型事件)
  - [工具事件](#工具事件)
- [ExtensionContext](#extensioncontext)
- [ExtensionCommandContext](#extensioncommandcontext)
- [ExtensionAPI 方法](#extensionapi-方法)
- [状态管理](#状态管理)
- [自定义工具](#自定义工具)
- [自定义 UI](#自定义-ui)
- [错误处理](#错误处理)
- [模式行为](#模式行为)
- [示例参考](#示例参考)

## 快速开始

创建 `~/.pi/agent/extensions/my-extension.ts`：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  // 响应事件
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("扩展已加载！", "info");
  });

  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("危险！", "允许执行 rm -rf 吗？");
      if (!ok) return { block: true, reason: "被用户阻止" };
    }
  });

  // 注册自定义工具
  pi.registerTool({
    name: "greet",
    label: "问候",
    description: "按名字问候某人",
    parameters: Type.Object({
      name: Type.String({ description: "要问候的名字" }),
    }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      return {
        content: [{ type: "text", text: `你好，${params.name}！` }],
        details: {},
      };
    },
  });

  // 注册命令
  pi.registerCommand("hello", {
    description: "问好",
    handler: async (args, ctx) => {
      ctx.ui.notify(`你好 ${args || "世界"}！`, "info");
    },
  });
}
```

使用 `--extension`（或 `-e`）标志测试：

```bash
pi -e ./my-extension.ts
```

## 扩展位置

> **安全：** 扩展以你的完整系统权限运行，可以执行任意代码。仅安装来自可信来源的扩展。

扩展从以下位置自动发现：

| 位置 | 范围 |
|------|------|
| `~/.pi/agent/extensions/*.ts` | 全局（所有项目） |
| `~/.pi/agent/extensions/*/index.ts` | 全局（子目录） |
| `.pi/extensions/*.ts` | 项目本地 |
| `.pi/extensions/*/index.ts` | 项目本地（子目录） |

通过 `settings.json` 添加额外路径：

```json
{
  "packages": [
    "npm:@foo/bar@1.0.0",
    "git:github.com/user/repo@v1"
  ],
  "extensions": [
    "/path/to/local/extension.ts",
    "/path/to/local/extension/dir"
  ]
}
```

要通过 npm 或 git 作为 pi 包共享扩展，请参见 [packages.md](packages.md)。

## 可用导入

| 包 | 用途 |
|-----|------|
| `@earendil-works/pi-coding-agent` | 扩展类型（`ExtensionAPI`、`ExtensionContext`、事件） |
| `typebox` | 工具参数的模式定义 |
| `@earendil-works/pi-ai` | AI 工具（Google 兼容枚举的 `StringEnum`） |
| `@earendil-works/pi-tui` | 自定义渲染的 TUI 组件 |

npm 依赖也可以使用。在扩展旁边添加 `package.json`（或在父目录中），运行 `npm install`，`node_modules/` 中的导入会自动解析。

对于通过 `pi install` 安装的分布式 pi 包（npm 或 git），运行时依赖必须在 `dependencies` 中。包安装默认使用生产环境安装（`npm install --omit=dev`），因此 `devDependencies` 在运行时不可用；当配置了 `npmCommand` 时，git 包使用纯 `install` 以与包装器兼容。

Node.js 内置模块（`node:fs`、`node:path` 等）也可用。

## 编写扩展

扩展导出一个默认工厂函数，接收 `ExtensionAPI`。工厂可以是同步或异步的：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  // 订阅事件
  pi.on("event_name", async (event, ctx) => {
    // ctx.ui 用于用户交互
    const ok = await ctx.ui.confirm("标题", "确定吗？");
    ctx.ui.notify("完成！", "info");
    ctx.ui.setStatus("my-ext", "处理中...");  // 页脚状态
    ctx.ui.setWidget("my-ext", ["第 1 行", "第 2 行"]);  // 编辑器上方的小部件（默认）
  });

  // 注册工具、命令、快捷键、标志
  pi.registerTool({ ... });
  pi.registerCommand("name", { ... });
  pi.registerShortcut("ctrl+x", { ... });
  pi.registerFlag("my-flag", { ... });
}
```

扩展通过 [jiti](https://github.com/unjs/jiti) 加载，因此 TypeScript 无需编译即可工作。

如果工厂返回 `Promise`，pi 会在继续启动前等待它完成。这意味着异步初始化会在 `session_start` 之前、在 `resources_discover` 之前完成，并且在通过 `pi.registerProvider()` 排队的提供者注册被刷新之前完成。

### 异步工厂函数

使用异步工厂进行一次性启动工作，例如获取远程配置或动态发现可用模型。

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default async function (pi: ExtensionAPI) {
  const response = await fetch("http://localhost:1234/v1/models");
  const payload = (await response.json()) as {
    data: Array<{
      id: string;
      name?: string;
      context_window?: number;
      max_tokens?: number;
    }>;
  };

  pi.registerProvider("local-openai", {
    baseUrl: "http://localhost:1234/v1",
    apiKey: "LOCAL_OPENAI_API_KEY",
    api: "openai-completions",
    models: payload.data.map((model) => ({
      id: model.id,
      name: model.name ?? model.id,
      reasoning: false,
      input: ["text"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: model.context_window ?? 128000,
      maxTokens: model.max_tokens ?? 4096,
    })),
  });
}
```

此模式使获取的模型在正常启动和 `pi --list-models` 中可用。

### 扩展样式

**单文件** - 最简单，适用于小型扩展：

```
~/.pi/agent/extensions/
└── my-extension.ts
```

**带 index.ts 的目录** - 适用于多文件扩展：

```
~/.pi/agent/extensions/
└── my-extension/
    ├── index.ts        # 入口点（导出默认函数）
    ├── tools.ts        # 辅助模块
    └── utils.ts        # 辅助模块
```

**带依赖的包** - 适用于需要 npm 包的扩展：

```
~/.pi/agent/extensions/
└── my-extension/
    ├── package.json    # 声明依赖和入口点
    ├── package-lock.json
    ├── node_modules/   # npm install 之后
    └── src/
        └── index.ts
```

```json
// package.json
{
  "name": "my-extension",
  "dependencies": {
    "zod": "^3.0.0",
    "chalk": "^5.0.0"
  },
  "pi": {
    "extensions": ["./src/index.ts"]
  }
}
```

在扩展目录中运行 `npm install`，然后 `node_modules/` 中的导入会自动工作。

## 事件

### 生命周期概览

```
pi 启动
  │
  ├─► session_start { reason: "startup" }
  └─► resources_discover { reason: "startup" }
      │
      ▼
用户发送提示 ────────────────────────────────────┐
  │                                                │
  ├─►（首先检查扩展命令，找到则绕过）            │
  ├─► input（可以拦截、转换或处理）                │
  ├─►（如果未处理，展开技能/模板）                │
  ├─► before_agent_start（可以注入消息、修改系统提示）
  ├─► agent_start                                  │
  ├─► message_start / message_update / message_end │
  │                                                │
  │   ┌─── 轮次（LLM 调用工具时重复）───┐         │
  │   │                               │         │
  │   ├─► turn_start                  │         │
  │   ├─► context（可以修改消息）     │         │
  │   ├─► before_provider_request（可以检查或替换有效负载）
  │   ├─► after_provider_response（状态 + 头部，在消耗流之前）
  │   │                               │         │
  │   │   LLM 响应，可能调用工具：     │         │
  │   │     ├─► tool_execution_start  │         │
  │   │     ├─► tool_call（可以阻止）  │         │
  │   │     ├─► tool_execution_update │         │
  │   │     ├─► tool_result（可以修改）│         │
  │   │     └─► tool_execution_end    │         │
  │   │                               │         │
  │   └─► turn_end                    │         │
  │                                                │
  └─► agent_end                                    │
                                                   │
用户发送另一个提示 ◄───────────────────────────────┘

/new（新会话）或 /resume（切换会话）
  ├─► session_before_switch（可以取消）
  ├─► session_shutdown
  ├─► session_start { reason: "new" | "resume", previousSessionFile? }
  └─► resources_discover { reason: "startup" }

/fork 或 /clone
  ├─► session_before_fork（可以取消）
  ├─► session_shutdown
  ├─► session_start { reason: "fork", previousSessionFile }
  └─► resources_discover { reason: "startup" }

/compact 或自动压缩
  ├─► session_before_compact（可以取消或自定义）
  └─► session_compact

/tree 导航
  ├─► session_before_tree（可以取消或自定义）
  └─► session_tree

/model 或 Ctrl+P（模型选择/循环）
  ├─► thinking_level_select（如果模型更改改变/限制思考级别）
  └─► model_select

思考级别更改（设置、快捷键、pi.setThinkingLevel()）
  └─► thinking_level_select

退出（Ctrl+C、Ctrl+D、SIGHUP、SIGTERM）
  └─► session_shutdown
```

### 资源事件

#### resources_discover

在 `session_start` 之后触发，以便扩展可以贡献额外的技能、提示和主题路径。
启动路径使用 `reason: "startup"`。重新加载使用 `reason: "reload"`。

```typescript
pi.on("resources_discover", async (event, _ctx) => {
  // event.cwd - 当前工作目录
  // event.reason - "startup" | "reload"
  return {
    skillPaths: ["/path/to/skills"],
    promptPaths: ["/path/to/prompts"],
    themePaths: ["/path/to/themes"],
  };
});
```

### 会话事件

参见 [Session Format](session-format.md) 了解会话存储内部和 SessionManager API。

#### session_start

当会话启动、加载或重新加载时触发。

```typescript
pi.on("session_start", async (event, ctx) => {
  // event.reason - "startup" | "reload" | "new" | "resume" | "fork"
  // event.previousSessionFile - 适用于 "new"、"resume" 和 "fork"
  ctx.ui.notify(`会话：${ctx.sessionManager.getSessionFile() ?? "临时"}`, "info");
});
```

#### session_before_switch

在启动新会话（`/new`）或切换会话（`/resume`）之前触发。

```typescript
pi.on("session_before_switch", async (event, ctx) => {
  // event.reason - "new" 或 "resume"
  // event.targetSessionFile - 我们正在切换到的会话（仅适用于 "resume"）

  if (event.reason === "new") {
    const ok = await ctx.ui.confirm("清空？", "删除所有消息吗？");
    if (!ok) return { cancel: true };
  }
});
```

在成功切换或新会话操作后，pi 为旧扩展实例发出 `session_shutdown`，为新会话重新加载并重新绑定扩展，然后发出带有 `reason: "new" | "resume"` 和 `previousSessionFile` 的 `session_start`。
在 `session_shutdown` 中进行清理工作，然后在 `session_start` 中重建任何内存中的状态。

#### session_before_fork

通过 `/fork` 分支或通过 `/clone` 克隆时触发。

```typescript
pi.on("session_before_fork", async (event, ctx) => {
  // event.entryId - 选定条目的 ID
  // event.position - /fork 为 "before"，/clone 为 "at"
  return { cancel: true }; // 取消分支/克隆
  // 或
  return { skipConversationRestore: true }; // 保留用于未来对话恢复控制
});
```

在成功分支或克隆后，pi 为旧扩展实例发出 `session_shutdown`，为新会话重新加载并重新绑定扩展，然后发出带有 `reason: "fork"` 和 `previousSessionFile` 的 `session_start`。
在 `session_shutdown` 中进行清理工作，然后在 `session_start` 中重建任何内存中的状态。

#### session_before_compact / session_compact

在压缩时触发。参见 [compaction.md](compaction.md) 了解详细信息。

```typescript
pi.on("session_before_compact", async (event, ctx) => {
  const { preparation, branchEntries, customInstructions, signal } = event;

  // 取消：
  return { cancel: true };

  // 自定义摘要：
  return {
    compaction: {
      summary: "...",
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
    }
  };
});

pi.on("session_compact", async (event, ctx) => {
  // event.compactionEntry - 保存的压缩
  // event.fromExtension - 是否由扩展提供
});
```

#### session_before_tree / session_tree

在 `/tree` 导航时触发。参见 [Sessions](sessions.md) 了解树导航概念。

```typescript
pi.on("session_before_tree", async (event, ctx) => {
  const { preparation, signal } = event;
  return { cancel: true };
  // 或提供自定义摘要：
  return { summary: { summary: "...", details: {} } };
});

pi.on("session_tree", async (event, ctx) => {
  // event.newLeafId, oldLeafId, summaryEntry, fromExtension
});
```

#### session_shutdown

在扩展运行时被拆除之前触发。

```typescript
pi.on("session_shutdown", async (event, ctx) => {
  // event.reason - "quit" | "reload" | "new" | "resume" | "fork"
  // event.targetSessionFile - 会话替换流程的目标会话
  // 清理、保存状态等
});
```

### 代理事件

#### before_agent_start

在用户提交提示后、代理循环之前触发。可以注入消息和/或修改系统提示。

```typescript
pi.on("before_agent_start", async (event, ctx) => {
  // event.prompt - 用户的提示文本
  // event.images - 附加的图像（如果有）
  // event.systemPrompt - 此处理程序当前的链式系统提示
  //   （包括来自先前 before_agent_start 处理程序的更改）
  // event.systemPromptOptions - 用于构建系统提示的结构化选项
  //   .customPrompt - 任何自定义系统提示（来自 --system-prompt、SYSTEM.md 或自定义模板）
  //   .selectedTools - 提示中当前活动的工具
  //   .toolSnippets - 每个工具的单行描述
  //   .promptGuidelines - 自定义指南要点
  //   .appendSystemPrompt - 来自 --append-system-prompt 标志的文本
  //   .cwd - 工作目录
  //   .contextFiles - AGENTS.md 文件和其他加载的上下文文件
  //   .skills - 加载的技能

  return {
    // 注入持久化消息（存储在会话中，发送给 LLM）
    message: {
      customType: "my-extension",
      content: "给 LLM 的额外上下文",
      display: true,
    },
    // 替换此轮的系统提示（在扩展之间链式传递）
    systemPrompt: event.systemPrompt + "\n\n此轮的额外说明...",
  };
});
```

`systemPromptOptions` 字段使扩展能够访问与 Pi 用于构建系统提示相同的结构化数据。这让你可以检查 Pi 加载了什么——自定义提示、指南、工具片段、上下文文件、技能——而无需重新发现资源或重新解析标志。当你的扩展需要在尊重用户提供的配置的同时对系统提示进行深入、知情的更改时，请使用它。

在 `before_agent_start` 内部，`event.systemPrompt` 和 `ctx.getSystemPrompt()` 都反映了截至当前处理程序的链式系统提示。后续的 `before_agent_start` 处理程序仍然可以再次修改它。

#### agent_start / agent_end

每个用户提示触发一次。

```typescript
pi.on("agent_start", async (_event, ctx) => {});

pi.on("agent_end", async (event, ctx) => {
  // event.messages - 来自此提示的消息
});
```

#### turn_start / turn_end

每个轮次（一个 LLM 响应 + 工具调用）触发一次。

```typescript
pi.on("turn_start", async (event, ctx) => {
  // event.turnIndex, event.timestamp
});

pi.on("turn_end", async (event, ctx) => {
  // event.turnIndex, event.message, event.toolResults
});
```

#### message_start / message_update / message_end

为消息生命周期更新触发。

- `message_start` 和 `message_end` 为用户、助手和工具结果消息触发。
- `message_update` 为助手流式更新触发。
- `message_end` 处理程序可以返回 `{ message }` 来替换最终消息。替换必须保持相同的 `role`。

```typescript
pi.on("message_start", async (event, ctx) => {
  // event.message
});

pi.on("message_update", async (event, ctx) => {
  // event.message
  // event.assistantMessageEvent（逐令牌流事件）
});

pi.on("message_end", async (event, ctx) => {
  if (event.message.role !== "assistant") return;

  return {
    message: {
      ...event.message,
      usage: {
        ...event.message.usage,
        cost: {
          ...event.message.usage.cost,
          total: 0.123,
        },
      },
    },
  };
});
```

#### tool_execution_start / tool_execution_update / tool_execution_end

为工具执行生命周期更新触发。

在并行工具模式下：
- `tool_execution_start` 在预检阶段按助手源顺序发出
- `tool_execution_update` 事件可能在工具之间交错
- `tool_execution_end` 在每个工具完成后按工具完成顺序发出
- 最终的 `toolResult` 消息事件仍然在后续按助手源顺序发出

```typescript
pi.on("tool_execution_start", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.args
});

pi.on("tool_execution_update", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.args, event.partialResult
});

pi.on("tool_execution_end", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.result, event.isError
});
```

#### context

在每个 LLM 调用之前触发。非破坏性地修改消息。参见 [Session Format](session-format.md) 了解消息类型。

```typescript
pi.on("context", async (event, ctx) => {
  // event.messages - 深拷贝，可安全修改
  const filtered = event.messages.filter(m => !shouldPrune(m));
  return { messages: filtered };
});
```

#### before_provider_request

在构建提供者特定有效负载后、发送请求之前立即触发。处理程序按扩展加载顺序运行。返回 `undefined` 会保持有效负载不变。返回任何其他值会替换后续处理程序和实际请求的有效负载。

此钩子可以重写提供者级别的系统指令或完全删除它们。这些有效负载级别的更改不会反映在 `ctx.getSystemPrompt()` 中，后者报告 Pi 的系统提示字符串而不是最终序列化的提供者有效负载。

```typescript
pi.on("before_provider_request", (event, ctx) => {
  console.log(JSON.stringify(event.payload, null, 2));

  // 可选：替换有效负载
  // return { ...event.payload, temperature: 0 };
});
```

这主要用于调试提供者序列化和缓存行为。

#### after_provider_response

在收到 HTTP 响应后、在消耗其流之前触发。处理程序按扩展加载顺序运行。

```typescript
pi.on("after_provider_response", (event, ctx) => {
  // event.status - HTTP 状态码
  // event.headers - 规范化响应头
  if (event.status === 429) {
    console.log("限速", event.headers["retry-after"]);
  }
});
```

头部可用性取决于提供者和传输。抽象 HTTP 响应的提供者可能不会暴露头部。

### 模型事件

#### model_select

通过 `/model` 命令、模型循环（`Ctrl+P`）或会话恢复更改模型时触发。

```typescript
pi.on("model_select", async (event, ctx) => {
  // event.model - 新选择的模型
  // event.previousModel - 先前的模型（第一次选择时为 undefined）
  // event.source - "set" | "cycle" | "restore"

  const prev = event.previousModel
    ? `${event.previousModel.provider}/${event.previousModel.id}`
    : "无";
  const next = `${event.model.provider}/${event.model.id}`;

  ctx.ui.notify(`模型已更改（${event.source}）：${prev} → ${next}`, "info");
});
```

使用它来更新 UI 元素（状态栏、页脚）或在活动模型更改时执行模型特定的初始化。

#### thinking_level_select

在思考级别更改时触发。仅用于通知；忽略处理程序返回值。

```typescript
pi.on("thinking_level_select", async (event, ctx) => {
  // event.level - 新选择的思考级别
  // event.previousLevel - 先前的思考级别

  ctx.ui.setStatus("thinking", `思考：${event.level}`);
});
```

当 `pi.setThinkingLevel()`、模型更改或内置的思考级别控件更改活动思考级别时，使用它来更新扩展 UI。

### 工具事件

#### tool_call

在 `tool_execution_start` 之后、工具执行之前触发。**可以阻止**。使用 `isToolCallEventType` 进行缩小并获取类型化输入。

在 `tool_call` 运行之前，pi 等待先前发出的代理事件完成通过 `AgentSession` 的处理。这意味着 `ctx.sessionManager` 通过当前的助手工具调用消息是最新的。

在默认的并行工具执行模式下，来自同一助手消息的兄弟工具调用按顺序预检，然后并发执行。`tool_call` 不保证在 `ctx.sessionManager` 中看到来自同一助手消息的兄弟工具结果。

`event.input` 是可变的。在执行前就地修改它以修补工具参数。

行为保证：
- 对 `event.input` 的修改影响实际的工具执行
- 后续的 `tool_call` 处理程序看到先前处理程序所做的修改
- 修改后不执行重新验证
- `tool_call` 的返回值仅通过 `{ block: true, reason?: string }` 控制阻止

```typescript
import { isToolCallEventType } from "@earendil-works/pi-coding-agent";

pi.on("tool_call", async (event, ctx) => {
  // event.toolName - "bash"、"read"、"write"、"edit" 等
  // event.toolCallId
  // event.input - 工具参数（可变）

  // 内置工具：无需类型参数
  if (isToolCallEventType("bash", event)) {
    // event.input 是 { command: string; timeout?: number }
    event.input.command = `source ~/.profile\n${event.input.command}`;

    if (event.input.command.includes("rm -rf")) {
      return { block: true, reason: "危险命令" };
    }
  }

  if (isToolCallEventType("read", event)) {
    // event.input 是 { path: string; offset?: number; limit?: number }
    console.log(`正在读取：${event.input.path}`);
  }
});
```

#### 类型化自定义工具输入

自定义工具应该导出它们的输入类型：

```typescript
// my-extension.ts
export type MyToolInput = Static<typeof myToolSchema>;
```

使用带有显式类型参数的 `isToolCallEventType`：

```typescript
import { isToolCallEventType } from "@earendil-works/pi-coding-agent";
import type { MyToolInput } from "my-extension";

pi.on("tool_call", (event) => {
  if (isToolCallEventType<"my_tool", MyToolInput>("my_tool", event)) {
    event.input.action;  // 类型化
  }
});
```

#### tool_result

在工具执行完成后、`tool_execution_end` 之前触发，并发出最终工具结果消息事件。**可以修改结果**。

在并行工具模式下，`tool_result` 和 `tool_execution_end` 可能按工具完成顺序交错，而最终的 `toolResult` 消息事件仍然在后续按助手源顺序发出。

`tool_result` 处理程序像中间件一样链式传递：
- 处理程序按扩展加载顺序运行
- 每个处理程序看到先前处理程序更改后的最新结果
- 处理程序可以返回部分补丁（`content`、`details` 或 `isError`）；省略的字段保留其当前值

在处理程序内部使用 `ctx.signal` 进行嵌套异步工作。这让 Esc 可以取消模型调用、`fetch()` 和扩展启动的其他可中止操作。

```typescript
import { isBashToolResult } from "@earendil-works/pi-coding-agent";

pi.on("tool_result", async (event, ctx) => {
  // event.toolName, event.toolCallId, event.input
  // event.content, event.details, event.isError

  if (isBashToolResult(event)) {
    // event.details 被类型化为 BashToolDetails
  }

  const response = await fetch("https://example.com/summarize", {
    method: "POST",
    body: JSON.stringify({ content: event.content }),
    signal: ctx.signal,
  });

  // 修改结果：
  return { content: [...], details: {...}, isError: false };
});
```

### 用户 Bash 事件

#### user_bash

当用户执行 `!` 或 `!!` 命令时触发。**可以拦截**。

```typescript
import { createLocalBashOperations } from "@earendil-works/pi-coding-agent";

pi.on("user_bash", (event, ctx) => {
  // event.command - bash 命令
  // event.excludeFromContext - !! 前缀时为 true
  // event.cwd - 工作目录

  // 选项 1：提供自定义操作（例如 SSH）
  return { operations: remoteBashOps };

  // 选项 2：包装 pi 的内置本地 bash 后端
  const local = createLocalBashOperations();
  return {
    operations: {
      exec(command, cwd, options) {
        return local.exec(`source ~/.profile\n${command}`, cwd, options);
      }
    }
  };

  // 选项 3：完全替换 - 直接返回结果
  return { result: { output: "...", exitCode: 0, cancelled: false, truncated: false } };
});
```

### 输入事件

#### input

在收到用户输入时、在检查扩展命令之后但在技能和模板展开之前触发。事件看到原始输入文本，因此 `/skill:foo` 和 `/template` 尚未展开。

**处理顺序：**
1. 首先检查扩展命令（`/cmd`）- 如果找到，运行处理程序并跳过输入事件
2. 触发 `input` 事件 - 可以拦截、转换或处理
3. 如果未处理：技能命令（`/skill:name`）展开为技能内容
4. 如果未处理：提示模板（`/template`）展开为模板内容
5. 代理处理开始（`before_agent_start` 等）

```typescript
pi.on("input", async (event, ctx) => {
  // event.text - 原始输入（在技能/模板展开之前）
  // event.images - 附加的图像（如果有）
  // event.source - "interactive"（键入）、"rpc"（API）或 "extension"（通过 sendUserMessage）

  // 转换：在展开前重写输入
  if (event.text.startsWith("?quick "))
    return { action: "transform", text: `简短回答：${event.text.slice(7)}` };

  // 处理：不使用 LLM 响应（扩展显示自己的反馈）
  if (event.text === "ping") {
    ctx.ui.notify("pong", "info");
    return { action: "handled" };
  }

  // 按源路由：跳过扩展注入消息的处理
  if (event.source === "extension") return { action: "continue" };

  // 在展开前拦截技能命令
  if (event.text.startsWith("/skill:")) {
    // 可以转换、阻止或放行
  }

  return { action: "continue" };  // 默认：传递给展开
});
```

**结果：**
- `continue` - 不变地传递（如果处理程序未返回任何内容则为默认）
- `transform` - 修改文本/图像，然后继续展开
- `handled` - 完全绕过代理（第一个返回此值的处理程序获胜）

转换在处理程序之间链式传递。参见 [input-transform.ts](../examples/extensions/input-transform.ts)。

## ExtensionContext

所有处理程序都接收 `ctx: ExtensionContext`。

### ctx.ui

用于用户交互的 UI 方法。参见 [自定义 UI](#自定义-ui) 了解完整详细信息。

### ctx.hasUI

在打印模式（`-p`）和 JSON 模式下为 `false`。在交互和 RPC 模式下为 `true`。在 RPC 模式下，对话框方法（`select`、`confirm`、`input`、`editor`）通过扩展 UI 子协议工作，即发即弃方法（`notify`、`setStatus`、`setWidget`、`setTitle`、`setEditorText`）向客户端发出请求。一些 TUI 特定的方法是空操作或返回默认值（参见 [rpc.md](rpc.md#extension-ui-protocol)）。

### ctx.cwd

当前工作目录。

### ctx.sessionManager

会话状态的只读访问。参见 [Session Format](session-format.md) 了解完整的 SessionManager API 和条目类型。

对于 `tool_call`，此状态在处理程序运行前通过当前助手消息进行同步。在并行工具执行模式下，它仍然不保证包含来自同一助手消息的兄弟工具结果。

```typescript
ctx.sessionManager.getEntries()       // 所有条目
ctx.sessionManager.getBranch()        // 当前分支
ctx.sessionManager.getLeafId()        // 当前叶子条目 ID
```

### ctx.modelRegistry / ctx.model

访问模型和 API 密钥。

### ctx.signal

当前代理中止信号，或在无代理轮次活动时为 `undefined`。

将其用于扩展处理程序启动的可中止嵌套工作，例如：
- `fetch(..., { signal: ctx.signal })`
- 接受 `signal` 的模型调用
- 接受 `AbortSignal` 的文件或进程帮助程序

`ctx.signal` 通常在活动轮次事件期间定义，例如 `tool_call`、`tool_result`、`message_update` 和 `turn_end`。
它通常在空闲或非轮次上下文中为 `undefined`，例如会话事件、扩展命令和在 pi 空闲时触发的快捷键。

```typescript
pi.on("tool_result", async (event, ctx) => {
  const response = await fetch("https://example.com/api", {
    method: "POST",
    body: JSON.stringify(event),
    signal: ctx.signal,
  });

  const data = await response.json();
  return { details: data };
});
```

### ctx.isIdle() / ctx.abort() / ctx.hasPendingMessages()

控制流帮助程序。

### ctx.shutdown()

请求 pi 正常关闭。

- **交互模式：** 延迟到代理变为空闲（在处理所有排队的转向和后续消息后）。
- **RPC 模式：** 延迟到下一个空闲状态（在完成当前命令响应后，等待下一个命令时）。
- **打印模式：** 空操作。进程在所有提示都处理后自动退出。

在退出前向所有扩展发出 `session_shutdown` 事件。在所有上下文（事件处理程序、工具、命令、快捷键）中可用。

```typescript
pi.on("tool_call", (event, ctx) => {
  if (isFatal(event.input)) {
    ctx.shutdown();
  }
});
```

### ctx.getContextUsage()

返回活动模型的当前上下文使用情况。可用时使用最后的助手使用情况，然后为尾随消息估算令牌数。

```typescript
const usage = ctx.getContextUsage();
if (usage && usage.tokens > 100_000) {
  // ...
}
```

### ctx.compact()

触发压缩而不等待完成。使用 `onComplete` 和 `onError` 进行后续操作。

```typescript
ctx.compact({
  customInstructions: "专注于最近的更改",
  onComplete: (result) => {
    ctx.ui.notify("压缩完成", "info");
  },
  onError: (error) => {
    ctx.ui.notify(`压缩失败：${error.message}`, "error");
  },
});
```

### ctx.getSystemPrompt()

返回 Pi 的当前系统提示字符串。

- 在 `before_agent_start` 期间，这反映了截至当前轮次目前为止的链式系统提示更改。
- 它不包括后续的 `context` 消息修改。
- 它不包括 `before_provider_request` 有效负载重写。
- 如果后续加载的扩展在你的扩展之后运行，它们仍然可以更改最终发送的内容。

```typescript
pi.on("before_agent_start", (event, ctx) => {
  const prompt = ctx.getSystemPrompt();
  console.log(`系统提示长度：${prompt.length}`);
});
```

## ExtensionCommandContext

命令处理程序接收 `ExtensionCommandContext`，它通过会话控制方法扩展 `ExtensionContext`。这些仅在命令中可用，因为如果从事件处理程序调用，它们可能会导致死锁。

### ctx.waitForIdle()

等待代理完成流式传输：

```typescript
pi.registerCommand("my-cmd", {
  handler: async (args, ctx) => {
    await ctx.waitForIdle();
    // 代理现在空闲，可以安全地修改会话
  },
});
```

### ctx.newSession(options?)

创建新会话：

```typescript
const parentSession = ctx.sessionManager.getSessionFile();
const kickoff = "在替换会话中继续";

const result = await ctx.newSession({
  parentSession,
  setup: async (sm) => {
    sm.appendMessage({
      role: "user",
      content: [{ type: "text", text: "来自先前会话的上下文..." }],
      timestamp: Date.now(),
    });
  },
  withSession: async (ctx) => {
    // 仅在此处使用替换会话的 ctx。
    await ctx.sendUserMessage(kickoff);
  },
});

if (result.cancelled) {
  // 扩展取消了新会话
}
```

选项：
- `parentSession`：要在新会话头部记录的父会话文件
- `setup`：在 `withSession` 运行之前变更新会话的 `SessionManager`
- `withSession`：对新鲜的替换会话上下文运行切换后的工作。不要使用捕获的旧 `pi` / 命令 `ctx`；参见 [会话替换生命周期和陷阱](#会话替换生命周期和陷阱)。

### ctx.fork(entryId, options?)

从特定条目分支，创建新会话文件：

```typescript
const result = await ctx.fork("entry-id-123", {
  withSession: async (ctx) => {
    // 仅在此处使用替换会话的 ctx。
    ctx.ui.notify("现在在分支会话中", "info");
  },
});
if (result.cancelled) {
  // 扩展取消了分支
}

const cloneResult = await ctx.fork("entry-id-456", { position: "at" });
if (cloneResult.cancelled) {
  // 扩展取消了克隆
}
```

选项：
- `position`：`"before"`（默认）在选定用户消息之前分支，将该提示恢复到编辑器中
- `position`：`"at"` 复制通过选定条目的活动路径而不恢复编辑器文本
- `withSession`：对新鲜的替换会话上下文运行切换后的工作。不要使用捕获的旧 `pi` / 命令 `ctx`；参见 [会话替换生命周期和陷阱](#会话替换生命周期和陷阱)。

### ctx.navigateTree(targetId, options?)

导航到会话树中的不同点：

```typescript
const result = await ctx.navigateTree("entry-id-456", {
  summarize: true,
  customInstructions: "专注于错误处理更改",
  replaceInstructions: false, // true = 完全替换默认提示
  label: "review-checkpoint",
});
```

选项：
- `summarize`：是否生成废弃分支的摘要
- `customInstructions`：摘要器的自定义说明
- `replaceInstructions`：如果为 true，`customInstructions` 替换默认提示而不是追加
- `label`：要附加到分支摘要条目的标签（或不摘要时附加到目标条目）

### ctx.switchSession(sessionPath, options?)

切换到不同的会话文件：

```typescript
const result = await ctx.switchSession("/path/to/session.jsonl", {
  withSession: async (ctx) => {
    await ctx.sendUserMessage("在替换会话中继续工作");
  },
});
if (result.cancelled) {
  // 扩展通过 session_before_switch 取消了切换
}
```

选项：
- `withSession`：对新鲜的替换会话上下文运行切换后的工作。不要使用捕获的旧 `pi` / 命令 `ctx`；参见 [会话替换生命周期和陷阱](#会话替换生命周期和陷阱)。

要发现可用的会话，请使用静态的 `SessionManager.list()` 或 `SessionManager.listAll()` 方法：

```typescript
import { SessionManager } from "@earendil-works/pi-coding-agent";

pi.registerCommand("switch", {
  description: "切换到另一个会话",
  handler: async (args, ctx) => {
    const sessions = await SessionManager.list(ctx.cwd);
    if (sessions.length === 0) return;
    const choice = await ctx.ui.select(
      "选择会话：",
      sessions.map(s => s.file),
    );
    if (choice) {
      await ctx.switchSession(choice, {
        withSession: async (ctx) => {
          ctx.ui.notify("已切换会话", "info");
        },
      });
    }
  },
});
```

### 会话替换生命周期和陷阱

`withSession` 接收一个新鲜的 `ReplacedSessionContext`，它通过绑定到替换会话的异步 `sendMessage()` 和 `sendUserMessage()` 帮助程序扩展 `ExtensionCommandContext`。

生命周期和陷阱：
- `withSession` 仅在旧会话发出 `session_shutdown`、旧运行时被拆除、替换会话被重新绑定并且新扩展实例已收到 `session_start` 之后才运行。
- 回调仍然在原始闭包中执行，而不是在新扩展实例内部。这意味着你的旧扩展实例可能已经在 `withSession` 开始之前运行了其关闭清理。
- 捕获的旧 `pi` / 旧命令 `ctx` 会话绑定对象在替换后是过时的，如果使用会抛出异常。仅使用传递给 `withSession` 的 `ctx` 进行会话绑定工作。
- 先前提取的原始对象仍然是你的责任。例如，如果你在替换前捕获了 `const sm = ctx.sessionManager`，`sm` 仍然是旧的 `SessionManager` 对象。替换后不要重用它。
- `withSession` 中的代码应该假设你的 `session_shutdown` 处理程序使无效的任何状态已经消失。仅捕获在关闭后能干净地存活的纯数据，例如字符串、ID 和序列化配置。

安全模式：

```typescript
pi.registerCommand("handoff", {
  handler: async (_args, ctx) => {
    const kickoff = "从替换会话继续";
    await ctx.newSession({
      withSession: async (ctx) => {
        await ctx.sendUserMessage(kickoff);
      },
    });
  },
});
```

不安全模式：

```typescript
pi.registerCommand("handoff", {
  handler: async (_args, ctx) => {
    const oldSessionManager = ctx.sessionManager;
    await ctx.newSession({
      withSession: async (_ctx) => {
        // 过时的旧对象：不要这样做
        oldSessionManager.getSessionFile();
        pi.sendUserMessage("错误");
      },
    });
  },
});
```

### ctx.reload()

运行与 `/reload` 相同的重新加载流程。

```typescript
pi.registerCommand("reload-runtime", {
  description: "重新加载扩展、技能、提示和主题",
  handler: async (_args, ctx) => {
    await ctx.reload();
    return;
  },
});
```

重要行为：
- `await ctx.reload()` 为当前扩展运行时发出 `session_shutdown`
- 然后它重新加载资源并发出带有 `reason: "reload"` 的 `session_start` 和带有 reason `"reload"` 的 `resources_discover`
- 当前运行的命令处理程序仍然在旧调用帧中继续
- `await ctx.reload()` 之后的代码仍然从预重新加载版本运行
- `await ctx.reload()` 之后的代码不得假设旧的内存中扩展状态仍然有效
- 处理程序返回后，未来的命令/事件/工具调用使用新的扩展版本

为了可预测的行为，将重新加载视为该处理程序的终端（`await ctx.reload(); return;`）。

工具使用 `ExtensionContext` 运行，因此它们无法直接调用 `ctx.reload()`。使用命令作为重新加载入口点，然后暴露一个工具，将该命令作为后续用户消息排队。

LLM 可以调用以触发重新加载的示例工具：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  pi.registerCommand("reload-runtime", {
    description: "重新加载扩展、技能、提示和主题",
    handler: async (_args, ctx) => {
      await ctx.reload();
      return;
    },
  });

  pi.registerTool({
    name: "reload_runtime",
    label: "重新加载运行时",
    description: "重新加载扩展、技能、提示和主题",
    parameters: Type.Object({}),
    async execute() {
      pi.sendUserMessage("/reload-runtime", { deliverAs: "followUp" });
      return {
        content: [{ type: "text", text: "已将 /reload-runtime 排队为后续命令。" }],
      };
    },
  });
}
```

## ExtensionAPI 方法

### pi.on(event, handler)

订阅事件。参见 [事件](#事件) 了解事件类型和返回值。

### pi.registerTool(definition)

注册 LLM 可调用的自定义工具。参见 [自定义工具](#自定义工具) 了解完整详细信息。

`pi.registerTool()` 在扩展加载期间和启动后都可以工作。你可以在 `session_start`、命令处理程序或其他事件处理程序内部调用它。新工具在同一会话中立即刷新，因此它们出现在 `pi.getAllTools()` 中，并且 LLM 无需 `/reload` 即可调用它们。

使用 `pi.setActiveTools()` 在运行时启用或禁用工具（包括动态添加的工具）。

使用 `promptSnippet` 使自定义工具加入 `可用工具` 中的单行条目，使用 `promptGuidelines` 在工具活动时向默认 `指南` 部分追加工具特定要点。

**重要：** `promptGuidelines` 要点被平铺追加到 `指南` 部分，没有工具名称前缀。每个指南必须命名它所指的工具——避免“在...时使用此工具”，因为 LLM 无法知道“此”指哪个工具。改为写“在...时使用 my_tool”。

参见 [dynamic-tools.ts](../examples/extensions/dynamic-tools.ts) 了解完整示例。

```typescript
import { Type } from "typebox";
import { StringEnum } from "@earendil-works/pi-ai";

pi.registerTool({
  name: "my_tool",
  label: "我的工具",
  description: "此工具的作用",
  promptSnippet: "根据操作总结或转换文本",
  promptGuidelines: ["当用户要求总结先前生成的文本时使用 my_tool。"],
  parameters: Type.Object({
    action: StringEnum(["list", "add"] as const),
    text: Type.Optional(Type.String()),
  }),
  prepareArguments(args) {
    // 可选的兼容性垫片。在模式验证前运行。
    // 返回当前模式形状，例如将旧字段折叠到现代参数对象中。
    return args;
  },

  async execute(toolCallId, params, signal, onUpdate, ctx) {
    // 流式传输进度
    onUpdate?.({ content: [{ type: "text", text: "正在工作..." }] });

    return {
      content: [{ type: "text", text: "完成" }],
      details: { result: "..." },
    };
  },

  // 可选：自定义渲染
  renderCall(args, theme, context) { ... },
  renderResult(result, options, theme, context) { ... },
});
```

### pi.sendMessage(message, options?)

向会话注入自定义消息。

```typescript
pi.sendMessage({
  customType: "my-extension",
  content: "消息文本",
  display: true,
  details: { ... },
}, {
  triggerTurn: true,
  deliverAs: "steer",
});
```

**选项：**
- `deliverAs` - 交付模式：
  - `"steer"`（默认）- 在流式传输时排队消息。在当前助手轮次完成执行其工具调用之后、下一个 LLM 调用之前交付。
  - `"followUp"` - 等待代理完成。仅在代理没有更多工具调用时交付。
  - `"nextTurn"` - 为下一个用户提示排队。不中断或触发任何内容。
- `triggerTurn: true` - 如果代理空闲，立即触发 LLM 响应。仅适用于 `"steer"` 和 `"followUp"` 模式（对于 `"nextTurn"` 忽略）。

### pi.sendUserMessage(content, options?)

向代理发送用户消息。与发送自定义消息的 `sendMessage()` 不同，这发送看起来像是由用户键入的实际用户消息。始终触发一轮。

```typescript
// 简单文本消息
pi.sendUserMessage("2+2 等于多少？");

// 带内容数组（文本 + 图像）
pi.sendUserMessage([
  { type: "text", text: "描述此图像：" },
  { type: "image", source: { type: "base64", mediaType: "image/png", data: "..." } },
]);

// 在流式传输期间 - 必须指定交付模式
pi.sendUserMessage("专注于错误处理", { deliverAs: "steer" });
pi.sendUserMessage("然后总结", { deliverAs: "followUp" });
```

**选项：**
- `deliverAs` - 代理正在流式传输时是必需的：
  - `"steer"` - 排队消息以便在当前助手轮次完成执行其工具调用之后交付
  - `"followUp"` - 等待代理完成所有工具

未流式传输时，消息立即发送并触发新轮次。流式传输时没有 `deliverAs` 会抛出错误。

参见 [send-user-message.ts](../examples/extensions/send-user-message.ts) 了解完整示例。

### pi.appendEntry(customType, data?)

持久化扩展状态（**不参与** LLM 上下文）。

```typescript
pi.appendEntry("my-state", { count: 42 });

// 重新加载时恢复
pi.on("session_start", async (_event, ctx) => {
  for (const entry of ctx.sessionManager.getEntries()) {
    if (entry.type === "custom" && entry.customType === "my-state") {
      // 从 entry.data 重建
    }
  }
});
```

### pi.setSessionName(name)

设置会话显示名称（在会话选择器中显示而不是第一条消息）。

```typescript
pi.setSessionName("重构身份验证模块");
```

### pi.getSessionName()

获取当前会话名称（如果已设置）。

```typescript
const name = pi.getSessionName();
if (name) {
  console.log(`会话：${name}`);
}
```

### pi.setLabel(entryId, label)

设置或清除条目上的标签。标签是用于书签和导航的用户定义标记（显示在 `/tree` 选择器中）。

```typescript
// 设置标签
pi.setLabel(entryId, "checkpoint-before-refactor");

// 清除标签
pi.setLabel(entryId, undefined);

// 通过 sessionManager 读取标签
const label = ctx.sessionManager.getLabel(entryId);
```

标签持久化在会话中并在重启后保留。使用它们标记对话树中的重要点（轮次、检查点）。

### pi.registerCommand(name, options)

注册命令。

如果多个扩展注册相同的命令名称，pi 会保留所有命令并按加载顺序分配数字调用后缀，例如 `/review:1` 和 `/review:2`。

```typescript
pi.registerCommand("stats", {
  description: "显示会话统计信息",
  handler: async (args, ctx) => {
    const count = ctx.sessionManager.getEntries().length;
    ctx.ui.notify(`${count} 个条目`, "info");
  }
});
```

可选：为 `/command ...` 添加参数自动完成：

```typescript
import type { AutocompleteItem } from "@earendil-works/pi-tui";

pi.registerCommand("deploy", {
  description: "部署到环境",
  getArgumentCompletions: (prefix: string): AutocompleteItem[] | null => {
    const envs = ["dev", "staging", "prod"];
    const items = envs.map((e) => ({ value: e, label: e }));
    const filtered = items.filter((i) => i.value.startsWith(prefix));
    return filtered.length > 0 ? filtered : null;
  },
  handler: async (args, ctx) => {
    ctx.ui.notify(`正在部署：${args}`, "info");
  },
});
```

### pi.getCommands()

获取当前会话中可通过 `prompt` 调用的斜杠命令。包括扩展命令、提示模板和技能命令。
该列表匹配 RPC `get_commands` 排序：首先是扩展，然后是模板，然后是技能。

```typescript
const commands = pi.getCommands();
const bySource = commands.filter((command) => command.source === "extension");
const userScoped = commands.filter((command) => command.sourceInfo.scope === "user");
```

每个条目具有以下形状：

```typescript
{
  name: string; // 不带前导斜杠的可调用命令名称。可能带有后缀，如 "review:1"
  description?: string;
  source: "extension" | "prompt" | "skill";
  sourceInfo: {
    path: string;
    source: string;
    scope: "user" | "project" | "temporary";
    origin: "package" | "top-level";
    baseDir?: string;
  };
}
```

使用 `sourceInfo` 作为规范来源字段。不要从命令名称或临时路径解析推断所有权。

内置的交互命令（如 `/model` 和 `/settings`）不包含在此处。它们仅在交互模式中处理，并且如果通过 `prompt` 发送不会执行。

### pi.registerMessageRenderer(customType, renderer)

为带有你的 `customType` 的消息注册自定义 TUI 渲染器。参见 [自定义 UI](#自定义-ui)。

### pi.registerShortcut(shortcut, options)

注册键盘快捷键。参见 [keybindings.md](keybindings.md) 了解快捷键格式和内置键绑定。

```typescript
pi.registerShortcut("ctrl+shift+p", {
  description: "切换计划模式",
  handler: async (ctx) => {
    ctx.ui.notify("已切换！");
  },
});
```

### pi.registerFlag(name, options)

注册 CLI 标志。

```typescript
pi.registerFlag("plan", {
  description: "以计划模式启动",
  type: "boolean",
  default: false,
});

// 检查值
if (pi.getFlag("plan")) {
  // 计划模式已启用
}
```

### pi.exec(command, args, options?)

执行 shell 命令。

```typescript
await pi.exec("echo", ["hello"]);
```

### pi.setActiveTools(tools)

设置活动工具。这控制哪些工具对 LLM 可用。

```typescript
pi.setActiveTools(["read", "write"]); // 仅启用 read 和 write
pi.setActiveTools(["*"]); // 启用所有工具
pi.setActiveTools(["*", "-bash"]); // 启用除 bash 外的所有工具
```

### pi.getAllTools()

获取所有可用工具，包括扩展注册的工具。

```typescript
const tools = pi.getAllTools();
```

### pi.getActiveTools()

获取当前活动工具。

```typescript
const activeTools = pi.getActiveTools();
```

### pi.getFlag(name)

获取标志的值。

```typescript
const planMode = pi.getFlag("plan");
```

### pi.registerProvider(name, config)

注册自定义提供者。参见 [custom-provider.md](custom-provider.md)。

### pi.unregisterProvider(name)

注销先前注册的提供者。

```typescript
pi.unregisterProvider("my-provider");
```

### pi.setThinkingLevel(level)

设置思考级别。

```typescript
pi.setThinkingLevel("high");
```

### pi.getThinkingLevel()

获取当前思考级别。

```typescript
const level = pi.getThinkingLevel();
```

### pi.getModel()

获取当前活动模型。

```typescript
const model = pi.getModel();
```

### pi.setModel(provider, modelId)

设置活动模型。

```typescript
pi.setModel("anthropic", "claude-3-5-sonnet-latest");
```

## 状态管理

使用 `pi.appendEntry()` 持久化状态。这会将会话条目添加到会话历史中，在重启后仍然保留。

```typescript
// 保存
pi.appendEntry("my-extension-state", { count: 42, lastAction: "edit" });

// 加载
pi.on("session_start", async (_event, ctx) => {
  for (const entry of ctx.sessionManager.getEntries()) {
    if (entry.type === "custom" && entry.customType === "my-extension-state") {
      // 使用 entry.data 恢复状态
    }
  }
});
```

对于仅在当前会话期间需要的临时内存状态，只需使用模块级变量。

## 自定义工具

自定义工具让你扩展 LLM 的能力。使用 `pi.registerTool()` 注册它们。

### 工具定义

```typescript
pi.registerTool({
  name: "my_tool",           // LLM 调用的标识符
  label: "我的工具",         // TUI 中显示的人类可读名称
  description: "工具的作用", // 告诉 LLM 何时使用此工具
  parameters: Type.Object({  // 使用 typebox 定义参数模式
    filename: Type.String(),
    content: Type.String(),
  }),
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    // 工具实现
    return {
      content: [{ type: "text", text: "结果" }],
      details: { ... },
    };
  },
});
```

### execute 函数

`execute` 接收：
- `toolCallId` - 此工具调用的唯一 ID
- `params` - 验证后的工具参数（匹配你的参数模式）
- `signal` - 用于取消的 AbortSignal
- `onUpdate` - 用于流式传输部分结果的回调
- `ctx` - ExtensionContext

### 流式结果

使用 `onUpdate` 流式传输部分结果：

```typescript
async execute(toolCallId, params, signal, onUpdate, ctx) {
  onUpdate?.({ content: [{ type: "text", text: "开始..." }] });
  await delay(500);
  onUpdate?.({ content: [{ type: "text", text: "继续..." }] });
  await delay(500);
  return {
    content: [{ type: "text", text: "完成！" }],
    details: {},
  };
}
```

### 自定义渲染

使用 `renderCall` 和 `renderResult` 自定义工具在 TUI 中的显示方式：

```typescript
pi.registerTool({
  name: "my_tool",
  // ...
  renderCall(args, theme, context) {
    // 返回自定义渲染的工具调用
    return [theme.fg("accent", "自定义工具调用：" + args.filename)];
  },
  renderResult(result, options, theme, context) {
    // 返回自定义渲染的工具结果
    return [theme.fg("success", "自定义工具结果")];
  },
});
```

### 提示片段和指南

使用 `promptSnippet` 在 `可用工具` 中包含一行描述，使用 `promptGuidelines` 在工具活动时向系统提示添加指南。

```typescript
pi.registerTool({
  name: "my_tool",
  promptSnippet: "执行自定义操作",
  promptGuidelines: [
    "当用户要求 X 时使用 my_tool",
    "使用 Y 参数执行 Z 时使用 my_tool",
  ],
  // ...
});
```

**重要：** 指南必须明确命名工具！不要写“使用此工具”，要写“使用 my_tool”。

## 自定义 UI

通过 `ctx.ui.custom()` 使用自定义 UI 组件创建交互式体验。

### 基本用法

```typescript
const result = await ctx.ui.custom<string | null>((tui, theme, keybindings, done) => {
  // 你的组件实现
  return {
    render(width: number) {
      return [
        theme.fg("accent", "我的自定义 UI"),
        "",
        "按 Enter 确认，按 Escape 取消",
      ];
    },
    handleInput(data: string) {
      if (matchesKey(data, Key.enter)) {
        done("确认");
      } else if (matchesKey(data, Key.escape)) {
        done(null);
      }
    },
    invalidate() {},
  };
});
```

### 内置组件

使用来自 `@earendil-works/pi-tui` 的内置组件：

```typescript
import { Text, Box, Container, SelectList } from "@earendil-works/pi-tui";
import { DynamicBorder } from "@earendil-works/pi-coding-agent";
```

### 覆盖层

使用 `{ overlay: true }` 在现有内容上方渲染而不清空屏幕：

```typescript
const result = await ctx.ui.custom<string | null>((tui, theme, keybindings, done) => {
  const container = new Container();
  container.addChild(new DynamicBorder((s: string) => theme.fg("accent", s)));
  container.addChild(new Text("选择一个选项", 1, 0));
  // ... 更多组件 ...
  return container;
}, { overlay: true });
```

有关更多详细信息，请参见 [tui.md](tui.md)。

## 错误处理

扩展应该优雅地处理错误。使用 `try/catch` 并通过 `ctx.ui.notify()` 通知用户。

```typescript
pi.on("tool_call", async (event, ctx) => {
  try {
    // 你的代码
  } catch (error) {
    ctx.ui.notify(`错误：${error instanceof Error ? error.message : String(error)}`, "error");
  }
});
```

## 模式行为

Pi 在不同模式下运行，扩展的行为可能会有所不同：

- **交互模式：** 完整的 TUI 和 UI 方法可用
- **打印模式：** 无 UI，`ctx.hasUI` 为 `false`
- **JSON 模式：** 无 UI，`ctx.hasUI` 为 `false`，通过 JSON 事件通信
- **RPC 模式：** 受限 UI，某些方法通过 RPC 子协议工作

检查 `ctx.hasUI` 以确定 UI 方法是否可用。

## 示例参考

查看 [examples/extensions/](../examples/extensions/) 目录以获取更多示例：

- `summarize.ts` - 对话摘要
- `snake.ts` - 等待时的蛇形游戏
- `todo.ts` - 待办列表工具，带有自定义渲染
- `preset.ts` - 选择列表 UI 组件
- `qna.ts` - 带有取消功能的异步操作
- `tools.ts` - 设置切换
- `plan-mode.ts` - 状态指示器和小部件
- `working-indicator.ts` - 自定义工作指示器
- `custom-footer.ts` - 自定义页脚
- `modal-editor.ts` - 自定义编辑器（Vim 模式）
- `input-transform.ts` - 输入转换
- `dynamic-tools.ts` - 动态工具注册
- `overlay-qa-tests.ts` - 覆盖层测试
