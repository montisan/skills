---
name: haoshi-agent
description: 通过 haoshi CLI 查询号视账号、图文、视频、存稿/发布任务、评论任务、分组/标签/代理 IP/分类、员工（只读），支持 Cookie 导入加号、账号元数据写、代理写、统计/未读刷新、图文与视频导入改删、设封面/换配图、话题标签、定时发表、发表并评论。默认 Bridge 端口 19327。不支持扫码登录、员工管理、升级会员、删账号。Use when the user asks to query or update HaoShi data, mentions haoshi CLI / Agent Bridge.
---

# 号视 Agent（查询 + 写操作）

用 Shell 执行 `haoshi`，经本机 Agent Bridge（**127.0.0.1:19327**）查询或更新号视数据。

**Announce at start:** 「我在用 haoshi-agent skill 查询/更新号视数据。」

## 前置条件

| 条件 | 说明 |
|------|------|
| 号视运行 | 桌面端已启动并保持运行 |
| Agent Bridge 已开启 | 设置 → Agent Bridge → 启用 |
| API Key | 号视内生成；`haoshi auth set-key` 写入 `~/.haoshi/config.json` |
| CLI | `npm i -g @haoshi/cli@1.0.0` 或 `npx @haoshi/cli@1.0.0` |

首次：`haoshi auth status --json`（确认 Bridge 通、已登录、`readonly: false`）。

## 通用约定

- 始终加 `--json`
- 删除类须 `--confirm`（CLI）或 `confirm: true`（MCP）
- 本机绝对路径：稿件导入、设封面、素材上传等须号视与 Agent **同机**可读
- 异步 Job：`haoshi jobs get <id>`（代理 Excel 导入、批量代理检测、批量统计/未读刷新）
- 存稿/发布 `start` 后无 Job：轮询 `draft-tasks get` / `publish-tasks get`

## 能力边界

**允许（读）：** 账号、图文、视频、图文/视频存稿与发表任务、评论任务、分组、标签、代理、图文分类、视频分类、员工（主账号只读）、`notify summary`、存稿运行聚合态。`accounts list|get` **不含 cookie**；统计字段须 `--include stats`；未读字段须 `--include unread`。

**允许（写）：** Cookie 导入加号、导出 Cookie（须 `cookieExport`）、账号 owner/备注、移组、设标签、分组/标签/分类 CRUD、代理库写与绑定、统计/未读刷新、账号 detect、图文路径导入、改标题/描述/正文、整组换配图、批量描述、清除已存稿未分配稿件、设封面/插配图。存稿可用 `--payload-file` 传 `options.markTag`（话题标签）。发表 `from-draft` / `video-tasks from-draft` 支持三种发表方式（见下），以及 `update-schedule`、`delete-completed`。视频素材、视频分类、视频存稿/发表、评论任务同样可查可写。

**发表三种方式（图文 `publish-tasks` / 视频 `video-tasks` 相同）：**

| 方式 | flags | 说明 |
|------|--------|------|
| 平台定时 | `--schedule` + `--schedule-mode platform`（可省略 mode）+ `--start` | 创建后执行，由视频号到点发出；**不可**带 `--comment`；时间须在约 14 天内 |
| 本地定时发表及评论 | `--schedule` + `--schedule-mode local` + `--comment`（必填）`[--pin-comment]` | **勿** `--start`；号视进程到点再立即发表并建评论任务；软件未运行则不会发 |
| 立刻发表并评论 | `--comment` `[--pin-comment]` + `--start`（不传 `--schedule`） | 马上开跑，成功后建评论任务 |

**禁止：** 扫码登录、重新登录、打开视频号页面或创作台、删账号、员工增删改、升级会员、本机设置（锁屏/性能/缓存/检查更新）。

**`--quick-filter` 取值：** `all` / `mainOnline` / `mainOffline` / `renewedToday` / `notRenewedToday` / `needsAttention` / `withIp` / `direct`。在线离线只认视频号；`renewedToday` / `notRenewedToday` 按距上次延续是否不足 8 小时；旧值 `online` / `offline` / `creator*` 按视频号那组解释。

**`--type` 取值：** `all` / `official` / `cn` / `global` / `mini_program`。`official` = 全部视频号（国内版 + 海外版）；`cn` / `global` 精确到版本；`mini_program` 是历史取值，号视不再产生。

**海外版账号（`--type global`）三处不同：** ① 创作中心没有数据统计，`refresh-metrics` 回 `ACCOUNT_NOT_READY`，批量刷新算跳过而非失败、默认目标集也不含它；② 主页没有私信接口，`refresh-unread` 可用但只写通知三项，`unread.chatUnreadCount` 恒为 0；③ 创作中心没有长文入口，`draft-tasks create` 只能 `--content-mode note`。Cookie 导入加号时账号版本由 Cookie 条目的 domain 自动判定（纯 `k=v` 头无 domain，按国内版处理）。

## CLI flags 索引（与 `haoshi --help` 对齐）

`--json` `--page` `--page-size` `--group` `--tag` `--q` `--type` `--quick-filter` `--remark` `--category` `--assigned` `--saved` `--cover` `--paths` `--strip-duplicate-title` `--file` `--url` `--clear` `--position` `--files` `--urls` `--payload-file` `--account` `--manuscripts` `--videos` `--content-mode` `--image-dedup` `--note-empty-description` `--auto-clear-after-save` `--delete-local` `--delete-local-on-clear` `--start` `--draft-task` `--ids` `--status` `--kind` `--source` `--content-type` `--completed` `--confirm` `--owner` `--remark1` `--remark2` `--tags` `--accounts` `--mode` `--ip-proxy-id` `--identity-reuse` `--cookie` `--cookie-file` `--poll` `--force` `--no-force` `--host` `--port` `--include` `--label` `--region` `--username` `--password` `--anonymity` `--latency` `--extract-url` `--exit-ip` `--title` `--description` `--content` `--name` `--color` `--schedule` `--schedule-mode` `--comment` `--pin-comment`

## 常用命令

```bash
haoshi auth status --json
haoshi accounts list --quick-filter mainOnline --json
haoshi accounts list --include stats,unread --json
haoshi accounts refresh-unread <id> --json
haoshi notify summary --json
haoshi manuscripts list --assigned unassigned --json
haoshi manuscripts import-paths --paths /abs/a.zip --json
haoshi manuscripts patch <id> --title "新标题" --json
haoshi manuscripts delete <id> --confirm --json
haoshi draft-tasks create --account <id> --manuscripts m1,m2 --content-mode note --start --json
haoshi draft-tasks running-summary --json
# 平台定时
haoshi publish-tasks from-draft --draft-task <存稿任务ID> --schedule 2026-10-20T10:00 --schedule-mode platform --start --json
# 本地定时发表及评论（勿 --start）
haoshi publish-tasks from-draft --draft-task <存稿任务ID> --schedule 2026-10-20T10:00 --schedule-mode local --comment "谢谢" --pin-comment --json
# 立刻发表并评论
haoshi publish-tasks from-draft --draft-task <存稿任务ID> --comment "谢谢" --pin-comment --start --json
haoshi videos import-paths --paths /abs/a.mp4 --json
haoshi video-tasks from-draft --draft-task <id> --schedule-mode local --schedule 2026-10-20T10:00 --comment "谢谢" --json
haoshi comment-tasks list --source after_publish --json
haoshi publish-tasks cancel <id> --json
haoshi accounts refresh-metrics <id> --json
haoshi accounts detect <id> --json
```

完整命令见 `haoshi --help`。

## MCP（可选）

`npx -y @haoshi/mcp@1.0.0`，与 CLI 共用 API Key；环境变量 `HAOSHI_API_KEY`、`HAOSHI_BASE_URL`（默认 `http://127.0.0.1:19327`）。

## 安装 Skill

```bash
mkdir -p ~/.cursor/skills/haoshi-agent
cp "$(npm root -g)/@haoshi/cli/skills/haoshi-agent/SKILL.md" ~/.cursor/skills/haoshi-agent/
```

或仓库内：`node scripts/sync-agent-skill.mjs`
