# 终端设置

Pi 使用 [Kitty 键盘协议](https://sw.kovidgoyal.net/kitty/keyboard-protocol/) 进行可靠的修饰键检测。大多数现代终端支持此协议，但有些需要配置。

## Kitty、iTerm2

开箱即用。

## Ghostty

添加到你的 Ghostty 配置（macOS 上为 `~/Library/Application Support/com.mitchellh.ghostty/config`，Linux 上为 `~/.config/ghostty/config`）：

```
keybind = alt+backspace=text:\x1b\x7f
```

较旧的 Claude Code 版本可能添加了此 Ghostty 映射：

```
keybind = shift+enter=text:\n
```

该映射发送原始换行字节。在 pi 内部，这与 `Ctrl+J` 无法区分，因此 tmux 和 pi 不再看到真实的 `shift+enter` 键事件。

如果 Claude Code 2.x 或更高版本是你添加该映射的唯一原因，则可以删除它，除非你想在 tmux 中使用 Claude Code，而它仍然需要该 Ghostty 映射。

如果你希望 `Shift+Enter` 通过该重映射继续在 tmux 中工作，请在 `~/.pi/agent/keybindings.json` 中向 pi 的 `newLine` 键绑定添加 `ctrl+j`：

```json
{
  "newLine": ["shift+enter", "ctrl+j"]
}
```

## WezTerm

创建 `~/.wezterm.lua`：

```lua
local wezterm = require 'wezterm'
local config = wezterm.config_builder()
config.enable_kitty_keyboard = true
return config
```

## VS Code（集成终端）

`keybindings.json` 位置：
- macOS：`~/Library/Application Support/Code/User/keybindings.json`
- Linux：`~/.config/Code/User/keybindings.json`
- Windows：`%APPDATA%\Code\User\keybindings.json`

添加到 `keybindings.json` 以启用 `Shift+Enter` 进行多行输入：

```json
{
  "key": "shift+enter",
  "command": "workbench.action.terminal.sendSequence",
  "args": { "text": "\u001b[13;2u" },
  "when": "terminalFocus"
}
```

## Windows Terminal

添加到 `settings.json`（Ctrl+Shift+, 或设置 → 打开 JSON 文件）以转发 pi 使用的修改后的 Enter 键：

```json
{
  "actions": [
    {
      "command": { "action": "sendInput", "input": "\u001b[13;2u" },
      "keys": "shift+enter"
    },
    {
      "command": { "action": "sendInput", "input": "\u001b[13;3u" },
      "keys": "alt+enter"
    }
  ]
}
```

- `Shift+Enter` 插入新行。
- Windows Terminal 默认将 `Alt+Enter` 绑定到全屏。这会阻止 pi 接收 `Alt+Enter` 进行后续排队。
- 将 `Alt+Enter` 重映射到 `sendInput` 会将真实的键和弦转发到 pi。

如果你已经有 `actions` 数组，请将对象添加到其中。如果旧的全屏行为仍然存在，请完全关闭并重新打开 Windows Terminal。

## xfce4-terminal、terminator

这些终端对转义序列的支持有限。像 `Ctrl+Enter` 和 `Shift+Enter` 这样的修改后的 Enter 键无法与普通 `Enter` 区分开来，从而阻止自定义键绑定（如 `submit: ["ctrl+enter"]`）工作。

为获得最佳体验，请使用支持 Kitty 键盘协议的终端：
- [Kitty](https://sw.kovidgoyal.net/kitty/)
- [Ghostty](https://ghostty.org/)
- [WezTerm](https://wezfurlong.org/wezterm/)
- [iTerm2](https://iterm2.com/)
- [Alacritty](https://github.com/alacritty/alacritty)（需要使用 Kitty 协议支持编译）

## IntelliJ IDEA（集成终端）

内置终端对转义序列的支持有限。在 IntelliJ 的终端中，Shift+Enter 无法与 Enter 区分开来。

如果你希望硬件光标可见，请在运行 pi 之前设置 `PI_HARDWARE_CURSOR=1`（为兼容性默认禁用）。

考虑使用专用终端模拟器以获得最佳体验。
