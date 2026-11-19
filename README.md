# agent-team

十二角色智能体团队插件（Step Code / DFSC 插件），附带 team-gate A2A 协作门禁。

插件资源位于 [plugins/agent-team](plugins/agent-team/)，包含：

- `step.plugin.json` — 插件清单（id `agent-team`，v2.0.0）
- `dfsc.plugin.json` — 市场展示声明（作者、版本范围、外部准备）
- `agents/` — 十二个角色定义（analyst / scout / debugger / architect / builder / tester / reviewer / security-auditor / perf-engineer / ux-reviewer / docs-writer / release-engineer）
- `skills/team-playbook/` — 编排手册
- `commands/` — /team-implement、/team-debug、/team-review、/team-ship
- `server/` — team-gate 零依赖 stdio MCP server（Node.js ≥ 18）

安装方式：在 DFSC 的社区市场 `dfsc-community` 中安装，或按 Step Code 插件格式从本仓库 `plugins/agent-team` 目录加载。

License: MIT
