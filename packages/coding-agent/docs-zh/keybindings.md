# 键绑定

所有键盘快捷键都可以通过 `~/.pi/agent/keybindings.json` 自定义。每个操作可以绑定到一个或多个按键。

配置文件使用与 pi 内部使用相同的命名空间键绑定 ID，扩展作者在 `keyHint()` 和注入的 `keybindings` 管理器中也使用这些 ID。

使用命名空间前的旧配置（如 `cursorUp` 或 `expandTools`）在启动时会自动迁移到命名空间 ID。

编辑 `keybindings.json` 后，在 pi 中运行 `/reload` 以应用更改而无需重启会话。

## 按键格式

`modifier+key`，其中修饰键为 `ctrl`、`shift`、`alt`（可组合），按键为：

- **字母：** `a-z`
- **数字：** `0-9`
- **特殊键：** `escape`、`esc`、`enter`、`return`、`tab`、`space`、`backspace`、`delete`、`insert`、`clear`、`home`、`end`、`pageUp`、`pageDown`、`up`、`down`、`left`、`right`
- **功能键：** `f1`-`f12`
- **符号：** `` ` ``、`-`、`=`、`[`、`]`、`\`、`;`、`'`、`,`、`.`、`/`、`!`、`@`、`#`、`$`、`%`、`^`、`&`、`*`、`(`、`)`、`_`、`+`、`|`、`~`、`{`、`}`、`:`、`<`、`>`、`?`

修饰键组合：`ctrl+shift+x`、`alt+ctrl+x`、`ctrl+shift+alt+x`、`ctrl+1` 等。

## 所有操作

### TUI 编辑器光标移动

| 键绑定 ID | 默认值 | 说明 |
|-----------|--------|------|
| `tui.editor.cursorUp` | `up` | 向上移动光标 |
| `tui.editor.cursorDown` | `down` | 向下移动光标 |
| `tui.editor.cursorLeft` | `left`、`ctrl+b` | 向左移动光标 |
| `tui.editor.cursorRight` | `right`、`ctrl+f` | 向右移动光标 |
| `tui.editor.cursorWordLeft` | `alt+left`、`ctrl+left`、`alt+b` | 向左移动一个词 |
| `tui.editor.cursorWordRight` | `alt+right`、`ctrl+right`、`alt+f` | 向右移动一个词 |
| `tui.editor.cursorLineStart` | `home`、`ctrl+a` | 移动到行首 |
| `tui.editor.cursorLineEnd` | `end`、`ctrl+e` | 移动到行尾 |
| `tui.editor.jumpForward` | `ctrl+]` | 向前跳转到字符 |
| `tui.editor.jumpBackward` | `ctrl+alt+]` | 向后跳转到字符 |
| `tui.editor.pageUp` | `pageUp` | 向上翻页 |
| `tui.editor.pageDown` | `pageDown` | 向下翻页 |

### TUI 编辑器删除

| 键绑定 ID | 默认值 | 说明 |
|-----------|--------|------|
| `tui.editor.deleteCharBackward` | `backspace` | 向后删除字符 |
| `tui.editor.deleteCharForward` | `delete`、`ctrl+d` | 向前删除字符 |
| `tui.editor.deleteWordBackward` | `ctrl+w`、`alt+backspace` | 向后删除词 |
| `tui.editor.deleteWordForward` | `alt+d`、`alt+delete` | 向前删除词 |
| `tui.editor.deleteToLineStart` | `ctrl+u` | 删除到行首 |
| `tui.editor.deleteToLineEnd` | `ctrl+k` | 删除到行尾 |

### TUI 输入

| 键绑定 ID | 默认值 | 说明 |
|-----------|--------|------|
| `tui.input.newLine` | `shift+enter` | 插入新行 |
| `tui.input.submit` | `enter` | 提交输入 |
| `tui.input.tab` | `tab` | Tab / 自动完成 |

### TUI 杀死环

| 键绑定 ID | 默认值 | 说明 |
|-----------|--------|------|
| `tui.editor.yank` | `ctrl+y` | 粘贴最近删除的文本 |
| `tui.editor.yankPop` | `alt+y` | 在粘贴后循环删除的文本 |
| `tui.editor.undo` | `ctrl+-` | 撤销上次编辑 |

### TUI 剪贴板和选择

| 键绑定 ID | 默认值 | 说明 |
|-----------|--------|------|
| `tui.input.copy` | `ctrl+c` | 复制选择 |
| `tui.select.up` | `up` | 向上移动选择 |
| `tui.select.down` | `down` | 向下移动选择 |
| `tui.select.pageUp` | `pageUp` | 在列表中向上翻页 |
| `tui.select.pageDown` | `pageDown` | 在列表中向下翻页 |
| `tui.select.confirm` | `enter` | 确认选择 |
| `tui.select.cancel` | `escape`、`ctrl+c` | 取消选择 |

### 应用程序

| 键绑定 ID | 默认值 | 说明 |
|-----------|--------|------|
| `app.interrupt` | `escape` | 取消 / 中止 |
| `app.clear` | `ctrl+c` | 清空编辑器 |
| `app.exit` | `ctrl+d` | 退出（编辑器为空时） |
| `app.suspend` | `ctrl+z`（Windows 上无） | 挂起后台 |
| `app.editor.external` | `ctrl+g` | 在外部编辑器中打开（`$VISUAL` 或 `$EDITOR`） |
| `app.clipboard.pasteImage` | `ctrl+v`（Windows 上为 `alt+v`） | 从剪贴板粘贴图像 |

### 会话

| 键绑定 ID | 默认值 | 说明 |
|-----------|--------|------|
| `app.session.new` | *（无）* | 启动新会话（`/new`） |
| `app.session.tree` | *（无）* | 打开会话树导航器（`/tree`） |
| `app.session.fork` | *（无）* | 分支当前会话（`/fork`） |
| `app.session.resume` | *（无）* | 打开会话恢复选择器（`/resume`） |
| `app.session.togglePath` | `ctrl+p` | 切换路径显示 |
| `app.session.toggleSort` | `ctrl+s` | 切换排序模式 |
| `app.session.toggleNamed` | `ctrl+n` | 切换仅命名过滤 |
| `app.session.rename` | `ctrl+r` | 重命名会话 |
| `app.session.delete` | `ctrl+d` | 删除会话 |
| `app.session.deleteNoninvasive` | `ctrl+backspace` | 查询为空时删除会话 |

### 模型和思考

| 键绑定 ID | 默认值 | 说明 |
|-----------|--------|------|
| `app.model.select` | `ctrl+l` | 打开模型选择器 |
| `app.model.cycleForward` | `ctrl+p` | 循环到下一个模型 |
| `app.model.cycleBackward` | `shift+ctrl+p` | 循环到上一个模型 |
| `app.thinking.cycle` | `shift+tab` | 循环思考级别 |
| `app.thinking.toggle` | `ctrl+t` | 折叠或展开思考块 |

### 显示和消息队列

| 键绑定 ID | 默认值 | 说明 |
|-----------|--------|------|
| `app.tools.expand` | `ctrl+o` | 折叠或展开工具输出 |
| `app.message.followUp` | `alt+enter` | 排队后续消息 |
| `app.message.dequeue` | `alt+up` | 将排队的消息恢复到编辑器 |

### 树导航

| 键绑定 ID | 默认值 | 说明 |
|-----------|--------|------|
| `app.tree.foldOrUp` | `ctrl+left`、`alt+left` | 折叠当前分支段，或跳转到上一个段开始 |
| `app.tree.unfoldOrDown` | `ctrl+right`、`alt+right` | 展开当前分支段，或跳转到下一个段开始或分支结束 |
| `app.tree.editLabel` | `shift+l` | 编辑选定树节点上的标签 |
| `app.tree.toggleLabelTimestamp` | `shift+t` | 在树中切换标签时间戳 |
| `app.tree.filter.default` | `ctrl+d` | 将树过滤器设置为默认视图 |
| `app.tree.filter.noTools` | `ctrl+t` | 切换隐藏工具结果的树过滤器 |
| `app.tree.filter.userOnly` | `ctrl+u` | 切换仅显示用户消息的树过滤器 |
| `app.tree.filter.labeledOnly` | `ctrl+l` | 切换仅显示带标签条目的树过滤器 |
| `app.tree.filter.all` | `ctrl+a` | 切换显示所有条目的树过滤器 |
| `app.tree.filter.cycleForward` | `ctrl+o` | 向前循环树过滤器 |
| `app.tree.filter.cycleBackward` | `shift+ctrl+o` | 向后循环树过滤器 |

### 范围化模型选择器

在范围化模型选择器中使用（通过 `/scoped-models` 打开）。

| 键绑定 ID | 默认值 | 说明 |
|-----------|--------|------|
| `app.models.save` | `ctrl+s` | 将当前模型选择保存到设置 |
| `app.models.enableAll` | `ctrl+a` | 启用所有模型（或所有匹配当前搜索的模型） |
| `app.models.clearAll` | `ctrl+x` | 清除所有模型（或所有匹配当前搜索的模型） |
| `app.models.toggleProvider` | `ctrl+p` | 切换当前提供者的所有模型 |
| `app.models.reorderUp` | `alt+up` | 在循环顺序中向上移动选定模型 |
| `app.models.reorderDown` | `alt+down` | 在循环顺序中向下移动选定模型 |

## 自定义配置

创建 `~/.pi/agent/keybindings.json`：

```json
{
  "tui.editor.cursorUp": ["up", "ctrl+p"],
  "tui.editor.cursorDown": ["down", "ctrl+n"],
  "tui.editor.deleteWordBackward": ["ctrl+w", "alt+backspace"]
}
```

每个操作可以有单个按键或按键数组。用户配置会覆盖默认值。

在原生 Windows 上，`app.suspend` 没有默认绑定，因为 Windows 终端不支持 Unix 作业控制。如果你手动绑定它，pi 会显示状态消息而不是挂起。在 WSL 中，正常的 Linux `ctrl+z`/`fg` 行为仍然适用。

### Emacs 示例

```json
{
  "tui.editor.cursorUp": ["up", "ctrl+p"],
  "tui.editor.cursorDown": ["down", "ctrl+n"],
  "tui.editor.cursorLeft": ["left", "ctrl+b"],
  "tui.editor.cursorRight": ["right", "ctrl+f"],
  "tui.editor.cursorWordLeft": ["alt+left", "alt+b"],
  "tui.editor.cursorWordRight": ["alt+right", "alt+f"],
  "tui.editor.deleteCharForward": ["delete", "ctrl+d"],
  "tui.editor.deleteCharBackward": ["backspace", "ctrl+h"],
  "tui.input.newLine": ["shift+enter", "ctrl+j"]
}
```

### Vim 示例

```json
{
  "tui.editor.cursorUp": ["up", "alt+k"],
  "tui.editor.cursorDown": ["down", "alt+j"],
  "tui.editor.cursorLeft": ["left", "alt+h"],
  "tui.editor.cursorRight": ["right", "alt+l"],
  "tui.editor.cursorWordLeft": ["alt+left", "alt+b"],
  "tui.editor.cursorWordRight": ["alt+right", "alt+w"]
}
```
