# 开发

请参阅 [AGENTS.md](../../../AGENTS.md) 获取其他指南。

## 设置

```bash
git clone https://github.com/earendil-works/pi-mono
cd pi-mono
npm install
npm run build
```

从源代码运行：

```bash
/path/to/pi-mono/pi-test.sh
```

该脚本可以从任何目录运行。Pi 保持调用者的当前工作目录。

## 分支/重新品牌化

通过 `package.json` 配置：

```json
{
  "piConfig": {
    "name": "pi",
    "configDir": ".pi"
  }
}
```

为你的分支更改 `name`、`configDir` 和 `bin` 字段。影响 CLI 横幅、配置路径和环境变量名称。

## 路径解析

三种执行模式：npm 安装、独立二进制、从源代码 tsx。

**始终对包资源使用 `src/config.ts`**：

```typescript
import { getPackageDir, getThemeDir } from "./config.js";
```

永远不要直接对包资源使用 `__dirname`。

## 调试命令

`/debug`（隐藏的）写入 `~/.pi/agent/pi-debug.log`：
- 带有 ANSI 代码的渲染 TUI 行
- 发送到 LLM 的最后消息

## 测试

```bash
./test.sh                         # 运行非 LLM 测试（不需要 API 密钥）
npm test                          # 运行所有测试
npm test -- test/specific.test.ts # 运行特定测试
```

## 项目结构

```
packages/
  ai/           # LLM 提供者抽象
  agent/        # 代理循环和消息类型  
  tui/          # 终端 UI 组件
  coding-agent/ # CLI 和交互模式
```
