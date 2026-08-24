---
name: feishu-companion-agent
description: 用 Hermes Agent + 飞书自建应用搭建「知识库伴学/答疑机器人」并接入飞书私聊。当用户要做飞书机器人、AI 伴学助手、知识库问答 bot、或要求把本地 Agent 接入飞书时使用。覆盖知识库结构设计、伴学提示词、飞书凭证配置、gateway 启动、DM 配对审批全流程。
---

# 飞书伴学 Agent 搭建（Hermes + 飞书 CLI）

把本地知识库变成飞书里的伴学/答疑机器人：学员私聊即可提问，Agent 基于知识库作答并维护学员画像。

## 架构

```
飞书私聊 → Hermes gateway（WebSocket 长连接，免公网 IP）→ 伴学引擎（AGENTS.md SOP + 画像）
                                                        → knowledge/（结构化知识库）
```

## 前置条件

- 已安装 Hermes Agent（`~/.hermes/hermes-agent`），`hermes --version` 可用
- 模型已配置（`~/.hermes/config.yaml` 的 `model:` 段）
- 飞书自建应用：开放平台创建 → 添加「机器人」能力 → 事件订阅选**长连接模式** → 拿到 `App ID` / `App Secret`

## 搭建步骤

### 1. 建 Agent 目录结构

```
<项目>/_agent/
├── AGENTS.md          # 伴学人设 + SOP（核心）
├── knowledge/         # 知识库
│   ├── KNOWLEDGE.md   # 总索引：资料地图 + 框架摘要 + 重资产登记（唯一入口）
│   ├── K1-xxx.md      # 原始资料（md 最佳，按 K1/K2…编号）
│   └── ...
├── profiles/          # 学员画像（一人一文件，飞书用户自动隔离）
│   └── <用户>.md
└── README.md          # 交付说明 + 验收步骤
```

**KNOWLEDGE.md 索引表三列**：编号 / 文件 / 什么时候读。再加一节「课程核心框架摘要」——先给结论级骨架，细节回原文，这是省 token 的关键。

### 2. 写 AGENTS.md（伴学 SOP 四阶段）

```markdown
# 角色：XXX 伴学官
## 每次对话固定动作
1. 先读画像 profiles/<用户>.md；无画像则 3-5 问建档写回
2. 答疑先查 knowledge/KNOWLEDGE.md 定位再读原文；输出「结论→依据（哪份资料哪节）→落地动作」
3. 伴学不止答疑：发现认知偏差用课程心法点破，一次一个
4. 对话结束前更新画像
## 伴学 SOP：定档（新手/入门/进阶）→ 路线图 → 陪跑（一次一步，做完验收再给下一步）→ 复盘（对照交付标准）
## 边界：不虚构（资料没有就说明）；付费内容不复述全文
```

### 3. 写入飞书凭证

```bash
~/.hermes/hermes-agent/venv/bin/python -c "
import sys; sys.path.insert(0, '/Users/<user>/.hermes/hermes-agent')
from hermes_cli.config import save_env_value
save_env_value('FEISHU_APP_ID', '<app_id>')
save_env_value('FEISHU_APP_SECRET', '<app_secret>')
save_env_value('FEISHU_DOMAIN', 'feishu')
save_env_value('FEISHU_CONNECTION_MODE', 'websocket')
"
```

存入 `~/.hermes/.env`。注意：必须用 hermes 自带 venv 的 python（依赖 yaml），系统 python3 会报 ModuleNotFoundError。

### 4. 指定 gateway 工作目录

`~/.hermes/config.yaml` → `terminal.cwd: /绝对路径/_agent`。gateway 会话的 Agent 以该目录为工作区，自动读到 AGENTS.md 和知识库。

### 5. 启动与配对

```bash
hermes gateway run          # 前台跑；日志见 ~/.hermes/logs/gateway.log
# 看到 "[Feishu] Connected in websocket mode" 即成功

# 学员首次私聊发消息 → 机器人回 8 位配对码 → 管理员批准：
hermes pairing list                          # 查看待批
hermes pairing approve feishu <CODE>         # 批准
# 批准后该飞书用户进入 ~/.hermes/platforms/pairing/feishu-approved.json，永久生效
```

常驻：用户本人在 Terminal 跑 `hermes gateway install`（launchd 服务，沙箱内不可用）。

### 6. 验收

飞书私聊发真实业务问题 → 收到「结论+出处+动作」三段式回答 = 通过。

## 放量给多人

- 飞书层：应用「可用范围」圈定人员（跨组织需建团队拉人）
- Hermes 层：每人首次私聊出配对码，管理员逐个 approve
- 会话隔离：config.yaml `group_sessions_per_user: true`（默认已开），群聊每人独立上下文

## 附：飞书 wiki 文档全文抓取（入库利器）

应用凭证可直读飞书文档全文（wiki 公开网页只有部分内容）：

```bash
# 1. 取 token
TOKEN=$(curl -s -X POST "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal" \
  -H "Content-Type: application/json" \
  -d '{"app_id":"<id>","app_secret":"<secret>"}' | python3 -c "import json,sys;print(json.load(sys.stdin)['tenant_access_token'])")
# 2. wiki 链接尾缀 → obj_token
OBJ=$(curl -s "https://open.feishu.cn/open-apis/wiki/v2/spaces/get_node?token=<wiki_token>" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "import json,sys;print(json.load(sys.stdin)['data']['node']['obj_token'])")
# 3. 读全文
curl -s "https://open.feishu.cn/open-apis/docx/v1/documents/$OBJ/raw_content" -H "Authorization: Bearer $TOKEN"
```

## 踩坑记录（实测）

| 坑 | 解法 |
|---|---|
| nohup 后台跑 gateway 被回收 | 用工具级后台任务（run_in_background）或 `hermes gateway install` 常驻 |
| 系统 python3 写 env 报 no yaml | 用 `~/.hermes/hermes-agent/venv/bin/python` |
| pairing approve 报 not found | 码已被消费/过期，`hermes pairing list` 看实际状态；部分流程自动批准 |
| 首条消息 unauthorized | 正常：配对码流程走完才授权 |
| 网页抓飞书 wiki 只有半篇 | 走上面的 API 抓全文 |
