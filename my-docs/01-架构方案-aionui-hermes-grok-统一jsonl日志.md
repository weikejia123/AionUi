# AionUi + Hermes + Grok 三合一架构方案评估

> **分析日期**：2026-08-09 00:49（与 coder-agent 全目录对比分析同日）
> **版本**：V1-20260809
> **结论依据**：全部来自源码实证（AionUi 2.1.52 / hermes-agent 本地源码 / grok-build 快照），不采信宣传文档
> **关联文档**：`../ANALYSIS-DIR-CODER-AGENT.md`（8 个 agent 全量对比）

---

## 0. 结论先行

**方向正确，是当前可选范围内最优解之一，可以按此实施。** 打分：

| 维度 | 评分 | 说明 |
|---|---|---|
| AionUi 作总驾驶舱 | ⭐⭐⭐⭐⭐ | 这就是它的设计本分（ACP 控制平面），且三方能全部验证 ACP 兼容 |
| Hermes 作通用工作 | ⭐⭐⭐⭐⭐ | 它是唯一深度绑定你知识体系（.gshare/SOUL.md/skills）的 agent，通用工作主场 |
| Grok 作开发工作 | ⭐⭐⭐⭐ | 开发能力顶级；唯一缺口是**快照无浏览器自动化**（见 §5） |
| 统一 JSONL 日志 | ⭐⭐⭐ | **AionUi 原生没有 JSONL 导出**（v2.1.15 起隐藏了导出 UI），需自建一层（§3.3 给了三种方案）|

**两个必须知道的修正**：
1. "三合一"实际是"**驾驶舱（AionUi）+ 两个 ACP agent（Hermes、Grok）**"——AionUi 不跑自己的 agent 引擎，智能全部来自后两者。这正是它设计的用法。
2. 统一 JSONL 日志**不能开箱即得**，需要在 ACP 层或 AionUi API 层自建一个小工具（工作量不大，§3.3）。

---

## 1. 为什么是最优：三方代码实证

### 1.1 AionUi：天然的多 agent 驾驶舱（非 agent 本体）
- `common/types/agent/detectedAgent.ts`：执行引擎抽象，`acp` 类 agent 通过 `cli_path` + `acpArgs` 接入任意 ACP 兼容 CLI。
- **设置 → Agents → Add custom Agent**（`AgentSettings/LocalAgents.tsx` + `InlineAgentEditor.tsx`）：手动填 `command`（如 `/usr/local/bin/hermes-acp`）+ `args` + 环境变量 + 高级 JSON，后端 aioncore 托管 CRUD（`/api/agents/custom*`），自带连接测试（`/api/agents/custom/try-connect`：CLI 检测 + ACP 握手）。
- 扩展可贡献 acpAdapters（`aion-extension.json` 的 `contributes.acpAdapters`）；Team 模式、cron、WebUI/IM 远程、应用内浏览器全部对任意 agent 生效。

**结论**：Hermes 和 Grok 各填一个 Custom Agent 即可被 AionUi 统一驱动，零改 AionUi 代码。

### 1.2 Hermes：完整 ACP 适配器（已具备，非规划中）
`~/.hermes/hermes-agent/acp_adapter/`（**5831 行**，Python 官方 `acp` 包）：
- `server.py`（ACP server：Initialize/NewSession/ForkSession/LoadSession 全生命周期）、`session.py`、`permissions.py`、`auth.py`、`provenance.py`、`edit_approval.py`、`tools.py`、`events.py`。
- **工具映射是全量 Hermes 工具集**（`tools.py` 的 TOOL_KIND_MAP + build_tool_title 覆盖）：terminal / read_file / write_file / patch / search_files / web_search / web_extract / process / delegate_task / session_search / memory / execute_code / todo / skill_view / skills_list / skill_manage / browser_navigate / browser_snapshot / browser_vision / vision_analyze / image_generate / cronjob……
- 入口：`python -m acp_adapter`（`entry.py` 带 `--version/--check/--setup` 等参数）。

**结论**：Hermes 的通用能力（知识库、skills、记忆、浏览器、图像生成）接入 AionUi 后基本不丢失。⚠️ 待实测项：ACP 会话下 SOUL.md 人格注入是否完整（见 §5）。

### 1.3 grok-build：官方 ACP 支持（xAI 原生）
- `xai-grok-shell/src/agent/app.rs:277`：`pub async fn run_stdio_agent(...)`——ACP stdio agent 模式。
- 全链路用官方 `agent_client_protocol`（ACP）crate（`acp_handler/mod.rs`：`use agent_client_protocol as acp;`，含 ACP 消息路由、权限请求队列、session 通知）。

**结论**：grok-build 以 `grok agent stdio`（或等价命令）接入 AionUi 是官方支持的路径。

### 1.4 为什么是这两个 agent（对照全目录 8 个）
| 场景 | 首选 | 备选 | 选它的理由 |
|---|---|---|---|
| 通用工作 | **Hermes** | penguin / pi | 唯一深度绑定你个人知识体系；ACP 适配器现成；其余通用型（penguin 无记忆/pi 极简）能力不足 |
| 开发工作 | **Grok** | jcode / claude-code | xAI 官方、工程顶级（7000+ 测试）、autoDream 记忆 + Imagine 生成；备选仅在"需要浏览器自动化/多 provider"时考虑（§5）|

> "只要最好的"：两 agent 分别是各自场景的顶级选手，且 1+1 覆盖了"通用 + 开发"全场景，无冗余。

---

## 2. 推荐架构

```
┌────────────────────────────────────────────────────────────┐
│  AionUi（总驾驶舱 · Electron/Web/移动）                      │
│  · 统一聊天界面 · Team 协作 · cron 定时 · 远程 IM/WebUI       │
│  · 会话/审批/权限统一管理                                    │
└───────┬────────────────────────────┬───────────────────────┘
        │ ACP (stdio JSON-RPC)       │ ACP (stdio JSON-RPC)
        ▼                            ▼
┌───────────────┐          ┌────────────────────┐
│  ACP 日志代理   │──JSONL──▶│ 统一工作日志存储    │
│ (可选，§3.3-方案A)│          │ ./agent-logs/*.jsonl│
└───────┬───────┘          └────────────────────┘
        ▼
┌──────────────────┐   ┌──────────────────────────┐
│ Hermes (通用工作)  │   │ grok-build (开发工作)     │
│ python -m acp_adapter│  │ grok agent stdio        │
│ 知识库/skills/记忆  │   │ 文件/终端/LSP/代码图/记忆  │
└──────────────────┘   └──────────────────────────┘
```

- **AionUi**：总驾驶舱（UI + 编排 + 权限 + 远程 + 定时），不参与智能。
- **Hermes**：通用工作（文档/知识/日常/研究），吃自己知识体系。
- **Grok**：开发工作（编码/调试/重构/代码库理解）。
- **统一日志**：所有 ACP 流量 + 会话记录落 JSONL（方案见下）。

---

## 3. 统一 JSONL 日志：三种方案（关键决策）

**现状（代码实证）**：
- 会话数据存在 **aioncore 后端 SQLite**（前端不落盘，仅内存态 + REST 拉取）。
- 消息统一数据形态 `TMessage` 已存在（14 种：text/tips/tool_call/tool_group/agent_status/permission/acp_permission/ask/acp_tool_call/plan/thinking/available_commands/acp_terminal_output），字段 `id/msg_id/conversation_id/type/content/created_at/position/status/backend_turn_id`。
- **无 JSONL 导出 UI**：纯文本导出（`useConversationExport.tsx`）和 ZIP JSON 导出（`exportHelpers.ts`）都已被 v2.1.15 隐藏（`SendBox/index.tsx:475` "intentionally not registered"、`GroupedHistory` 的 onExport 未接线）。
- 全仓 `jsonl` 零命中（与会话无关）。

### 方案 A：ACP 透传代理日志（推荐 · 格式最统一）
在所有 agent 的 ACP stdio 前插一层代理：AionUi 连接代理，代理转发到真实 agent，同时把**全部 ACP JSON-RPC 消息**（initialize/NewSession/message/thought/tool_call/tool_result/permission 等）逐行追加到 `<agent>/<session>/<timestamp>.jsonl`。

- ✅ 格式天然统一：所有 agent 都走 ACP，日志 schema 一模一样，跨 agent 可比。
- ✅ 不依赖 AionUi 内部实现；agent 视角的完整工作记录（含 reasoning、工具调用、结果）。
- ⚠️ 不含 AionUi 层信息（Team 编排、AionUi 生成的系统消息）——需要的话叠加方案 C。
- 实现量：一个 ~200 行脚本/小服务（stdio 双向转发 + JSON 落盘），可选保留心跳/会话边界。

### 方案 B：AionUi REST 拉取导出（依赖官方 API）
写小工具定时调 `GET /api/conversations/{id}/messages?content_mode=full`（cursor 分页，前端已有 `loadAllConversationMessagesPaged` 现成逻辑可参考），把 TMessage 全量转 JSONL。

- ✅ 含 AionUi 层信息（权限、ask、plan、team）。
- ⚠️ 仅含落库消息：`acp_terminal_output` 是 stream-only 不持久化，agent 内部 reasoning 也未必全量落库——**"工作记录"会缺 agent 内部细节**。

### 方案 C：A+B 双份（最完整）
ACP 层完整流量（agent 视角）+ AionUi 层会话导出（驾驶舱视角），按 `conversation_id + backend_turn_id` 关联。

**推荐**：先上方案 A（满足"所有 agent 工作记录统一 JSONL"的核心诉求，格式统一、实现简单），有需要再补 B/C。方案 B 的 API 已存在，随时可加。

---

## 4. 实施步骤（按顺序）

1. **接入 Hermes**：设置 → Agents → Add custom Agent → command=`python -m acp_adapter`（或打包后的可执行入口），args 按需；点连接测试验证 ACP 握手。
2. **接入 Grok**：同上，command=`grok`，args=`agent stdio`（以本机 `grok --help` 实测为准）。
3. **建立 ACP 日志代理**：方案 A 实现，先跑 Hermes 验证 JSONL 落盘质量，再接入 Grok。
4. **验证统一性**：对比 Hermes/Grok 两条日志的 schema 一致性；确认 reasoning/tool_call/tool_result 全捕获。
5. **补驾驶舱层**（可选）：方案 B 的导出工具，定时归档 AionUi 会话 JSONL。
6. **运行验证**：Team 模式同时调 Hermes+Grok，确认 AionUi 权限弹窗/事件流对两 agent 都正常。

---

## 5. 风险与注意点（代码实证的坑）

| # | 风险 | 详情 | 缓解 |
|---|---|---|---|
| 1 | **grok-build 快照无浏览器自动化** | 本仓库快照不含 `xai-grok-browser-tools` crate（tests 有引用但代码缺失）；开发中要浏览器场景会缺 | 用 Hermes 的 browser_* 工具兜底；或后续把 claude-code/jcode 加为第三个 agent（它们有 Chrome/Firefox 浏览器控制）|
| 2 | **grok 默认模型 grok-4.5 + BYOK** | 无预置第三方 provider（要配 OpenAI 兼容端点走 BYOK）| 开发场景 grok-4.5 足够；特殊模型走 `ModelProviderConfig` |
| 3 | **Hermes ACP 会话的人格/知识注入待实测** | ACP 适配器工具映射全，但 SOUL.md 人格、.gshare 知识召回是否在 ACP 会话完整生效需实测 | 接入后跑一个"知识问答 + 人格风格"冒烟测试 |
| 4 | **AionUi 自定义 agent 是后端托管** | Custom Agent 配置存 aioncore 后端（`agent_metadata` 表），不是本地文件 | 换机器/重装需重新配置或备份该表 |
| 5 | **Agent Hub 未接线** | `AgentHubModal` 组件存在但无页面挂载（开发中）| 不依赖它，用 Custom Agent 手动接入 |
| 6 | **JSONL 需自建** | AionUi 无导出 UI（v2.1.15 隐藏）| 方案 A 代理 ~200 行可实现 |
| 7 | **版本漂移** | grok-build 是快照（0.2.105），Hermes 本地可编辑安装 | 定期同步上游（参照 wkj-dev 流程）|

---

## 6. 一句话总结

**AionUi（驾驶舱）+ Hermes（通用）+ Grok（开发）是当前可选范围内的最优组合**：三方 ACP 兼容全部代码实证、无冗余、各自主场顶级；唯一要动手的是自建一个 ACP 层 JSONL 日志代理（~200 行），以及实测 Hermes 接入后的人格/知识注入完整性。若开发工作强依赖浏览器自动化，再考虑把 claude-code/jcode 作为第三 agent 补充。
