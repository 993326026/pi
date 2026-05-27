# 快速开始

本页面将带你从安装到完成第一个有用的 pi 会话。

## 安装

Pi 作为 npm 包分发：

```bash
npm install -g @earendil-works/pi-coding-agent
```

然后在你希望它工作的项目目录中启动 pi：

```bash
cd /path/to/project
pi
```

## 身份验证

Pi 可以通过 `/login` 使用订阅提供商，或通过环境变量或身份验证文件使用 API 密钥提供商。

### 选项 1：订阅登录

启动 pi 并运行：

```text
/login
```

然后选择一个提供商。内置的订阅登录包括 Claude Pro/Max、ChatGPT Plus/Pro (Codex) 和 GitHub Copilot。

### 选项 2：API 密钥

在启动 pi 之前设置 API 密钥：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

你也可以运行 `/login` 并选择一个 API 密钥提供商，将密钥存储在 `~/.pi/agent/auth.json` 中。

有关所有支持的提供商、环境变量和云提供商设置，请参阅 [提供商](providers.md)。

## 第一个会话

pi 启动后，输入请求并按 Enter：

```text
总结这个仓库并告诉我如何运行其检查。
```

默认情况下，pi 为模型提供四个工具：

- `read` - 读取文件
- `write` - 创建或覆盖文件
- `edit` - 修补文件
- `bash` - 运行 shell 命令

其他内置的只读工具（`grep`、`find`、`ls`）可通过工具选项获得。Pi 在你当前的工作目录中运行并可以修改那里的文件。如果你想要轻松回滚，请使用 git 或其他检查点工作流程。

## 给 pi 项目说明

Pi 在启动时加载上下文文件。添加一个 `AGENTS.md` 文件来告诉它如何在项目中工作：

```markdown
# 项目说明

- 代码更改后运行 `npm run check`。
- 不要在本地运行生产迁移。
- 保持回复简洁。
```

Pi 加载：

- `~/.pi/agent/AGENTS.md` 用于全局说明
- 从父目录和当前目录加载 `AGENTS.md` 或 `CLAUDE.md`

更改上下文文件后，重启 pi 或运行 `/reload`。

## 常见尝试内容

### 引用文件

在编辑器中输入 `@` 以模糊搜索文件，或在命令行上传递文件：

```bash
pi @README.md "总结这个"
pi @src/app.ts @src/app.test.ts "一起审查这些"
```

图像可以用 Ctrl+V（Windows 上是 Alt+V）粘贴，或拖入支持的终端。

### 运行 shell 命令

在交互模式下：

```text
!npm run lint
```

命令输出被发送到模型。使用 `!!command` 运行命令而不将其输出添加到模型上下文中。

### 切换模型

使用 `/model` 或 Ctrl+L 选择模型。使用 Shift+Tab 循环思考级别。使用 Ctrl+P / Shift+Ctrl+P 在作用域模型之间循环。

### 稍后继续

会话会自动保存：

```bash
pi -c                  # 继续最近的会话
pi -r                  # 浏览之前的会话
pi --session <path|id> # 打开特定会话
```

在 pi 内部，使用 `/resume`、`/new`、`/tree`、`/fork` 和 `/clone` 来管理会话。

### 非交互模式

用于一次性提示：

```bash
pi -p "总结这个代码库"
cat README.md | pi -p "总结这段文本"
pi -p @screenshot.png "这张图片里有什么？"
```

使用 `--mode json` 获取 JSON 事件输出，或使用 `--mode rpc` 进行进程集成。

## 下一步

- [使用 Pi](usage.md) - 交互模式、斜杠命令、会话、上下文文件和 CLI 参考。
- [提供商](providers.md) - 身份验证和模型设置。
- [设置](settings.md) - 全局和项目配置。
- [按键绑定](keybindings.md) - 快捷键和自定义。
- [Pi 包](packages.md) - 安装共享扩展、技能、提示词和主题。

平台说明：[Windows](windows.md)、[Termux](termux.md)、[tmux](tmux.md)、[终端设置](terminal-setup.md)、[Shell 别名](shell-aliases.md)。
