> pi 可以创建技能。让它为你的用例构建一个。

# 技能

技能是自包含的能力包，代理按需加载。技能为特定任务提供专业的工作流程、设置说明、辅助脚本和参考文档。

Pi 实现了 [Agent Skills 标准](https://agentskills.io/specification)，警告大多数违规但保持宽容。Pi 允许技能名称与其父目录不同，尽管标准不允许这样做；该规则对于跨多个代理工具使用的共享技能目录来说并不理想。

## 目录

- [位置](#位置)
- [技能如何工作](#技能如何工作)
- [技能命令](#技能命令)
- [技能结构](#技能结构)
- [前置元数据](#前置元数据)
- [验证](#验证)
- [示例](#示例)
- [技能仓库](#技能仓库)

## 位置

> **安全：** 技能可以指示模型执行任何操作，并且可能包含模型调用的可执行代码。使用前请检查技能内容。

Pi 从以下位置加载技能：

- 全局：
  - `~/.pi/agent/skills/`
  - `~/.agents/skills/`
- 项目：
  - `.pi/skills/`
  - `cwd` 和祖先目录中的 `.agents/skills/`（向上到 git 仓库根，或不在仓库中时到文件系统根）
- 包：`skills/` 目录或 package.json 中的 pi.skills 条目
- 设置：包含文件或目录的 `skills` 数组
- CLI：`--skill <path>`（可重复，即使使用 `--no-skills` 也会添加）

发现规则：
- 在 `~/.pi/agent/skills/` 和 `.pi/skills/` 中，直接根目录的 `.md` 文件被发现为单独的技能
- 在所有技能位置中，包含 `SKILL.md` 的目录被递归发现
- 在 `~/.agents/skills/` 和项目 `.agents/skills/` 中，根目录的 `.md` 文件被忽略

使用 `--no-skills` 禁用发现（显式的 `--skill` 路径仍会加载）。

### 使用来自其他工具的技能

要使用来自 Claude Code 或 OpenAI Codex 的技能，请将它们的目录添加到设置中：

```json
{
  "skills": [
    "~/.claude/skills",
    "~/.codex/skills"
  ]
}
```

对于项目级 Claude Code 技能，请添加到 `.pi/settings.json`：

```json
{
  "skills": ["../.claude/skills"]
}
```

## 技能如何工作

1. 在启动时，pi 扫描技能位置并提取名称和描述
2. 系统提示按照 [规范](https://agentskills.io/integrate-skills) 包含可用的技能，格式为 XML
3. 当任务匹配时，代理使用 `read` 加载完整的 SKILL.md（模型不总是这样做；使用提示或 `/skill:name` 强制它）
4. 代理遵循说明，使用相对路径引用脚本和资源

这是渐进式披露：只有描述始终在上下文中，完整说明按需加载。

## 技能命令

技能注册为 `/skill:name` 命令：

```bash
/skill:brave-search           # 加载并执行技能
/skill:pdf-tools extract      # 带参数加载技能
```

命令后的参数作为 `User: <args>` 附加到技能内容。

在交互模式下使用 `/settings` 或在 `settings.json` 中切换技能命令：

```json
{
  "enableSkillCommands": true
}
```

## 技能结构

技能是一个包含 `SKILL.md` 文件的目录。其他所有内容都是自由形式的。

```
my-skill/
├── SKILL.md              # 必需：前置元数据 + 说明
├── scripts/              # 辅助脚本
│   └── process.sh
├── references/         # 按需加载的详细文档
│   └── api-reference.md
└── assets/
    └── template.json
```

### SKILL.md 格式

````markdown
---
name: my-skill
description: 该技能的作用以及何时使用它。请具体说明。
---

# 我的技能

## 设置

首次使用前运行一次：
```bash
cd /path/to/skill && npm install
```

## 使用

```bash
./scripts/process.sh <input>
```
````

使用来自技能目录的相对路径：

```markdown
有关详细信息，请参阅 [参考指南](references/REFERENCE.md)。
```

## 前置元数据

根据 [Agent Skills 规范](https://agentskills.io/specification#frontmatter-required)：

| 字段 | 必需 | 描述 |
|------|------|------|
| `name` | 是 | 最多 64 个字符。小写字母 a-z、0-9、连字符。与标准不同，Pi 不要求这与父目录匹配，因为该标准要求对于共享技能目录来说并不理想。 |
| `description` | 是 | 最多 1024 个字符。技能的作用以及何时使用它。 |
| `license` | 否 | 许可证名称或对捆绑文件的引用。 |
| `compatibility` | 否 | 最多 500 个字符。环境要求。 |
| `metadata` | 否 | 任意键值映射。 |
| `allowed-tools` | 否 | 空格分隔的预先批准工具列表（实验性）。 |
| `disable-model-invocation` | 否 | 当为 `true` 时，技能对系统提示隐藏。用户必须使用 `/skill:name`。 |

### 名称规则

- 1-64 个字符
- 仅小写字母、数字、连字符
- 无前导/尾随连字符
- 无连续连字符
- Pi 不要求名称与父目录匹配。Agent Skills 标准要求这样，但该要求对于多个工具使用的共享技能目录来说并不理想。

有效：`pdf-processing`、`data-analysis`、`code-review`
无效：`PDF-Processing`、`-pdf`、`pdf--processing`

### 描述最佳实践

描述决定了代理何时加载技能。请具体说明。

好的：
```yaml
description: 从 PDF 文件中提取文本和表格，填写 PDF 表单，并合并多个 PDF。在处理 PDF 文档时使用。
```

不好的：
```yaml
description: 帮助处理 PDF。
```

## 验证

Pi 根据 Agent Skills 标准验证技能。大多数问题会产生警告但仍会加载技能：

- 名称超过 64 个字符或包含无效字符
- 名称以连字符开头/结尾或有连续连字符
- 描述超过 1024 个字符

未知的前置元数据字段被忽略。

**例外：** 缺少描述的技能不会被加载。

名称冲突（来自不同位置的相同名称）会警告并保留找到的第一个技能。

## 示例

```
brave-search/
├── SKILL.md
├── search.js
└── content.js
```

**SKILL.md:**
````markdown
---
name: brave-search
description: 通过 Brave Search API 的网络搜索和内容提取。用于搜索文档、事实或任何网络内容。
---

# Brave Search

## 设置

```bash
cd /path/to/brave-search && npm install
```

## 搜索

```bash
./search.js "query"              # 基本搜索
./search.js "query" --content    # 包含页面内容
```

## 提取页面内容

```bash
./content.js https://example.com
```
````

## 技能仓库

- [Anthropic Skills](https://github.com/anthropics/skills) - 文档处理（docx、pdf、pptx、xlsx）、网络开发
- [Pi Skills](https://github.com/badlogic/pi-skills) - 网络搜索、浏览器自动化、Google API、转录
