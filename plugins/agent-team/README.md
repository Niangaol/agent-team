# agent-team 插件

一支覆盖成熟工程团队完整环节的智能体团队，通过 Step Code 插件市场（路线 B）分发，附带 team-gate 协作门禁（A2A）。**v2.0.0**：server 重构为三层架构，新增按团队分组、多进程安全的状态持久化、stale 检测。

## 十二个角色

| 环节 | 角色 | 职责 | 工具边界 |
| - | - | - | - |
| 需求澄清 | analyst | 把模糊需求变成可判定的验收标准、边界与影响面 | 只读 |
| 代码调研 | scout | 定位文件、调用链、现有约定，带证据 | 只读 |
| 缺陷定位 | debugger | 复现收敛、二分定位、根因证据链（不修） | 只读 + 命令 |
| 方案设计 | architect | 编号执行步骤，每步带完成标准与回滚 | 只读 + 命令 |
| 编码实现 | builder | 按方案精确实现并自验 | 读写 |
| 测试保障 | tester | 基线 + 补测，验证测试会红，只许改测试 | 读写（限测试） |
| 功能审查 | reviewer | 正确性/回归/约定，分级问题清单 | 只读 + 命令 |
| 安全审计 | security-auditor | 注入、越权、密钥、敏感数据、依赖风险 | 只读 + 命令 |
| 性能把关 | perf-engineer | 热点、复杂度、IO，用测量数字说话 | 只读 + 命令 |
| 界面审查 | ux-reviewer | 状态覆盖、交互反馈、文案、可访问性 | 只读 |
| 文档同步 | docs-writer | README/API/CHANGELOG 与代码事实一致 | 写文档 |
| 交付发布 | release-engineer | 构建验证、版本检查、发布清单与回滚方案 | 读写交付文件 |

附带 `team-playbook` skill（环节映射 + 五种编排模式）与 `/team-implement`、`/team-debug`、`/team-review`、`/team-ship` 四条命令。

## team-gate：六个协作工具

零依赖 stdio MCP server（Node ≥ 18），全部工具接受 `team` 参数做按团队分组：

| 工具 | 使用者 | 作用 |
| - | - | - |
| `agent-team__team-gate__team_wait` | 主智能体 | **主动暂停运行**，阻塞等待任务完成信号。`expect: [标签...]` 等全部；`mode: "any"` 等任意一个；`timeoutMs` 超时返回 pending + **stale 失联清单**；`reset` 只清本团队 |
| `agent-team__team-gate__agent_done` | 子智能体 | 发送**任务完成信号**（label / status / summary / artifacts） |
| `agent-team__team-gate__agent_report` | 子智能体 | 注册/更新自己的存在与**当前任务**（upsert，定期调用可续期防失联） |
| `agent-team__team-gate__team_status` | 任何 agent | **实时总览**：本团队 label、角色、当前任务、状态、最后更新、stale 清单、主代理等待中的标签、所有团队规模 |
| `agent-team__team-gate__agent_send` | 任何 agent | **A2A 消息**：点对点（to=对方 label）或广播（to="*"），仅本团队内投递 |
| `agent-team__team-gate__agent_recv` | 任何 agent | 读取发给我的未读消息（含广播），默认标记已读 |

工具名的限定名规则来自宿主：插件 MCP 服务以 `<插件id>__<服务名>` 注册（这里是 `agent-team__team-gate`），
工具再拼上服务内的工具名。不同宿主构建的拼接方式可能不同，以 `/mcp` 中实际列出的名字为准。

## 架构（v2.0.0 三层，零依赖）

```text
agent-team/server/
├── state.mjs      状态层：按团队分组的存储 + 多进程安全持久化
│                  · 每个 team 独立 signals/presence/messages
│                  · 落盘 <插件目录>/state/teams.json（TEAM_GATE_STATE_FILE 可覆盖）
│                  · 写入走「状态文件锁 + 读-改-写 + 同目录原子替换」
│                  · 读取按 mtime/size 判断是否重读磁盘，能看到别的进程刚写的状态
│                  · 团队数上限 50（淘汰最久未更新）；损坏文件备份后从空启动
├── tools.mjs      工具层：六个工具的定义与实现，team 参数解析、stale 检测
└── team-gate.mjs  协议层：MCP over stdio JSON-RPC 2.0 收发（stdout 只走协议）
```

**设计取舍**：
- **按团队分组**：并发跑多个团队时 label 不再串扰；`reset` 只影响当前团队；`team_status` 的 `teams` 字段可发现其他并行团队。分组是**状态分区**，不是安全隔离：同一服务里所有团队的数据在同一个文件中，任何能调用工具的一方都能读到其他团队的摘要。
- **多进程共用一份状态**：宿主可能为不同会话各起一个 `team-gate` 进程，也可能复用同一个；两种情况都只写同一个状态文件。写入一律经过文件锁和读-改-写，进程之间不会用旧快照覆盖对方。验证见 `tests/two-session-state.mjs`。
- **持久化**：进行到一半的团队在 MCP server/应用重启后恢复，信号、在线状态、消息不丢。
- **stale 检测**：子代理超过 `staleAfterMs`（默认 5 分钟）未 `agent_report` 即判失联，`team_wait` 超时返回时列出，避免主代理干等；长任务中子代理定期 `agent_report` 续期。

## A2A 协作模型

- **实时可见**：每个子代理开始执行时 `agent_report` 注册，阶段推进时更新 `task`；任何 agent 随时 `team_status` 看到谁在做什么、什么状态、主代理在等谁。
- **直接通信**：结论、依赖、警告用 `agent_send` 点对点或广播；接收方在动手前与关键节点 `agent_recv` 读取。消息用于协调，不替代各自的任务输出。
- **里程碑语义（关键）**：`agent_done` 信号标记的是「分配的必要任务已完成」这个里程碑，**不是子智能体终止**。子智能体发完信号后继续执行后续工作；主智能体 `team_wait` 收齐信号即恢复运行并汇总。同一子智能体可用不同 label 发多次里程碑信号，主智能体用 `mode=any` 渐进式汇总。
- **补充信息**：要向仍在运行的子智能体追加指令，single 模式后台通道用 `agent_send` 的 `reply`（内置工具，非本插件）；其他情况重新派发并沿用原 label。

**典型流程**：

1. 主智能体为本次任务分配团队名（如 `impl-1`），派发时把 label 与 team 写进任务文本。
2. 子智能体 `agent_report` 注册 → 执行中随时 `team_status`/`agent_recv` 掌握全局 → 必要时 `agent_send` 通知同伴。
3. 子智能体完成必要任务 → `agent_done` → 继续执行。
4. 主智能体 `team_wait` 暂停，收齐期望 label 的信号后恢复，按 status/summary 汇总。

## 编排模式

| 模式 | 角色链 | 命令 |
| - | - | - |
| 全流程 | analyst → scout → architect → builder → tester → reviewer（按需叠加安全/性能/UI/文档） | `/team-implement` |
| 缺陷修复 | debugger → builder → tester → reviewer | `/team-debug` |
| 并行审查 | reviewer / security-auditor / perf-engineer / ux-reviewer 多视角并行 | `/team-review`、`/team-ship` 前半 |
| 发布把关 | 四路并行审查 → docs-writer → release-engineer | `/team-ship` |

## 安装

在 Step Code 会话中把本目录加入市场源并安装：

```text
/plugin marketplace add ./agent-team
/plugin install agent-team
```

或在插件管理菜单 `/plugin` 中操作。**本插件含 MCP server，安装后需重启 Step Code**，再用 `/mcp` 确认 `team-gate` 已加载（应看到上表六个工具）。

## 目录结构

```text
agent-team/
├── step.plugin.json          # 插件清单（id 必填，含 mcpServers）
├── server/                   # team-gate 三层架构（state / tools / protocol）
├── agents/                   # 12 个角色定义（Markdown + frontmatter）
├── skills/team-playbook/     # 编排手册（环节映射 + A2A + 信号语义 + 团队分组）
└── commands/                 # team-implement / team-debug / team-review / team-ship
```

## 费用

- **本地 MCP 不单独收费**：`team-gate` 是进程内实现，只做内存与本地文件操作，不连网、不调用任何模型或外部服务，不消耗账户额度，也不产生计费。
- **子代理的模型调用另计**：派发明细里每个角色都是一次真实的模型会话，会消耗宿主账户的额度/套餐，和本地 MCP 是否收费无关。角色越多、并行度越高，消耗越大。默认建议控制在 5 个以内，扩大前先确认值得。

## 本地数据与清理

**保存什么**：`<插件目录>/state/teams.json` 保存团队协调状态——
- 任务摘要：`agent_done` 的 `summary`（截断到 8000 字符）与 `artifacts` 产出路径清单（最多 100 条）
- 消息：`agent_send` 的点对点/广播内容（截断到 8000 字符，每团队最多 200 条）
- 在线状态：label、角色、当前任务描述（截断到 2000 字符）、状态、时间戳
- 信号：每个 `agent_done` 的 label / status / summary / artifacts / 时间戳（每团队最多 200 条）

**不保存什么**：不写会话正文、不写模型输出原文、不写文件内容、不写凭据。产出路径只是字符串记录，插件不会去读取那些路径。

**写在哪**：默认插件安装目录下的 `state/teams.json`；设置环境变量 `TEAM_GATE_STATE_FILE` 可改到别处。运行时会伴随生成同目录的 `.lock`（写入期间的短时存在）与 `<文件>.<pid>.tmp`（原子替换的临时文件，正常写入后即删除）。

**清理**：
1. 停用/卸载插件后重启会话，MCP 进程停止。
2. 删除 `<插件目录>/state/` 整个目录（等价于只删 `teams.json` + 可能残留的 `.lock` / `.tmp` / `.corrupt-*`）。这一步不需要先启动插件。
3. 若改过 `TEAM_GATE_STATE_FILE`，删那个路径。
4. 卸载不会自动删除这些文件——宿主只清理它自己登记的安装资源。想彻底清干净需要手动删上一步的目录。

## 安全说明

`server/*.mjs` 是 stdio MCP 服务器，按当前用户权限执行，代码零第三方依赖，只做进程内状态记录与本地状态文件写入，不访问网络、不读取工作区文件、不请求凭据。角色 `tools:` 边界由宿主强制（见上表说明），但带 `run_command` 的角色拥有完整 shell 权限。安装前可自行阅读全部源码。

## 兼容范围

- **实际验证**：DFSC 0.4.0，macOS arm64——插件加载、`team-gate` 六个工具注册与调用、团队分组、完成信号、状态查询与消息收发跑通。
- **Windows 已验证**：仓库内 `tests/two-session-state.mjs` 在 Windows（Node.js 24，Git Bash）跑通双进程共用状态文件的六个场景。这是 **server 层**的验证，**不等于** Windows 上的 DFSC 宿主加载已验收。
- **未验证**：其他 DFSC 版本与其他平台的宿主加载差异；Windows 下 DFSC 宿主的实际安装与 `/mcp` 挂载；跨机协作（状态仅存本机）；高并发与长时间运行下的表现。

## 排错

- `/mcp` 里看不到 team-gate：确认已重启；确认 Node 在 PATH 且版本 ≥ 18。
- 若相对路径 `./server/team-gate.mjs` 解析失败：把 `step.plugin.json` 中 `mcpServers.team-gate.args` 改为该文件的绝对路径。
- `team_wait` 一直超时：看返回里的 `stale`（失联标签）与 `recentSignals`，核对 label 与 `team` 是否一致。
- 状态异常：删除 `<插件目录>/state/teams.json` 后重启即可从零开始（不影响插件功能）。
- 看不到某个 agent：它还没 `agent_report`；消息没收到：确认 `agent_recv` 的 label 与发送时 `to` 一致、team 一致，或消息已被标记已读。
