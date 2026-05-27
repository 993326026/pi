# JSON 事件流模式

```bash
pi --mode json "你的提示"
```

将所有会话事件作为 JSON 行输出到 stdout。用于将 pi 集成到其他工具或自定义 UI 中。

## 事件类型

事件在 [`AgentSessionEvent`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/agent-session.ts#L102) 中定义：

```typescript
type AgentSessionEvent =
  | AgentEvent
  | { type: "queue_update"; steering: readonly string[]; followUp: readonly string[] }
  | { type: "compaction_start"; reason: "manual" | "threshold" | "overflow" }
  | { type: "compaction_end"; reason: "manual" | "threshold" | "overflow"; result: CompactionResult | undefined; aborted: boolean; willRetry: boolean; errorMessage?: string }
  | { type: "auto_retry_start"; attempt: number; maxAttempts: number; delayMs: number; errorMessage: string }
  | { type: "auto_retry_end"; success: boolean; attempt: number; finalError?: string };
```

`queue_update` 在队列更改时发出完整的待处理转向和后续队列。`compaction_start` 和 `compaction_end` 涵盖手动和自动压缩。

来自 [`AgentEvent`](https://github.com/earendil-works/pi-mono/blob/main/packages/agent/src/types.ts#L179) 的基础事件：

```typescript
type AgentEvent =
  // 代理生命周期
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  // 轮次生命周期
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  // 消息生命周期
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  // 工具执行
  | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
  | { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
  | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

## 消息类型

来自 [`packages/ai/src/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/types.ts#L134) 的基础消息：
- `UserMessage`（第 134 行）
- `AssistantMessage`（第 140 行）
- `ToolResultMessage`（第 152 行）

来自 [`packages/coding-agent/src/core/messages.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/messages.ts#L29) 的扩展消息：
- `BashExecutionMessage`（第 29 行）
- `CustomMessage`（第 46 行）
- `BranchSummaryMessage`（第 55 行）
- `CompactionSummaryMessage`（第 62 行）

## 输出格式

每行都是一个 JSON 对象。第一行是会话标题：

```json
{"type":"session","version":3,"id":"uuid","timestamp":"...","cwd":"/path"}
```

随后是发生的事件：

```json
{"type":"agent_start"}
{"type":"turn_start"}
{"type":"message_start","message":{"role":"assistant","content":[],...}}
{"type":"message_update","message":{...},"assistantMessageEvent":{"type":"text_delta","delta":"你好",...}}
{"type":"message_end","message":{...}}
{"type":"turn_end","message":{...},"toolResults":[]}
{"type":"agent_end","messages":[...]}
```

## 示例

```bash
pi --mode json "列出文件" 2>/dev/null | jq -c 'select(.type == "message_end")'
```
