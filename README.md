# feishu-companion-agent

用 Hermes Agent + 飞书自建应用搭建知识库伴学/答疑机器人，接入飞书私聊的完整方法论。

## 能做什么

- 飞书私聊机器人：基于本地知识库作答，引用出处，给落地动作
- 学员画像：一人一档，对话自动建档与更新，跨会话记忆
- 伴学 SOP：定档 → 路线图 → 陪跑 → 复盘
- DM 配对审批：白名单控制谁能用
- 附：飞书 wiki 文档 API 全文抓取（网页只能看半篇的问题一并解决）

## 快速开始

1. 前置：Hermes Agent 已装、模型已配、飞书自建应用（机器人能力 + 长连接模式）凭证在手
2. 按 SKILL.md 的 6 步搭建：目录结构 → AGENTS.md → 凭证写入 → cwd 指向 → gateway 启动 → 配对验收
3. 多人放量：应用可用范围 + 逐人 `hermes pairing approve`

## 文件

- `SKILL.md` — 完整搭建流程 + 踩坑记录（WorkBuddy/Hermes 通用 skill 格式，可直接放 `~/.workbuddy/skills/`）

## 实测环境

- macOS + Hermes Agent v0.20.0 + 飞书自建应用（WebSocket 长连接）
- 已跑通：知识库加载、伴学问答、飞书收发、配对审批全链路
