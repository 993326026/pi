# 会话文件格式

会话存储为 JSONL（JSON 行）文件。每行都是带有 `type` 字段的 JSON 对象。会话条目通过 `id`/`parentId` 字段形成树结构，支持原地分支而无需创建新文件。

## 文件位置

```
~/.pi/agent/sessions/--<path>--/<timestamp>_<uuid>.jsonl
```

其中 `<path>` 是工作目录，`/` 被替换为 `-`。

## 删除会话

可以通过删除 `~/.pi/agent/sessions/` 下的 `.jsonl` 文件来删除会话。

Pi 还支持通过 `/resume` 交互删除（选择会话并按 `Ctrl+D`，然后确认）。如果可用，Pi 使用 `trash` CLI 来避免永久删除。

## 会话版本

会话在标题中具有版本字段：

- **版本 1**：线性条目序列（旧版，加载时自动迁移）
- **版本 2**：具有 `id`/`parentId` 链接的树结构
- **版本 3**：将 `hookMessage` 角色重命名为 `custom`（扩展统一）

加载时，现有会话会自动迁移到当前版本（v3）。

## 源代码

GitHub 上的源代码 ([pi-mono](https://github.com/earendil-works/pi-mono)：
- [`packages/coding-agent/src/core/session-manager.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/session-manager.ts) - 会话条目类型和 SessionManager
- [`packages/coding-agent/src/core/messages.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/messages.ts) - 扩展消息类型（BashExecutionMessage、CustomMessage 等）
- [`packages/ai/src/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/types.ts) - 基础消息类型（UserMessage、AssistantMessage、ToolResultMessage）
- [`packages/agent/src/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/agent/src/types.ts) - AgentMessage 联合类型

对于项目中的 TypeScript 定义，请检查 `node_modules/@earendil-works/pi-coding-agent/dist/` 和 `node_modules/@earendil-works/pi-ai/dist/`。

## 消息类型

会话条目包含 `AgentMessage` 对象。理解这些类型对于解析会话和编写扩展至关重要。

### 内容块

消息包含类型化内容块数组：

```typescript
interface TextContent {
  type: "text";
  text: string;
}

interface ImageContent {
  type: "image";
  data: string;      // base64 编码
  mimeType: string;  // 例如 "image/jpeg"、"image/png"
}

interface ThinkingContent {
  type: "thinking";
  thinking: string;
}

interface ToolCall {
  type: "toolCall";
  id: string;
  name: string;
  arguments: Record<string, any>;
}
```

### 基础消息类型（来自 pi-ai）

```typescript
interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number;  // Unix 毫秒
}

interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: string;
  provider: string;
  model: string;
  usage: Usage;
  stopReason: "stop" | "length" | "toolUse" | "error" | "aborted";
  errorMessage?: string;
  timestamp: number;
}

interface ToolResultMessage {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];
  details?: any;      // 工具特定的元数据
  isError: boolean;
  timestamp: number;
}

interface Usage {
  input: number;
  output: number;
  cacheRead: number;
  cacheWrite: number;
  totalTokens: number;
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
    total: number;
  };
}
```

### 扩展消息类型（来自 pi-coding-agent）

```typescript
interface BashExecutionMessage {
  role: "bashExecution";
  command: string;
  output: string;
  exitCode: number | undefined;
  cancelled: boolean;
  truncated: boolean;
  fullOutputPath?: string;
  excludeFromContext?: boolean;  // !! 前缀命令为 true
  timestamp: number;
}

interface CustomMessage {
  role: "custom";
  customType: string;            // 扩展标识符
  content: string | (TextContent | ImageContent)[];
  display: boolean;            // 在 TUI 中显示
  details?: any;               // 扩展特定元数据
  timestamp: number;
}

interface BranchSummaryMessage {
  role: "branchSummary";
  summary: string;
  fromId: string;              // 我们分支的条目
  timestamp: number;
}

interface CompactionSummaryMessage {
  role: "compactionSummary";
  summary: string;
  tokensBefore: number;
  timestamp: number;
}
```

### AgentMessage 联合

```typescript
type AgentMessage =
  | UserMessage
  | AssistantMessage
  | ToolResultMessage
  | BashExecutionMessage
  | CustomMessage
  | BranchSummaryMessage
  | CompactionSummaryMessage;
```

## 条目基础

所有条目（除了 `SessionHeader`）都扩展 `SessionEntryBase`：

```typescript
interface SessionEntryBase {
  type: string;
  id: string;           // 8 字符十六进制 ID
  parentId: string | null;  // 父条目 ID（第一个条目为 null）
  timestamp: string;    // ISO 时间戳
}
```

## 条目类型

### SessionHeader

文件的第一行。仅元数据，不是树的一部分（没有 `id`/`parentId`）。

```json
{"type":"session","version":3,"id":"uuid","timestamp":"2024-12-03T14:00:00.000Z","cwd":"/path/to/project"}
```

对于具有父级的会话（通过 `/fork`、`/clone` 或 `newSession({ parentSession })` 创建）：

```json
{"type":"session","version":3,"id":"uuid","timestamp":"2024-12-03T14:00:00.000Z","cwd":"/path/to/project","parentSession":"/path/to/original/session.jsonl"}
```

### SessionMessageEntry

对话中的消息。`message` 字段包含 `AgentMessage`。

```json
{"type":"message","id":"a1b2c3d4","parentId":"prev1234","timestamp":"2024-12-03T14:00:01.000Z","message":{"role":"user","content":"你好"}}
{"type":"message","id":"b2c3d4e5","parentId":"a1b2c3d4","timestamp":"2024-12-03T14:00:02.000Z","message":{"role":"assistant","content":[{"type":"text","text":"嗨！"}],"provider":"anthropic","model":"claude-3-5-sonnet-4-20250514","usage":{...},"stopReason":"stop"}}
{"type":"message","id":"c3d4e5f6","parentId":"b2c3d4e5","timestamp":"2024-12-03T14:00:03.000Z","message":{"role":"toolResult","toolCallId":"call_123","toolName":"bash","content":[{"type":"text","text":"输出"}],"isError":false}}
```

### ModelChangeEntry

在会话中用户切换模型时发出。

```json
{"type":"model_change","id":"d4e5f6g7","parentId":"c3d4e5f6","timestamp":"2024-12-03T14:05:00.000Z","provider":"openai","modelId":"gpt-4o"}
```

### ThinkingLevelChangeEntry

在用户更改思考/推理级别时发出。

```json
{"type":"thinking_level_change","id":"e5f6g7h8","parentId":"d4e5f6g7","timestamp":"2024-12-03T14:06:00.000Z","thinkingLevel":"high"}
```

### CompactionEntry

在上下文被压缩时创建。存储早期消息的摘要。

```json
{"type":"compaction","id":"f6g7h8i9","parentId":"e5f6g7h8","timestamp":"2024-12-03T14:10:00.000Z","summary":"用户讨论了 X、Y、Z...","firstKeptEntryId":"c3d4e5f6","tokensBefore":50000}
```

可选字段：
- `details`：实现特定数据（例如默认情况下为 `{ readFiles: string[], modifiedFiles: string[] }`，或扩展的自定义数据）
- `fromHook`：如果由扩展生成则为 `true`，如果由 pi 生成则为 `false`/`undefined`（旧字段名）

### BranchSummaryEntry

在通过 `/tree` 切换分支时创建，包含直到共同祖先的离开分支的 LLM 生成摘要。捕获来自废弃路径的上下文。

```json
{"type":"branch_summary","id":"g7h8i9j0","parentId":"a1b2c3d4","timestamp":"2024-12-03T14:15:00.000Z","fromId":"f6g7h8i9","summary":"分支探索了方法 A..."}
```

可选字段：
- `details`：文件跟踪数据（默认情况下为 `{ readFiles: string[], modifiedFiles: string[] }`，或扩展的自定义数据）
- `fromHook`：如果由扩展生成则为 `true`，如果由 pi 生成则为 `false`/`undefined`（旧字段名）

### CustomEntry

扩展状态持久化。**不参与** LLM 上下文。

```json
{"type":"custom","id":"h8i9j0k1","parentId":"g7h8i9j0","timestamp":"2024-12-03T14:20:00.000Z","customType":"my-extension","data":{"count":42}}
```

使用 `customType` 在重新加载时识别你扩展的条目。

### CustomMessageEntry

扩展注入的消息，**确实参与** LLM 上下文。

```json
{"type":"custom_message","id":"i9j0k1l2","parentId":"h8i9j0k1","timestamp":"2024-12-03T14:25:00.000Z","customType":"my-extension","content":"注入的上下文...","display":true}
```

字段：
- `content`：字符串或 `(TextContent | ImageContent)[]`（与 UserMessage 相同）
- `display`：`true` = 在 TUI 中以不同样式显示，`false` = 隐藏
- `details`：可选的扩展特定元数据（不发送到 LLM）

### LabelEntry

用户定义的条目标签/标记。

```json
{"type":"label","id":"j0k1l2m3","parentId":"i9j0k1l2","timestamp":"2024-12-03T14:30:00.000Z","targetId":"a1b2c3d4","label":"checkpoint-1"}
```

将 `label` 设置为 `undefined` 以清除标签。

### SessionInfoEntry

会话元数据（例如用户定义的显示名称）。通过 `/name` 命令或扩展中的 `pi.setSessionName()` 设置。

```json
{"type":"session_info","id":"k1l2m3n4","parentId":"j0k1l2m3","timestamp":"2024-12-03T14:35:00.000Z","name":"重构身份验证模块"}
```

设置后，会话名称会显示在会话选择器（`/resume`）中，而不是第一条消息。

## 树结构

条目形成树：
- 第一个条目的 `parentId: null`
- 每个后续条目通过 `parentId` 指向其父级
- 分支从早期条目创建新子级
- “叶子”是树中的当前位置

```
[用户消息] ─── [助手] ─── [用户消息] ─── [助手] ─┬─ [用户消息] ← 当前叶子
                                                       │
                                                       └─ [分支摘要] ─── [用户消息] ← 备用分支
```

## 上下文构建

`buildSessionContext()` 从当前叶子遍历到根，为 LLM 生成消息列表：

1. 收集路径上的所有条目
2. 提取当前模型和思考级别设置
3. 如果路径上有 `CompactionEntry`：
   - 首先发出摘要
   - 然后从 `firstKeptEntryId` 到压缩的消息
   - 然后是压缩后的消息
4. 将 `BranchSummaryEntry` 和 `CustomMessageEntry` 转换为适当的消息格式

## 解析示例

```typescript
import { readFileSync } from "fs";

const lines = readFileSync("session.jsonl", "utf8").trim().split("\n");

for (const line of lines) {
  const entry = JSON.parse(line);

  switch (entry.type) {
    case "session":
      console.log(`会话 v${entry.version ?? 1}：${entry.id}`);
      break;
    case "message":
      console.log(`[${entry.id}] ${entry.message.role}：${JSON.stringify(entry.message.content)}`);
      break;
    case "compaction":
      console.log(`[${entry.id}] 压缩：${entry.tokensBefore} 个令牌已摘要`);
      break;
    case "branch_summary":
      console.log(`[${entry.id}] 从 ${entry.fromId} 分支`);
      break;
    case "custom":
      console.log(`[${entry.id}] 自定义（${entry.customType}）：${JSON.stringify(entry.data)}`);
      break;
    case "custom_message":
      console.log(`[${entry.id}] 扩展消息（${entry.customType}）：${entry.content}`);
      break;
    case "label":
      console.log(`[${entry.id}] ${entry.targetId} 上的标签 "${entry.label}"`);
      break;
    case "model_change":
      console.log(`[${entry.id}] 模型：${entry.provider}/${entry.modelId}`);
      break;
    case "thinking_level_change":
      console.log(`[${entry.id}] 思考：${entry.thinkingLevel}`);
      break;
  }
}
```

## SessionManager API

用于以编程方式处理会话的关键方法。

### 静态创建方法
- `SessionManager.create(cwd, sessionDir?)` - 新会话
- `SessionManager.open(path, sessionDir?)` - 打开现有会话文件
- `SessionManager.continueRecent(cwd, sessionDir?)` - 继续最近的或创建新的
- `SessionManager.inMemory(cwd?)` - 无文件持久化
- `SessionManager.forkFrom(sourcePath, targetCwd, sessionDir?)` - 从另一个项目分支会话

### 静态列表方法
- `SessionManager.list(cwd, sessionDir?, onProgress?)` - 列出目录的会话
- `SessionManager.listAll(onProgress?)` - 列出所有项目的所有会话

### 实例方法 - 会话管理
- `newSession(options?)` - 启动新会话（选项：`{ parentSession?: string }`）
- `setSessionFile(path)` - 切换到不同的会话文件
- `createBranchedSession(leafId)` - 将分支提取到新会话文件

### 实例方法 - 追加（所有返回条目 ID）
- `appendMessage(message)` - 添加消息
- `appendThinkingLevelChange(level)` - 记录思考更改
- `appendModelChange(provider, modelId)` - 记录模型更改
- `appendCompaction(summary, firstKeptEntryId, tokensBefore, details?, fromHook?)` - 添加压缩
- `appendCustomEntry(customType, data?)` - 扩展状态（不在上下文中）
- `appendSessionInfo(name)` - 设置会话显示名称
- `appendCustomMessageEntry(customType, content, display, details?)` - 扩展消息（在上下文中）
- `appendLabelChange(targetId, label)` - 设置/清除标签

### 实例方法 - 树导航
- `getLeafId()` - 当前位置
- `getLeafEntry()` - 获取当前叶子条目
- `getEntry(id)` - 按 ID 获取条目
- `getBranch(fromId?)` - 从条目遍历到根
- `getTree()` - 获取完整树结构
- `getChildren(parentId)` - 获取直接子项
- `getLabel(id)` - 获取条目的标签
- `branch(entryId)` - 将叶子移动到早期条目
- `resetLeaf()` - 将叶子重置为 null（在任何条目之前）
- `branchWithSummary(entryId, summary, details?, fromHook?)` - 使用上下文摘要分支

### 实例方法 - 上下文和信息
- `buildSessionContext()` - 获取 LLM 的消息、thinkingLevel 和模型
- `getEntries()` - 所有条目（不包括标题）
- `getHeader()` - 会话标题元数据
- `getSessionName()` - 从最新的 session_info 条目获取显示名称
- `getCwd()` - 工作目录
- `getSessionDir()` - 会话存储目录
- `getSessionId()` - 会话 UUID
- `getSessionFile()` - 会话文件路径（内存中为 undefined）
- `isPersisted()` - 会话是否保存到磁盘
