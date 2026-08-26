---
name: haoshu-agent
description: 通过 haoshu CLI 查询号薯账号、稿件、存稿/发布任务、分组/标签/代理 IP/稿件分类、员工（只读），支持 Cookie 导出与导入加号、账号元数据写、代理写、统计/未读刷新、稿件路径导入与改删、设封面/插配图、存稿/发布写操作。默认 Bridge 端口 19323（与号续 19321 区分）。不支持扫码登录、员工管理、删账号、公众号远端面板等。Use when the user asks to query or update HaoShu data, mentions haoshu CLI / Agent Bridge.
---

# 号薯 Agent（查询 + 写操作）

用 Shell 执行 `haoshu`，经本机 Agent Bridge（**127.0.0.1:19323**）查询或更新号薯数据。

**Announce at start:** 「我在用 haoshu-agent skill 查询/更新号薯数据。」

## 前置条件

| 条件 | 说明 |
|------|------|
| 号薯运行 | 桌面端已启动并保持运行 |
| Agent Bridge 已开启 | 设置 → Agent Bridge → 启用 |
| API Key | 号薯内生成；`haoshu auth set-key` 写入 `~/.haoshu/config.json` |
| CLI | `npm i -g @haoshu/cli@1.7.0` 或 `npx @haoshu/cli@1.7.0` |

首次：`haoshu auth status --json`（确认 Bridge 通、已登录、`readonly: false`）。

## 通用约定

- 始终加 `--json`
- 删除类须 `--confirm`（CLI）或 `confirm: true`（MCP）
- 本机绝对路径：稿件导入、设封面、素材上传等须号薯与 Agent **同机**可读
- 异步 Job：`haoshu jobs get <id>`（代理 Excel 导入、批量代理检测、批量统计/未读刷新）
- 存稿/发布 `start` 后无 Job：轮询 `draft-tasks get` / `publish-tasks get`

## 能力边界

**允许（读）：** 账号、稿件、存稿任务、发布任务、分组、标签、代理、稿件分类、员工（主账号只读）、`notify summary`、存稿运行聚合态。`accounts list|get` **不含 cookie**；统计字段须 `--include stats`（`stats.fansCount` / `stats.seven.*` / `stats.thirty.*`）；未读字段须 `--include unread`（`unread.unreadCount` / `unread.mentions` 等）。`refresh-metrics` / `refresh-unread` 单账号直接回带对应字段。

**允许（写）：** Cookie 导入加号、导出 Cookie（须 `cookieExport`）、账号 owner/备注、移组、设标签（含批量）、分组/标签 CRUD、代理库写与绑定、统计/未读刷新、账号 detect（登录态检测）、稿件路径导入、改删与批量分类/清标题、稿件分类 CRUD、设封面/插配图、存稿 create/start/cancel/delete、从已完成存稿创建发布（`publish-tasks from-draft`）、发布 start/start-batch/start-all-pending/cancel/delete。

**禁止：** 扫码/登录、删账号、员工增删改、公众号远端（mp-*）、链接导入稿件、设原文链接、飞书导入等。

**`--quick-filter` 取值：** `all` / `mainOnline` / `mainOffline` / `creatorOnline` / `creatorOffline` / `needsAttention` / `withIp` / `direct`。主页与创作中心各自独立登录，在线离线因此分两组；旧值 `online` / `offline` 按主页那组解释。

## CLI flags 索引（与 `haoshu --help` 对齐）

`--json` `--page` `--page-size` `--group` `--tag` `--q` `--type` `--quick-filter` `--remark` `--category` `--assigned` `--saved` `--cover` `--paths` `--strip-duplicate-title` `--file` `--url` `--clear` `--position` `--files` `--urls` `--payload-file` `--account` `--manuscripts` `--content-mode` `--image-dedup` `--note-empty-title` `--auto-clear-after-save` `--start` `--draft-task` `--ids` `--status` `--content-type` `--completed` `--confirm` `--owner` `--remark1` `--remark2` `--tags` `--accounts` `--mode` `--ip-proxy-id` `--identity-reuse` `--cookie` `--cookie-file` `--poll` `--force` `--no-force` `--host` `--port` `--include` `--label` `--region` `--username` `--password` `--anonymity` `--latency` `--extract-url` `--exit-ip` `--title` `--content`

## 常用命令

```bash
haoshu auth status --json
haoshu accounts list --quick-filter mainOnline --json
haoshu accounts list --include stats,unread --json
haoshu accounts refresh-unread <id> --json
haoshu notify summary --json
haoshu manuscripts list --assigned unassigned --json
haoshu manuscripts import-paths --paths /abs/a.zip --json
haoshu manuscripts patch <id> --title "新标题" --json
haoshu manuscripts delete <id> --confirm --json
haoshu draft-tasks create --account <id> --manuscripts m1,m2 --content-mode note --start --json
haoshu draft-tasks running-summary --json
haoshu publish-tasks from-draft --draft-task <存稿任务ID> --start --json
haoshu publish-tasks cancel <id> --json
haoshu accounts refresh-metrics <id> --json
haoshu accounts detect <id> --json
```

完整命令见 `haoshu --help`。

## MCP（可选）

`npx -y @haoshu/mcp@1.7.0`，与 CLI 共用 API Key；环境变量 `HAOSHU_API_KEY`、`HAOSHU_BASE_URL`（默认 `http://127.0.0.1:19323`）。

## 安装 Skill

```bash
mkdir -p ~/.cursor/skills/haoshu-agent
cp "$(npm root -g)/@haoshu/cli/skills/haoshu-agent/SKILL.md" ~/.cursor/skills/haoshu-agent/
```

或仓库内：`node scripts/sync-agent-skill.mjs`
