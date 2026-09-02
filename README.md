# wechat-miniprogram-devtools-qa

微信小程序在开发者工具里的协议优先验收与调试 AI agent skill。

用 CLI + `miniprogram-automator` 做 DOM 几何/路由/API 流程验证，Computer Use 仅作最后兜底。每次验证前必须走 clean-build gate（项目级清缓存 + 重新编译），不接受用构建成功或源码审查代替实际运行验收。截图、DOM 几何、API 响应、服务端日志都算证据，但单独一个不够——要交叉验证。

和 rebuild-miniprogram（重建）、wechat-devtools-automation-recovery（恢复）组成小程序开发验证的完整链路。

## 安装

把 `SKILL.md`（及 `agents/`、`references/` 子目录如有）复制到你的 agent skill 目录下即可。

## License

MIT
