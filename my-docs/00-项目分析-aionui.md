# AionUi 项目分析

> 分析日期：2026-08-03 ｜ 版本：2.1.45 ｜ 上游：iOfficeAI/AionUi（31.3k stars）

## 项目基本信息

| 维度 | 内容 |
|------|------|
| 名称 | AionUi |
| 组织 | iOfficeAI |
| 语言 | TypeScript |
| License | Apache-2.0 |
| 版本 | 2.1.45 |
| 技术栈 | Electron 37 + React 19 + Arco Design + UnoCSS + Bun + Vitest 4 + Oxlint/Oxfmt |
| 平台 | macOS / Windows / Linux + Web + Mobile |
| 包结构 | `packages/desktop`（Electron 主端）、`packages/web-cli`、`packages/web-host`、`packages/shared-scripts`、`mobile/` |

## 一、使用场景

AionUi 是一个 **AI Agent 协作（Cowork）工作台**，解决的核心痛点是：当你的机器上装了多个 CLI Agent（Claude Code、Codex、OpenCode、Hermes Agent、Gemini CLI、Qwen Code、Snow CLI……）时，每个 Agent 各自一个终端、一套配置、一种交互方式，无法统一查看、切换、遥控。

它提供的场景：

1. **统一多 Agent 界面** — 一个桌面 App 内接入 20+ CLI Agent，自动检测本机已安装的 Agent，统一界面操作
2. **内置 Agent 零配置开跑** — 自带完整 Agent 引擎（文件读写、网页浏览、MCP 工具、图片生成），粘贴任意 API Key 即用，不需要先装 CLI
3. **远程访问** — WebUI + Telegram / Lark（飞书）/ DingTalk / WeChat 多渠道，手机随时看 Agent 干活
4. **24/7 定时自动化** — Cron 计划任务，无人值守跑
5. **办公文档生成** — 内置 21 个专业助手：PPT Creator、Word Creator、Excel Creator、Morph PPT（动画过渡 PPT）、Academic Paper Writer 等，输出可编辑的 `.pptx/.docx/.xlsx`（底层调用同组织的 OfficeCLI）

## 二、特性与技术亮点

**1. Agent 客户端协议抽象（`@agentclientprotocol/sdk`）**

这是全项目最有价值的设计——它不是为某个 Agent 写适配器，而是定义了一套**统一的 Agent 客户端协议**（Agent Client Protocol，ACP）。20+ CLI Agent 通过同一协议接入，新增一个 Agent 不需要重写 UI 层。这解释了为什么它能同时兼容 OpenClaw、Hermes Agent、Claude Code 这些架构完全不同的 Agent。

**2. 双进程架构纪律**

```
Main 进程（packages/desktop/src/process/）  — 无 DOM API
Renderer（packages/desktop/src/renderer/）  — 无 Node.js API
        ↕ 只能通过 IPC 桥（src/preload/）通信
```

Electron 项目最常见的腐化就是进程边界模糊，AionUi 用 AGENTS.md 硬性约束 + lint 强制这个边界，作为贡献者的"Hard blockers"之一。

**3. 内置 Agent + 外部 CLI 双轨**

- 内置引擎：零配置，粘贴 API Key 即用
- 外部 CLI：自动检测已装 Agent，统一接入
两条轨道共存，覆盖"新手直接跑"和"老手接自己的 Agent"两个场景。

**4. 办公文档交付链**

PPT/Word/Excel 不是简单聊天输出，而是走 **OfficeCLI** 生成可编辑文件——这跟 my-agent-group 里用 Pandoc/Python 生成交付文档的思路一致，但打包成了产品级能力。

**5. 工程规范密度极高**

- 目录 ≤10 直接子项硬限制
- 覆盖率目标 ≥80%，普通改动必须带测试
- `just push`（lint → format → typecheck → test → push 全链）
- Conventional Commits + PR 模板 + prek 复刻 CI 检查
- 自带 `.claude/skills/`（architecture/i18n/testing/bump-version）——项目规范以 Agent 技能形式沉淀，Agent 自动遵守

## 三、同类产品对比

| 项目 | 定位 | 与 AionUi 差异 |
|------|------|---------------|
| **Claude Cowork**（Anthropic 商业版） | 桌面协作 App，内置 Claude | 闭源、单 Agent、绑定 Claude。AionUi 是开源替代，多 Agent + 自带引擎 + 远程 |
| **OpenClaw** | 个人助理型 Agent 框架 | 是"被接入的 Agent"而非"工作台"。AionUi 将 OpenClaw 列为支持对象 |
| **Zed / Cursor** | 代码编辑器内嵌 Agent | 编辑器中心。AionUi 是独立工作台，面向 Agent 而非代码文件 |
| **pi / kimi-code / claude-code**（本项目已有） | CLI Agent 本体 | 同类：AionUi 是它们的"驾驶舱"。pi、kimi-code 都可以被 AionUi 接入 |

**定位总结**：AionUi 不是又一个 Agent，而是 Agent 们的**控制平面**——把散落在各终端的 CLI Agent 收拢到一个界面，补上远程访问和定时任务。

## 四、对 Hermes 和 my-agent-group 的启发

1. **Hermes 被官方列为支持对象** — AionUi 的 README 明确列出 Hermes Agent。这意味着 Hermes 用户可以用 AionUi 作为图形界面。对我们来说，它是 Hermes 的**可选项装前端**，值得在评估 Hermes 生态时纳入。

2. **Agent Client Protocol 值得学习** — my-agent-group 管理着多个 Agent 项目（pi、kimi-code、deer-flow、hindsight），如果未来做统一接入层，ACP 的协议抽象思路（而非逐 Agent 适配）是正确方向。

3. **办公文档助手验证了我们的方向** — my-agent-group 已有 docx/pandoc/xlsx/fapiao-expense 等文档技能。AionUi 把同类能力产品化（Morph PPT、Word/Excel Creator），其"文档技能 + OfficeCLI 渲染管线"的分层值得对照。

4. **工程规范以 Agent 技能沉淀** — `.claude/skills/` 里放 architecture/i18n/testing 规范，Agent 干活时自动遵守。这与 Hermes 的 skill 体系同构，是一个"项目级规范如何让 AI 自动执行"的好案例。

## 附：硬件与运行

- 桌面端 Electron 37（Chromium），内存占用取决于打开的面板数与 Agent 数量，常规使用 1-2GB 级（与 VS Code 同量级）
- 无 GPU 硬性要求；WebUI/手机端仅需一台常开主机
- 仓库体积 ~690MB（含 mobile、resources 等），`--depth 1` 克隆即可
