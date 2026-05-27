# Termux（Android）设置

Pi 通过 [Termux](https://termux.dev/)（Android 的终端模拟器和 Linux 环境）在 Android 上运行。

## 先决条件

1. 从 GitHub 或 F-Droid 安装 [Termux](https://github.com/termux/termux-app#installation)（不是 Google Play，该版本已弃用）
2. 从 GitHub 或 F-Droid 安装 [Termux:API](https://github.com/termux/termux-api#installation) 以获取剪贴板和其他设备集成

## 安装

```bash
# 更新包
pkg update && pkg upgrade

# 安装依赖
pkg install nodejs termux-api git

# 安装 pi
npm install -g @earendil-works/pi-coding-agent

# 创建配置目录
mkdir -p ~/.pi/agent

# 运行 pi
pi
```

## 剪贴板支持

在 Termux 中运行时，剪贴板操作使用 `termux-clipboard-set` 和 `termux-clipboard-get`。必须安装 Termux:API 应用才能使这些工作。

Termux 上不支持图像剪贴板（`ctrl+v` 图像粘贴功能将无法工作）。

## Termux 的示例 AGENTS.md

创建 `~/.pi/agent/AGENTS.md` 以帮助代理理解 Termux 环境：

```markdown
# 代理环境：Android 上的 Termux

## 位置
- **操作系统**：Android（Termux 终端模拟器）
- **主目录**：`/data/data/com.termux/files/home`
- **前缀**：`/data/data/com.termux/files/usr`
- **共享存储**：`/storage/emulated/0`（下载、文档等）

## 打开 URL
```bash
termux-open-url "https://example.com"
```

## 打开文件
```bash
termux-open file.pdf          # 使用默认应用打开
termux-open --chooser image.jpg      # 选择应用
```

## 剪贴板
```bash
termux-clipboard-set "文本"   # 复制
termux-clipboard-get          # 粘贴
```

## 通知
```bash
termux-notification -t "标题" -c "内容"
```

## 设备信息
```bash
termux-battery-status         # 电池信息
termux-wifi-connectioninfo    # WiFi 信息
termux-telephony-deviceinfo   # 设备信息
```

## 共享
```bash
termux-share -a send file.txt # 共享文件
```

## 其他有用命令
```bash
termux-toast "消息"           # 快速 Toast 弹出窗口
termux-vibrate                # 振动设备
termux-tts-speak "你好"       # 文本到语音
termux-camera-photo out.jpg   # 拍照
```

## 注意事项
- `termux-*` 命令需要安装 Termux:API 应用
- 使用 `pkg install termux-api` 安装命令行工具
- 访问 `/storage/emulated/0` 需要存储权限
```

## 限制

- **无图像剪贴板**：Termux 剪贴板 API 仅支持文本
- **无本机二进制文件**：某些可选的本机依赖项（如剪贴板模块）在 Android ARM64 上不可用，并在安装过程中被跳过
- **存储访问**：要访问 `/storage/emulated/0`（下载等）中的文件，请运行一次 `termux-setup-storage` 以授予权限

## 故障排除

### 剪贴板不工作

确保两个应用都已安装：
1. Termux（来自 GitHub 或 F-Droid）
2. Termux:API（来自 GitHub 或 F-Droid）

然后安装命令行工具：
```bash
pkg install termux-api
```

### 共享存储权限被拒绝

运行一次以授予存储权限：
```bash
termux-setup-storage
```

### Node.js 安装问题

如果 npm 失败，请尝试清除缓存：
```bash
npm cache clean --force
```
