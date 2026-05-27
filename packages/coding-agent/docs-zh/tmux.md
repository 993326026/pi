# tmux 设置

Pi 在 tmux 中工作，但 tmux 默认会剥离某些键的修饰信息。如果没有配置，`Shift+Enter` 和 `Ctrl+Enter` 通常无法与普通 `Enter` 区分开来。

## 推荐配置

添加到 `~/.tmux.conf`：

```tmux
set -g extended-keys on
set -g extended-keys-format csi-u
```

然后完全重启 tmux：

```bash
tmux kill-server
tmux
```

当 Kitty 键盘协议不可用时，Pi 会自动请求扩展键报告。使用 `extended-keys-format csi-u`，tmux 会以 CSI-u 格式转发修饰键，这是最可靠的配置。

## 为什么推荐 `csi-u`

仅使用：

```tmux
set -g extended-keys on
```

tmux 默认使用 `extended-keys-format xterm`。当应用程序请求扩展键报告时，修饰键会以 xterm `modifyOtherKeys` 格式转发，例如：

- `Ctrl+C` → `\x1b[27;5;99~`
- `Ctrl+D` → `\x1b[27;5;100~`
- `Ctrl+Enter` → `\x1b[27;5;13~`

使用 `extended-keys-format csi-u`，相同的键会转发为：

- `Ctrl+C` → `\x1b[99;5u`
- `Ctrl+D` → `\x1b[100;5u`
- `Ctrl+Enter` → `\x1b[13;5u`

Pi 支持两种格式，但 `csi-u` 是推荐的 tmux 设置。

## 这解决了什么问题

没有 tmux 扩展键的情况下，修饰后的 Enter 键会折叠为旧序列：

| 键 | 没有 extkeys | 有 `csi-u` |
|----|--------------|-----------|
| Enter | `\r` | `\r` |
| Shift+Enter | `\r` | `\x1b[13;2u` |
| Ctrl+Enter | `\r` | `\x1b[13;5u` |
| Alt/Option+Enter | `\x1b\r` | `\x1b[13;3u` |

这会影响默认键绑定（`Enter` 提交，`Shift+Enter` 换行）以及任何使用修饰后 Enter 的自定义键绑定。

## 要求

- tmux 3.2 或更高版本（运行 `tmux -V` 检查）
- 支持扩展键的终端模拟器（Ghostty、Kitty、iTerm2、WezTerm、Windows Terminal）
