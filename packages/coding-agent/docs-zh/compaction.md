# 压缩和分支摘要

LLM 的上下文窗口是有限的。当对话变得太长时，pi 使用压缩来总结旧内容，同时保留最近的工作。本页面涵盖自动压缩和分支摘要。

**源文件**（[pi-mono](https://github.com/earendil-works/pi-mono)）：
- [`packages/coding-agent/src/core/compaction/compaction.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/compaction.ts) - 自动压缩逻辑
- [`packages/coding-agent/src/core/compaction/branch-summarization.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts) - 分支摘要
- [`packages/coding-agent/src/core/compaction/utils.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/utils.ts) - 共享实用程序（文件跟踪、序列化）
- [`packages/coding-agent/src/core/session-manager.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/session-manager.ts) - 条目类型（`CompactionEntry`、`BranchSummaryEntry`）
- [`packages/coding-agent/src/core/extensions/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/extensions/types.ts) - 扩展事件类型

要在项目中使用 TypeScript 定义，请查看 `node_modules/@earendil-works/pi-coding-agent/dist/`。

## 概述

Pi 有两种摘要机制：

| 机制 | 触发方式 | 用途 |
|------|---------|------|
| 压缩 | 上下文超过阈值，或 `/compact` | 总结旧消息以释放上下文 |
| 分支摘要 | `/tree` 导航 | 在切换分支时保留上下文 |

两者都使用相同的结构化摘要格式，并累积跟踪文件操作。

## 压缩

### 触发时机

自动压缩在以下情况下触发：

```
contextTokens > contextWindow - reserveTokens
```

默认情况下，`reserveTokens` 是 16384 个令牌（可在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置）。这为 LLM 的响应留出了空间。

你也可以使用 `/compact [instructions]` 手动触发，其中可选的说明可聚焦摘要内容。

### 工作原理

1. **找到裁剪点**：从最新消息向后走，累积令牌估计，直到达到 `keepRecentTokens`（默认 20000，可在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置）
2. **提取消息**：收集从之前保留的边界（或会话开始）到裁剪点的消息
3. **生成摘要**：调用 LLM 使用结构化格式进行总结，当存在之前的摘要时，将其作为迭代上下文传递
4. **追加条目**：保存带有摘要和 `firstKeptEntryId` 的 `CompactionEntry`
5. **重新加载**：会话重新加载，使用摘要 + 从 `firstKeptEntryId` 开始的消息

```
压缩前：

  条目： 0     1     2     3      4     5     6      7      8     9
        ┌─────┬─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┐
        │ 头  │ 用户 │ 助手 │ 工具 │ 用户 │ 助手 │ 工具 │ 工具 │ 助手 │ 工具 │
        └─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┴─────┘
               └────────┬───────┘ └──────────────┬──────────────┘
              要总结的消息              保留的消息
                                               ↑
                                       firstKeptEntryId（条目 4）

压缩后（追加新条目）：

  条目： 0     1     2     3      4     5     6      7      8     9     10
        ┌─────┬─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┬─────┐
        │ 头  │ 用户 │ 助手 │ 工具 │ 用户 │ 助手 │ 工具 │ 工具 │ 助手 │ 工具 │ 压缩 │
        └─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┴─────┴─────┘
              └──────────┬──────┘ └──────────────────┬───────────────────┘
                   不发送给 LLM                  发送给 LLM
                                                          ↑
                                                  从 firstKeptEntryId 开始

LLM 看到的内容：

  ┌────────┬─────────┬─────┬─────┬──────┬──────┬─────┬──────┐
  │ 系统   │ 摘要   │ 用户 │ 助手 │ 工具 │ 工具 │ 助手 │ 工具 │
  └────────┴─────────┴─────┴─────┴──────┴──────┴─────┴──────┘
       ↑         ↑      └─────────────────┬────────────────┘
    提示    来自压缩       从 firstKeptEntryId 开始的消息
```

在重复压缩时，摘要跨度从之前压缩的保留边界（`firstKeptEntryId`）开始，而不是从压缩条目本身开始，如果在路径中找不到该保留条目，则回退到之前压缩之后的条目。这通过在下次摘要传递中也包含它们，保留了在早期压缩中存活下来的消息。Pi 还会在写入新的 `CompactionEntry` 之前，从重建的会话上下文中重新计算 `tokensBefore`，因此令牌计数反映了正在替换的实际压缩前上下文。

### 分割回合

一个“回合”从用户消息开始，包括所有助手响应和工具调用，直到下一条用户消息。通常，压缩在回合边界处裁剪。

当单个回合超过 `keepRecentTokens` 时，裁剪点落在回合中间的助手消息处。这是一个“分割回合”：

```
分割回合（一个巨大的回合超出预算）：

  条目： 0     1     2      3     4      5      6     7      8
        ┌─────┬─────┬─────┬──────┬─────┬──────┬──────┬─────┬──────┐
        │ 头  │ 用户 │ 助手 │ 工具 │ 助手 │ 工具 │ 工具 │ 助手 │ 工具 │
        └─────┴─────┴─────┴──────┴─────┴──────┴──────┴─────┴──────┘
               ↑                                     ↑
        turnStartIndex = 1                  firstKeptEntryId = 7
               │                                     │
               └──── 回合前缀消息（1-6） ────────────┘
                                                     └── 保留（7-8）

  isSplitTurn = true
  要总结的消息 = [] （之前没有完整的回合）
  回合前缀消息 = [用户, 助手, 工具, 助手, 工具, 工具]
```

对于分割回合，pi 生成两个摘要并合并它们：
1. **历史摘要**：之前的上下文（如果有）
2. **回合前缀摘要**：分割回合的早期部分

### 裁剪点规则

有效的裁剪点是：
- 用户消息
- 助手消息
- Bash 执行消息
- 自定义消息（custom_message、branch_summary）

永远不要在工具结果处裁剪（它们必须与工具调用保持在一起）。

### CompactionEntry 结构

在 [`session-manager.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/session-manager.ts) 中定义：

```typescript
interface CompactionEntry<T = unknown> {
  type: "compaction";
  id: string;
  parentId: string;
  timestamp: number;
  summary: string;
  firstKeptEntryId: string;
  tokensBefore: number;
  fromHook?: boolean;  // 如果由扩展提供则为 true（遗留字段名称）
  details?: T;         // 实现特定的数据
}

// 默认压缩使用这个作为 details（来自 compaction.ts）：
interface CompactionDetails {
  readFiles: string[];
  modifiedFiles: string[];
}
```

扩展可以在 `details` 中存储任何 JSON 可序列化的数据。默认压缩跟踪文件操作，但自定义扩展实现可以使用自己的结构。

有关实现，请参阅 [`prepareCompaction()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/compaction.ts) 和 [`compact()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/compaction.ts)。

## 分支摘要

### 触发时机

当你使用 `/tree` 导航到不同分支时，pi 会提供总结你要离开的工作。这会将左侧分支的上下文注入到新分支中。

### 工作原理

1. **找到共同祖先**：新旧位置共享的最深节点
2. **收集条目**：从旧叶子向后走到共同祖先
3. **使用预算准备**：包含直到令牌预算的消息（最新优先）
4. **生成摘要**：使用结构化格式调用 LLM
5. **追加条目**：在导航点保存 `BranchSummaryEntry`

```
导航前的树：

         ┌─ B ─ C ─ D（旧叶子，正在被放弃）
    A ───┤
         └─ E ─ F（目标）

共同祖先：A
要总结的条目：B、C、D

带有摘要的导航后：

         ┌─ B ─ C ─ D ─ [B、C、D 的摘要]
    A ───┤
         └─ E ─ F（新叶子）
```

### 累积文件跟踪

压缩和分支摘要都累积跟踪文件。生成摘要时，pi 从以下位置提取文件操作：
- 正在总结的消息中的工具调用
- 之前的压缩或分支摘要 `details`（如果有）

这意味着文件跟踪在多次压缩或嵌套分支摘要中累积，保留了读取和修改文件的完整历史记录。

### BranchSummaryEntry 结构

在 [`session-manager.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/session-manager.ts) 中定义：

```typescript
interface BranchSummaryEntry<T = unknown> {
  type: "branch_summary";
  id: string;
  parentId: string;
  timestamp: number;
  summary: string;
  fromId: string;      // 我们从其导航的条目
  fromHook?: boolean;  // 如果由扩展提供则为 true（遗留字段名称）
  details?: T;         // 实现特定的数据
}

// 默认分支摘要使用这个作为 details（来自 branch-summarization.ts）：
interface BranchSummaryDetails {
  readFiles: string[];
  modifiedFiles: string[];
}
```

与压缩相同，扩展可以在 `details` 中存储自定义数据。

有关实现，请参阅 [`collectEntriesForBranchSummary()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)、[`prepareBranchEntries()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts) 和 [`generateBranchSummary()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)。

## 摘要格式

压缩和分支摘要都使用相同的结构化格式：

```markdown
## 目标
[用户想要实现的目标]

## 约束和偏好
- [用户提到的需求]

## 进度
### 已完成
- [x] [已完成的任务]

### 进行中
- [ ] [当前工作]

### 受阻
- [问题（如果有）]

## 关键决策
- **[决策]**：[理由]

## 下一步
1. [接下来应该发生什么]

## 关键上下文
- [继续所需的数据]

<read-files>
path/to/file1.ts
path/to/file2.ts
</read-files>

<modified-files>
path/to/changed.ts
</modified-files>
```

### 消息序列化

在摘要之前，消息通过 [`serializeConversation()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/utils.ts) 序列化为文本：

```
[用户]：他们说的话
[助手思考]：内部推理
[助手]：响应文本
[助手工具调用]：read(path="foo.ts"); edit(path="bar.ts", ...)
[工具结果]：来自工具的输出
```

这可以防止模型将其视为要继续的对话。

工具结果在序列化过程中被截断为 2000 个字符。超过该限制的内容将被替换为一个标记，指示被截断了多少个字符。这使摘要请求保持在合理的令牌预算内，因为工具结果（特别是来自 `read` 和 `bash` 的结果）通常是上下文大小的最大贡献者。

## 通过扩展自定义摘要

扩展可以拦截并自定义压缩和分支摘要。有关事件类型定义，请参阅 [`extensions/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/extensions/types.ts)。

### session_before_compact

在自动压缩或 `/compact` 之前触发。可以取消或提供自定义摘要。请参阅类型文件中的 `SessionBeforeCompactEvent` 和 `CompactionPreparation`。

```typescript
pi.on("session_before_compact", async (event, ctx) => {
  const { preparation, branchEntries, customInstructions, signal } = event;

  // preparation.messagesToSummarize - 要总结的消息
  // preparation.turnPrefixMessages - 分割回合前缀（如果 isSplitTurn）
  // preparation.previousSummary - 之前的压缩摘要
  // preparation.fileOps - 提取的文件操作
  // preparation.tokensBefore - 压缩前的上下文令牌
  // preparation.firstKeptEntryId - 保留消息的起始位置
  // preparation.settings - 压缩设置

  // branchEntries - 当前分支上的所有条目（用于自定义状态）
  // signal - AbortSignal（传递给 LLM 调用）

  // 取消：
  return { cancel: true };

  // 自定义摘要：
  return {
    compaction: {
      summary: "你的摘要...",
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
      details: { /* 自定义数据 */ },
    }
  };
});
```

#### 将消息转换为文本

要使用你自己的模型生成摘要，请使用 `serializeConversation` 将消息转换为文本：

```typescript
import { convertToLlm, serializeConversation } from "@earendil-works/pi-coding-agent";

pi.on("session_before_compact", async (event, ctx) => {
  const { preparation } = event;
  
  // 将 AgentMessage[] 转换为 Message[]，然后序列化为文本
  const conversationText = serializeConversation(
    convertToLlm(preparation.messagesToSummarize)
  );
  // 返回：
  // [用户]：消息文本
  // [助手思考]：思考内容
  // [助手]：响应文本
  // [助手工具调用]：read(path="..."); bash(command="...")
  // [工具结果]：输出文本

  // 现在发送到你的模型进行摘要
  const summary = await myModel.summarize(conversationText);
  
  return {
    compaction: {
      summary,
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
    }
  };
});
```

有关使用不同模型的完整示例，请参阅 [custom-compaction.ts](../examples/extensions/custom-compaction.ts)。

### session_before_tree

在 `/tree` 导航之前触发。无论用户是否选择摘要，都会始终触发。可以取消导航或提供自定义摘要。

```typescript
pi.on("session_before_tree", async (event, ctx) => {
  const { preparation, signal } = event;

  // preparation.targetId - 我们要导航到的位置
  // preparation.oldLeafId - 当前位置（正在被放弃）
  // preparation.commonAncestorId - 共同祖先
  // preparation.entriesToSummarize - 将被总结的条目
  // preparation.userWantsSummary - 用户是否想要摘要

  // 完全取消导航：
  return { cancel: true };

  // 提供自定义摘要（仅在 userWantsSummary 为 true 时使用）：
  if (preparation.userWantsSummary) {
    return {
      summary: {
        summary: "你的摘要...",
        details: { /* 自定义数据 */ },
      }
    };
  }
});
```

请参阅类型文件中的 `SessionBeforeTreeEvent` 和 `TreePreparation`。

## 设置

在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置压缩：

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

| 设置 | 默认值 | 描述 |
|------|--------|------|
| `enabled` | `true` | 启用自动压缩 |
| `reserveTokens` | `16384` | 为 LLM 响应保留的令牌数 |
| `keepRecentTokens` | `20000` | 保留的最近令牌数（不总结） |

使用 `"enabled": false` 禁用自动压缩。你仍然可以使用 `/compact` 手动压缩。
