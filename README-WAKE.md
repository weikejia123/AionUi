# AionUi — AI Agent Cowork 工作台

| 维度 | 内容 |
|------|------|
| **用途** | 免费开源的本地 24/7 AI Agent 协作工作台。内置完整 Agent 引擎，同时统一接入 20+ CLI Agent（OpenClaw、Hermes Agent、Claude Code、Codex、OpenCode、Gemini CLI、Qwen Code 等），提供文件操作、网页浏览、定时任务、远程访问（WebUI + Telegram/Lark/DingTalk/WeChat）、内置 21 个办公助手（PPT/Word/Excel 创建） |
| **fork 源** | `iOfficeAI/AionUi` → `weikejia123/AionUi` |
| **标签** | coder-agent / TypeScript / Cowork / Multi-Agent / Desktop App |
| **技术栈** | TypeScript, Bun, Electron (packages/desktop), Arco Design, UnoCSS, Vitest 4, Oxlint/Oxfmt |
| **内部版本** | V1-20260803 |
| **关联** | Hermes Agent（官方支持列表内）、Claude Code、Codex、pi、kimi-code |

## 位置

`projects/coder-agent/AionUi/` — 按 fork 分类目录规范置于 coder-agent。

## Git 远程

- `origin` — `weikejia123/AionUi.git`（个人 fork）
- `upstream` — `iOfficeAI/AionUi.git`（官方上游）

## 分支

- `main` — 跟踪 upstream/main（纯同步，保持纯净）
- `wkj-dev` — 主开发分支（基于 upstream/main，本地文档/改动均在此分支）

## 目录速览

- `packages/desktop/` — Electron 桌面端（main 进程 `src/process/`、renderer `src/renderer/`、IPC 桥 `src/preload/`）
- `packages/` — 其余功能包
- `mobile/` — 移动端
- `.claude/skills/` — 项目内置 Agent 技能（architecture/i18n/testing/bump-version）
- `docs/` — 架构与贡献文档（含多语言 readme）

## 本地开发注意

- 包管理器：**Bun**（`bun.lock`），测试 Vitest 4（覆盖率目标 ≥80%）
- 提交规范：Conventional Commits，AI agent 推代码用 `just push`（lint → format → typecheck → test → push）
- 上游项目自带 AGENTS.md（自动注入本目录会话）与 CONTRIBUTING.md

## 分析报告

见 `my-docs/00-项目分析-aionui.md`。
