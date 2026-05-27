# AgentHarness 生命周期

`AgentHarness` 是位于底层 agent 循环之上的编排层。它负责会话持久化、运行时配置、资源解析、操作锁定和面向扩展的变更语义。

本文档描述当前方向和已实现的行为。一些扩展/会话外观细节是计划中的，并已明确指出。

## 最终生命周期目标

Harness 监听器和钩子应该能够在 `AgentHarness` 实例上关闭，并从任何允许调用这些 API 的事件中调用公共 harness API。这些调用不得破坏正在进行的回合快照、重新排列持久化的对话历史条目、丢失待处理的写入、死锁结算或使 harness 处于错误阶段。

预期规则是：

- 结构操作在忙碌时仍保持被拒绝
- 队列操作在记录的回合安全点被接受
- 运行时配置设置更新未来的快照而不改变当前的提供者请求
- 忙碌期间进行的会话写入会被持久化排队并以确定的顺序刷新
- getter 返回最新的 harness 配置，而不是正在进行的快照
- 监听器/钩子目前不接收外观；如果它们在原始 harness 上关闭并在活动运行期间调用 `waitForIdle()` 等结算 API，可能会导致死锁。未来的外观应该改为公开 `runWhenIdle()`。

最终的生命周期强化应该通过广泛的监听器/钩子重入测试套件来证明这些保证。

## 错误处理

当前的划分是：

- 底层能力和助手使用 `Result<TValue, TError>`，其中预期的失败被包含且不得抛出，例如 `ExecutionEnv`、文件系统/外壳操作、外壳输出捕获、资源加载和压缩助手
- 高级变更/编排 API 如 `Session` 和 `AgentHarness` 拒绝/抛出，而不是返回可以被忽略的裸结果
- 公共 `AgentHarness` 失败在实际情况下被标准化为 `AgentHarnessError`；子系统错误被保留为 cause

Harness 事件观察已提交的状态。公共变更函数在实际情况下在提交前验证所需的输入和持久化，然后等待通知。如果提交后钩子或订阅者失败，状态变更不会回滚，公共方法会以 `AgentHarnessError` 代码 `"hook"` 拒绝。

## 状态模型

Harness 将状态分为四类。

### Harness 配置

Harness 配置是应用程序或扩展设置的最新运行时配置：

- model
- thinking level
- tools
- active tool names
- resources
- stream options
- system prompt 或 system prompt provider

Getter 返回 harness 配置。它们不返回正在进行的提供者请求所使用的快照。

Setter 立即更新 harness 配置，即使在回合正在进行时。变更会影响下一个回合快照，而不是当前运行的提供者请求。

`setResources()` 接受具体资源，并在每次调用时发出 `resources_update` 并附带浅拷贝的当前和之前资源。应用程序负责从磁盘或其他来源加载/重新加载资源，并应使用新值调用 `setResources()`。

`getResources()` 返回浅拷贝的当前资源。这是一个实时配置读取，而不是上一个回合快照。

### 回合快照

回合快照是用于一个 LLM 回合的具体状态。它由 `createTurnState()` 创建并包含：

- 持久化的会话消息
- 解析的资源
- 解析的 system prompt
- model
- thinking level
- all tools
- active tools
- stream options
- 派生的 session id

静态选项值被直接使用。System-prompt 提供者回调在每次 `createTurnState()` 调用时被调用一次。该回合的所有逻辑都使用相同的快照。

创建快照时，资源数组被浅拷贝。单独的技能和提示模板对象没有被深拷贝。

创建快照时，流选项被浅拷贝。`headers` 和 `metadata` 映射被浅拷贝；它们的值没有被深拷贝。`getApiKeyAndHeaders()` 的凭据根据每个提供者请求进行解析，以便过期的令牌可以刷新，但配置的流选项和派生的会话 id 来自当前回合快照。

### 会话

会话仅包含持久化条目。会话读取返回持久化状态，不包括待处理的写入。

会话存储实现必须将叶子变更持久化为 `leaf` 条目。`setLeafId()` 不是仅内存中的游标更新；它附加一个持久化条目，其 `targetId` 是活动树叶子或根的 null。重新打开存储必须从最新的持久化叶子影响条目中重建当前叶子。

### 待处理的会话写入

在操作活动期间请求的会话写入被排队为待处理会话写入。待处理写入基于会话条目形状，没有生成的字段（`id`、`parentId`、`timestamp`）。

待处理的会话写入始终被持久化。它们在保存点、操作结算和失败清理时被刷新。

计划了公共的待处理写入/会话外观 API，但尚未实现。

## 操作阶段

Harness 有一个明确的阶段：

```ts
type AgentHarnessPhase = "idle" | "turn" | "compaction" | "branch_summary" | "retry";
```

结构操作需要 `phase === "idle"` 并在第一个 `await` 之前同步设置阶段：

- `prompt`
- `skill`
- `promptFromTemplate`
- `compact`
- `navigateTree`

在 harness 不空闲时启动另一个结构操作会以 `AgentHarnessError` 代码 `"busy"` 拒绝。

在适当的回合期间允许以下操作：

- `steer`
- `followUp`
- `nextTurn`
- `abort`
- 运行时配置设置

阶段/结算语义仍是临时的，需要完整的生命周期检查。

## 回合执行

`prompt`、`skill` 和 `promptFromTemplate` 遵循相同的流程：

1. 断言空闲并将阶段设置为 `"turn"`。
2. 使用 `createTurnState()` 创建回合快照。
3. 从该快照派生调用文本。
4. 使用 `executeTurn()` 执行回合。

`skill` 和 `promptFromTemplate` 从传递给回合的同一快照解析它们的资源。它们不单独解析资源。

`steer`、`followUp` 和 `nextTurn` 接受文本加上可选图像并在内部创建用户消息。`nextTurn` 消息在下一个用户发起的回合的新用户消息之前插入。

队列模式是实时的，而不是回合快照的：

- `getSteeringMode()` / `setSteeringMode()`
- `getFollowUpMode()` / `setFollowUpMode()`

在运行期间更改队列模式会影响下一个队列排空。队列排空发生在安全点。

## 保存点

保存点发生在助手回合及其工具结果消息完成之后。

在保存点，harness 会：

1. 在该回合的 agent 发出的消息之后刷新待处理的会话写入
2. 如果底层循环可能继续，则创建一个新的回合快照
3. 在下一个提供者请求之前应用新的上下文/模型/思考级别/流选项/会话 id 状态

这使得在回合期间进行的模型、思考级别、工具、资源、流选项和系统提示变更在同一运行中影响下一个回合，同时永远不会改变正在进行的提供者请求。保存点不会重新创建循环回调。

底层循环在提供者边界将 harness `ThinkingLevel` 转换为提供者 `reasoning`：

- `"off"` → `undefined`
- 所有其他思考级别都通过传递

除了刷新剩余的待处理会话写入和清除操作阶段外，`agent_end` 上不需要状态刷新。确切的 `settled` 事件时间仍在审查中。

如果在启动 `prompt`、`skill` 或 `promptFromTemplate` 时系统提示回调抛出，操作会以 `AgentHarnessError` 拒绝，harness 返回到空闲状态。如果从 `prepareNextTurn` 创建的保存点快照抛出，底层 agent 运行会记录一个助手错误消息。

## 钩子和事件

目标钩子系统在 [hooks.md](./hooks.md) 中描述。

总结：

- `AgentHarness` 发出类型化的钩子事件并消费类型化的结果。
- 单个钩子实现拥有注册、清理、来源和结果归约器。
- 观察和变更钩子使用一个事件特定的 `on()` API；事件结果类型决定处理程序是否可以返回结果。
- 产生结果的事件由类型化归约器表归约；应用程序特定的钩子仅为应用程序特定的产生结果事件添加归约器。
- 钩子注册来源是注册上的附带元数据。资源和工具来源属于应用程序特定的具体值类型。
- 钩子上下文应该是外观的普通对象，而不是原始内部或延迟绑定的 getter 迷宫。

事件负载描述发生了什么。Harness getter 描述未来快照的最新配置。

## 计划中的会话外观

扩展最终应该与 harness 范围的 `HarnessSession` 外观交互，而不是原始会话。外观应该包装内部会话并强制执行 harness 待处理写入顺序语义。一旦存在，钩子和事件监听器可以接收一个上下文，该上下文公开完整的 `AgentHarness` 加上会话外观，而不直接访问无序的原始会话写入。

计划中的读取语义：

- 读取委托给持久化的会话状态
- 读取不包括排队的待处理写入

计划中的写入语义：

- 空闲：立即持久化
- 忙碌：作为待处理会话写入排队

计划中的诊断 API 可能明确公开待处理写入：

```ts
getPendingWrites(): readonly PendingSessionWrite[]
```

Agent 发出的消息在 `message_end` 上持久化以保持对话历史顺序。待处理的扩展/会话写入在保存点的这些消息之后刷新。

## 中止

在回合期间允许中止。它中止底层运行并清除引导/跟进队列。

中止不会清除 `nextTurn` 消息。使用 `nextTurn()` 排队的消息在中止后继续存在，并在下一个用户发起的回合的用户消息之前插入。

中止不会丢弃待处理的会话写入。如果到达下一个保存点、在 `agent_end` 上或在操作失败清理中，待处理写入会被刷新。

中止障碍语义仍需审计。

## 压缩和树导航

压缩和树导航是结构会话变更。

它们仅在空闲时被允许，并且不会被排队。它们对持久化的会话状态进行操作。下一个提示会创建一个新的回合快照。

分支摘要生成是树导航操作的一部分。

自动压缩和重试决策点尚未在 `AgentHarness` 中实现。

## 测试组织

Harness 测试应该按领域保持重点，而不是成长为一个大的包罗万象的文件。

当前结构：

- `packages/agent/test/harness/agent-harness.test.ts`：核心生命周期和公共 API 行为。
- `packages/agent/test/harness/agent-harness-stream.test.ts`：流选项和提供者钩子语义。

首选的未来结构：

- `agent-harness-resources.test.ts`：资源快照/加载语义。
- `agent-harness-tools.test.ts`：工具注册表 getter、活动工具语义和更新事件。
- `agent-harness-lifecycle.test.ts`：阶段/保存点/结算/重入行为。

使用 `pi-ai` 伪提供者（`registerFauxProvider`、`fauxAssistantMessage`）进行确定性的 harness/提供者测试。伪响应工厂可以检查 `StreamOptions`、调用 `options.onPayload` 并返回脚本化的助手消息，而无需真实的提供者 API 或网络访问。

Harness 覆盖率与默认包测试运行分开配置：

```bash
npm run test:harness
npm run coverage:harness
```

`coverage:harness` 运行 `test/harness/**/*.test.ts` 并将 `src/harness/**/*.ts` 以及它直接使用的非 harness 运行时文件（`src/agent.ts` 和 `src/agent-loop.ts`）的覆盖率报告到 `coverage/harness`。类型仅依赖项如 `src/types.ts` 不包括在内，因为它们没有有意义的运行时覆盖率。

## 实现待办事项

此列表跟踪在将 `AgentHarness` 视为迁移就绪之前的剩余工作。活跃/计划中的项目从最简单到最难排序。已完成的项目归档在底部。

### 1. 添加明确的工具注册表读取/更新语义

状态：进行中

已完成：

- 添加了 `setTools(tools, activeToolNames?)`。
- 添加了 `setActiveTools(toolNames)`。
- 无效的活动工具名称会以 `AgentHarnessError` 拒绝。
- 通过 `AgentHarness<TSkill, TPromptTemplate, TTool>` 添加了通用应用程序工具形状。
- 从核心类型导出了 `QueueMode`。
- 添加了 `AgentHarnessOptions.steeringMode` 和 `followUpMode`。
- 添加了实时 `getSteeringMode()` / `setSteeringMode()` 和 `getFollowUpMode()` / `setFollowUpMode()`。

剩余：

- 添加 `getTools()` 语义。
- 添加 `getActiveTools()` 语义。
- 决定并实现工具更新可观察性事件。
- 在运行时配置可观察性计划中包含仅活动工具的更新。

备注：

- 可观察性设计：[observability.md](./observability.md)

### 2. 设计每个 `AgentHarness` 模型注册表

状态：计划中

已完成：

- 保留了当前的 `setModel()` 行为。

剩余：

- 决定应用程序如何提供模型注册表。
- 决定 harness 是否存储具体的 `Model` 对象、模型引用或两者都存储。
- 根据注册表验证模型选择。
- 定义活动回合和保存点期间的模型变更语义。

### 3. 完整的 `AgentHarness` 生命周期/状态检查

状态：进行中

已完成：

- 移除了构造函数 `void syncFromTree()`、`syncFromTree()`、`liveOperationId` 和 `shell()`。
- 添加了 `createTurnState()`、`applyTurnState()` 和 `executeTurn()`。
- 添加了明确的 `phase` 来代替布尔空闲状态。
- 保存点刷新上下文、模型、思考级别、流选项和会话快照状态。
- 待处理的会话写入使用没有生成字段的会话条目形状。
- 待处理的会话写入在保存点、结算和失败清理时刷新。
- `steer`、`followUp` 和 `nextTurn` 从文本加上可选图像创建用户消息。
- `nextTurn` 消息在新用户提示之前插入。
- 结构压缩/树操作使用 `finally` 恢复阶段。
- 公共 harness 失败将子系统原因标准化为 `AgentHarnessError`。
- 待处理的会话写入逐一刷新，并且在失败时不会被丢弃。
- 如果队列更新通知失败，队列排空会回滚。
- `message_end` 持久化发生在订阅者通知之前。
- `abort()` 在通知之前信号取消，并仍然通过通知错误等待空闲。
- 空闲模型/思考/工具更新在提交内存中状态之前进行验证和持久化。
- `setLeafId()` 持久化持久化的 `leaf` 条目，以便树导航在存储重新打开后继续存在。

剩余：

- 最终确定阶段/空闲语义。
- 审计 `settled` 是否可能过早触发。
- 使 `settled` 回调内的会话写入具有确定性。
- 审计 `agent_end` 周围的跟进行为。
- 实现自动压缩决策点。
- 实现重试处理。
- 根据 coding-agent 验证 `before_agent_start` 钩子语义。
- 决定 `before_agent_start` 是否需要更多回合信息，例如工具/工具片段。
- 记录或更改忙碌期间的运行时配置事件时间。
- 审计 `abort()` 障碍语义。

### 4. 实现通用钩子/事件扩展机制

状态：在 [hooks.md](./hooks.md) 中设计，未实现

已完成：

- 移除了 `AgentHarnessContext`。
- 钩子仅接收事件负载。
- `emitHook(event)` 从 `event.type` 派生钩子类型。
- 提供者请求/负载钩子具有有序的转换语义。

剩余：

- 添加 `HookEvent`、`ResultOf`、带有通用源元数据的注册选项以及单个 `AgentHarnessHooks` 实现。
- 将结果链从 `AgentHarness` 移出到归约器函数中。
- 类型检查基础 harness 归约器，以便每个产生结果的 `AgentHarnessEvent` 都有归约器语义。
- 使 `AgentHarness` 接受并公开带有应用程序特定钩子的构造函数推断的具体钩子实例。
- 定义通过钩子上下文公开的初始 harness/上下文外观。
- 保留当前的提供者钩子行为，包括流选项补丁删除语义。
- 为归约器语义添加奇偶校验测试：转换链、补丁链、早期阻止/取消、清理、源元数据和类型化应用程序特定的归约器覆盖率。

备注：

- 钩子设计：[hooks.md](./hooks.md)

### 5. 原型半持久化 harness/会话恢复

状态：计划中

已完成：

- 编写了持久性设计：[durable-harness.md](./durable-harness.md)

剩余：

- 决定会话是否拥有所有持久化的 harness 状态，或者是否需要任何附带存储用于大的 blob。
- 为队列、待处理写入、操作、回合、提供者请求和工具调用定义持久化条目。
- 为应用程序提供的工具、模型、扩展、资源、钩子和认证提供者定义恢复要求。
- 为未完成的 agent 回合、提供者请求、工具调用、压缩和树导航定义保守的恢复策略。
- 原型化从会话条目中基于归约器的恢复。
- 决定中断的操作是否附加用户可见的消息或仅内部操作条目。

备注：

- 提供者流不可恢复；恢复应该从持久化边界重新启动或标记操作已中断。
- 未完成的工具调用不安全重试，除非工具声明幂等/重试安全行为。

### 6. 最终生命周期强化套件

状态：计划中

已完成：

- 无。

剩余：

- 在相关事件上添加广泛的监听器/钩子重入测试。
- 测试来自底层生命周期事件和 harness 事件的运行时配置设置。
- 测试模型、思考、资源、工具、活动工具和流选项的运行时配置可观察性。
- 测试在活动回合和保存点期间的资源/工具/模型/思考/流选项更新。
- 测试来自监听器和钩子的会话写入，包括 `settled` 写入。
- 测试来自回合事件、工具事件和提供者钩子的队列操作。
- 测试忙碌期间被拒绝的结构操作。
- 测试来自监听器/钩子的中止。
- 测试活动操作期间的 getter 行为。
- 测试 agent 发出的消息和待处理监听器写入的确定性顺序。
- 测试当异步监听器调用 harness API 并等待它们时没有死锁。
- 测试通过成功、提供者错误、钩子错误、中止、压缩和树导航的阶段清理。

### 7. 后续 coding-agent 迁移计划

状态：计划中

已完成：

- 无。

剩余：

- 将 coding-agent 资源映射到源加载器。
- 将应用程序级别的资源去重/来源保留在 harness 之外。
- 使扩展加载适应未来的钩子/会话外观。
- 将 UI/会话行为保留在核心之外。
- 将 coding-agent 流/认证/重试/标头行为移动到 harness 流配置和提供者钩子上。

---

## 已完成的实现待办事项

### 8. 从 `AgentHarness` 移除 `Agent` 依赖

状态：已完成

已完成：

- `AgentHarness` 直接调用 `runAgentLoop()`。
- Harness 拥有运行生命周期、中止控制器、队列排空、提供者流配置、事件归约、会话持久化、待处理写入刷新和保存点快照。
- Harness 测试覆盖提示构造、队列排空、中止行为、保存点刷新、待处理写入顺序、等待监听器结算、工具钩子和提供者流包装。

剩余：

- 无。

备注：

- 更广泛的监听器/钩子重入覆盖率在第 6 项中跟踪。

### 9. 完成精选的提供者/流配置

状态：已完成

已完成：

- 添加了精选的 `AgentHarnessOptions.streamOptions`、`getStreamOptions()` 和 `setStreamOptions()`。
- 每个回合都会快照流选项、标头、元数据和派生的会话 id。
- Harness 拥有的流包装器调用 `streamSimple()` 并保留来自底层循环的生命周期拥有的 `signal` 和 `reasoning`。
- `getApiKeyAndHeaders()` 根据每个提供者请求解析凭据。
- 实现了 `before_provider_request`、`before_provider_payload` 和 `after_provider_response` 钩子。
- 流选项补丁支持明确的字段删除和有序的钩子链。
- `agent-harness-stream.test.ts` 覆盖转发、认证合并、钩子补丁/删除/链接、负载钩子和忙碌/保存点快照行为。

剩余：

- 无。

### 10. 完成底层 `Result` 清理

状态：已完成

已完成：

- 添加了通用 `Result<TValue, TError>` 加上助手。
- 更新了 `ExecutionEnv` 和 `NodeExecutionEnv` 以返回文件系统/进程操作的类型化结果。
- 拆分了文件系统和外壳能力。
- 将 JSONL 会话存储/仓库移动到文件系统选择上，而不是直接的 Node 导入。
- 添加了 `ExecutionEnv.appendFile()` 用于流式附加用例。
- 更新了技能和提示模板加载器以消费 `ExecutionEnv` 结果。
- 更新了外壳输出捕获以返回结果并使用 `ExecutionEnv`，包括通过 `appendFile()` 的完整输出溢出。
- 从浏览器安全的根导出中移除了 `NodeExecutionEnv`。
- 使用运行时中性的 UTF-8 处理替换了通用截断工具中的 `Buffer` 用法。
- 将压缩和分支摘要助手转换为类型化结果返回。
- 添加了 `readTextLines()` 以便 JSONL 元数据加载仅读取标题行。
- 从取消没有意义的 Node 文件系统方法中移除了无操作中止处理。
- 将跨越会话边界的文件系统错误映射到类型化的 `SessionError`。
- 添加了类型化的分支摘要错误和原因感知的公共 harness 错误标准化。
- 资源加载器为非 `not_found` 文件系统失败报告结构化诊断。
- 扩展了 `NodeExecutionEnv` 测试，用于文件操作、执行错误、中止、回调、超时和外壳输出溢出。

剩余：

- 无。

备注：

- 保持底层能力/助手 API 在返回 `Result` 时不抛出。
- 保持会话存储/仓库/会话 API 抛出类型化的 `SessionError`。
- 保持公共结构 harness 失败标准化为 `AgentHarnessError`。
- 将 Node 特定的 API 隔离在 `src/harness/env/nodejs.ts`、Node 支持的存储/会话实现或明确的仅 Node 入口点下。
- 在添加 API 时审计通用 harness 工具是否存在 Node 全局变量。
- 审计包导出，以便浏览器/通用导入不会拉入仅 Node 模块。
- 随着 API 发展，继续扩展 `ExecutionEnv` 和外壳输出契约测试。
