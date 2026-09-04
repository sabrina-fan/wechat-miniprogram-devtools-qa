# wechat-miniprogram-devtools-qa

协议优先的微信小程序验收与调试。通过自动化协议、API 探针和只读数据库检查来验证真实页面行为——Computer Use 仅作为最后的兜底手段。

## 为什么需要

截图、DOM 几何、API 响应和服务端日志才是小程序能否正常工作的真实证据。仅凭构建成功或源码审查不能算验收通过。这个 skill 强制执行清洁构建门禁、当前工作树溯源和分层验证链，确保不会把过时产物误当成可运行的软件。

## 安装

### 方式 A — 交给 agent 安装

把仓库地址给你的 agent，让它安装：

```
https://github.com/sabrina-fan/wechat-miniprogram-devtools-qa
```

### 方式 B — 手动安装

把 `wechat-miniprogram-devtools-qa/` 目录复制到你的 agent skill 目录下。

## 配置

- **开发者工具 CLI 路径**：macOS 下自动检测（`/Applications/wechatwebdevtools.app/Contents/MacOS/cli`）。
- **端口**：IDE HTTP 默认 `9420`，自动化 WebSocket 默认 `9422`，通过 `lsof` 自动发现。
- **miniprogram-automator**：如未安装，在临时工作区执行 `npm i miniprogram-automator`。
- **项目路径**：从 `package.json` 和项目结构自动检测，无需硬编码路径或 API key。

## 使用方法

需要在开发者工具里验证小程序行为时触发。skill 按以下验证链执行：

1. **清洁构建门禁** — 清 `file`/`compile` 缓存，从当前工作树重新构建，验证编译标记。
2. **服务栈健康** — 启动本地服务栈，检查健康端点。
3. **协议自动化** — 通过 `miniprogram-automator` 连接、导航、求值、截图。
4. **API 探针** — 有等效端点时，在 UI 操作之前先测 API。
5. **只读数据库探针** — 用纯 `SELECT` 查询定位数据边界。
6. **Computer Use 兜底** — 仅当所有协议路径都无法使用时。

## 兼容性

- **macOS** — 主要平台（使用 `lsof`、macOS 开发者工具 CLI 路径、Computer Use 驱动 GUI）。
- **Windows / Linux** — 需按平台适配 CLI 路径和进程检查命令。

## 安全与边界

这个 skill 只验证行为；除非明确要求，不写或改源码。绝不打印 `.env`、token、cookie、密码或用户标识。所有测试数据使用 mock/loopback 身份，测试结束后尽量清理。不使用生产数据。
