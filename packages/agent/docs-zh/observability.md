<!-- 从 jot qe0ikdqs 同步。在仓库中编辑此文件以继续。 -->

# Pi 可观察性设计说明

## 目标

使 `packages/ai` 和 `packages/agent`/harness 可观察，而不依赖于 OpenTelemetry、Sentry 或任何 APM 供应商。

Pi 应该发出稳定的、结构化的生命周期事件。外部监听器可以将这些事件转换为 OTel 跨度、Sentry 跨度、日志、指标或自定义遥测。

## 心智模型

一个 trace 是工作的一个因果树，例如一个用户回合。

一个 span 是该树中的一个定时操作。它通常由 ID 表示，而不是对象指针：

```ts
interface SpanRecord {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  name: string;
  startTime: number;
  endTime?: number;
  attributes: Record<string, unknown>;
  status: "ok" | "error";
}
```

示例树：

```text
traceId=t1 spanId=s1 parent=-  name=pi.agent.prompt
traceId=t1 spanId=s2 parent=s1 name=pi.agent.turn
traceId=t1 spanId=s3 parent=s2 name=pi.ai.provider.request
traceId=t1 spanId=s4 parent=s2 name=pi.agent.tool_call
traceId=t1 spanId=s5 parent=s4 name=pi.session.append_entry
```

## 异步上下文

JavaScript 有一个事件循环，但多个异步链可以交错。单个全局 `currentContext` 在并发下会中断。

`AsyncLocalStorage` 是 Node 中用于异步延续的 `ThreadLocal` 等效物。它允许并发操作保持不同的当前上下文：

```ts
await Promise.all([
  runWithPiContext({ userId: "alice" }, () => harness.prompt("A")),
  runWithPiContext({ userId: "bob" }, () => harness.prompt("B")),
]);
```

深层代码然后可以为活动异步链读取正确的当前上下文。

Pi 必须在 Node、Bun、浏览器、worker 和其他 JS 运行时中运行，因此 ALS 不能是核心抽象。它应该是一个运行时适配器。

## 核心设计

Pi 拥有一个小的运行时中性可观察性抽象：

```ts
export interface PiObservabilityContext {
  traceId?: string;
  currentSpanId?: string;
  userContext?: Record<string, unknown>;
}

export interface PiObservabilityEvent {
  type: "start" | "end" | "error" | "event";
  name: string;
  traceId: string;
  spanId?: string;
  parentSpanId?: string;
  timestamp: number;
  durationMs?: number;
  context?: Record<string, unknown>;
  payload?: Record<string, unknown>;
  error?: { name: string; message: string };
}

export interface PiObservability {
  getContext(): PiObservabilityContext | undefined;
  runWithContext<T>(context: PiObservabilityContext, fn: () => T): T;
  emit(event: PiObservabilityEvent): void;
  hasSubscribers(): boolean;
}
```

公共 API：

```ts
export function configurePiObservability(observability: PiObservability): void;
export function subscribePiObservability(listener: (event: PiObservabilityEvent) => void): () => void;
export function runWithPiContext<T>(userContext: Record<string, unknown>, fn: () => T): T;
export function traceOperation<T>(name: string, payload: Record<string, unknown>, fn: () => T): T;
```

`traceOperation()`：

1. 读取当前上下文
2. 如果缺失则创建 `traceId`
3. 创建新的 `spanId`
4. 使用当前跨度作为 `parentSpanId`
5. 发出 `start`
6. 在子上下文中运行回调
7. 发出 `end` 或 `error`
8. 错误时重新抛出

伪代码：

```ts
function traceOperation<T>(name: string, payload: Record<string, unknown>, fn: () => T): T {
  const parent = getContext();
  const traceId = parent?.traceId ?? createId();
  const spanId = createId();
  const parentSpanId = parent?.currentSpanId;

  const child = { ...parent, traceId, currentSpanId: spanId };

  emit({ type: "start", name, traceId, spanId, parentSpanId, timestamp: Date.now(), context: parent?.userContext, payload });

  return runWithContext(child, () => {
    try {
      const result = fn();
      // 承诺感知的实现在结算后发出 end/error。
      emit({ type: "end", name, traceId, spanId, parentSpanId, timestamp: Date.now(), context: child.userContext, payload });
      return result;
    } catch (error) {
      emit({ type: "error", name, traceId, spanId, parentSpanId, timestamp: Date.now(), context: child.userContext, payload, error: serializeError(error) });
      throw error;
    }
  });
}
```

## 运行时适配器

核心包不应导入仅 Node 的 API。

可能的实现：

- Node 适配器：`AsyncLocalStorage` 用于上下文，可选的 `diagnostics_channel` 发布。
- 浏览器/worker 回退：本地订阅者集和有限/手动上下文传播。
- Bun/Deno 适配器：如果可用，使用运行时特定的异步上下文。

对于 Node，诊断通道可以用作被动事件总线：

```ts
import { channel } from "diagnostics_channel";
channel("pi.observability").publish(event);
```

订阅者可以创建 OTel/Sentry 跨度而无需猴子补丁 pi。

## Pi 发出什么

Pi 发出发生了什么。它不直接创建 OTel/Sentry 跨度。

初始最小事件名称：

```text
pi.agent.prompt
pi.agent.skill
pi.agent.prompt_template
pi.agent.compaction
pi.agent.branch_navigation
pi.agent.session.append_entry
pi.ai.provider.request
```

每个操作发出：

```text
start
end
error
```

稍后的添加：

```text
pi.agent.turn
pi.agent.tool_call
pi.agent.queue_update
pi.ai.provider.retry
pi.ai.provider.first_token
pi.ai.provider.usage
pi.session.read
pi.session.write
```

## 最小插装点

### packages/agent

包装：

- `AgentHarness.prompt()`
- `AgentHarness.skill()`
- `AgentHarness.promptFromTemplate()`
- `AgentHarness.compact()`
- `AgentHarness.navigateTree()`
- `Session.appendTypedEntry()` 或存储追加外观

示例：

```ts
return traceOperation(
  "pi.agent.prompt",
  {
    sessionId: turnState.sessionId,
    provider: turnState.model.provider,
    model: turnState.model.id,
    promptLength: text.length,
    imageCount: options?.images?.length ?? 0,
  },
  () => this.executeTurn(turnState, text, options),
);
```

会话写入：

```ts
return traceOperation(
  "pi.agent.session.append_entry",
  { entryType: entry.type },
  async () => {
    await this.unwrap(this.storage.appendEntry(entry));
    return entry.id;
  },
);
```

### packages/ai

包装常见的提供者边界：

- `streamSimple()`
- `completeSimple()`

示例：

```ts
return traceOperation(
  "pi.ai.provider.request",
  {
    api: model.api,
    provider: model.provider,
    model: model.id,
    sessionId: options.sessionId,
    reasoning: options.reasoning,
  },
  () => actualStreamSimple(model, context, options),
);
```

结束/错误负载可以包含安全的元数据：

- 停止原因
- 状态码
- 重试计数
- 输入/输出/总令牌
- 总成本
- 中止/超时标志

## 安全性和编辑

默认负载必须是安全的。

默认安全：

- 提供者
- 模型
- API 标识符
- 会话 ID
- 条目类型
- 工具名称
- 状态码
- 停止原因
- 令牌计数
- 成本
- 持续时间

默认不安全：

- 提示
- 完成
- 工具参数
- 工具结果
- 外壳输出
- 文件内容
- 提供者请求负载
- 提供者响应主体
- API 密钥
- 标头

内容捕获可以稍后通过明确的编辑钩子进行选择加入。

## 监听器行为

可观察性绝不能影响 pi 执行。

订阅者错误应该被吞下或隔离。Harness 钩子是控制平面并且可能影响执行；可观察性订阅者是被动的并且绝不能。

## 用户上下文

用户可以将任意上下文与一个回合关联：

```ts
await runWithPiContext(
  {
    userId: "u123",
    orgId: "acme",
    region: "eu",
  },
  () => harness.prompt("fix this"),
);
```

该异步链内的每个发出的事件都包含该上下文：

```ts
{
  type: "start",
  name: "pi.ai.provider.request",
  traceId: "t1",
  spanId: "s3",
  parentSpanId: "s1",
  context: {
    userId: "u123",
    orgId: "acme",
    region: "eu",
  },
  payload: {
    provider: "anthropic",
    model: "claude-sonnet-4",
  },
}
```

一个 OTel 适配器可以将其映射到跨度属性。一个 Sentry 适配器可以将其映射到 Sentry 上下文/跨度。自定义用户可以记录 JSON。

## 包故事

最小初始包：

```text
packages/observability
  runtime-agnostic context + traceOperation + subscribe
```

然后：

```text
packages/ai
  emits pi.ai.* events

packages/agent
  emits pi.agent.* / pi.session.* events
```

可选稍后：

```text
packages/observability-node
  AsyncLocalStorage + diagnostics_channel bridge

packages/otel
  subscribes to pi events and creates OpenTelemetry spans
```

## 论点

Pi 定义了一个稳定的、安全的事件契约。适配器定义事件去哪里。

这使得 ai/harness 可观察，而无需将核心包绑定到 OTel、Sentry、仅 Node API 或猴子补丁。
