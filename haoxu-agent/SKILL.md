---
name: haoxu-agent
description: 通过 haoxu CLI 查询号续账号、稿件、存稿任务、发表任务、分组/标签/代理 IP/稿件分类/员工，支持公众号远端草稿箱与发表记录查询，以及账号元数据写操作（号主/备注、移组、设标签、分组/标签 CRUD）。Use when the user asks to query or update HaoXu data, list accounts/manuscripts/draft tasks/publish tasks/groups/tags/ip-proxies/categories/employees, query WeChat MP drafts/published posts, patch account owner/remarks, move groups, set tags, or mentions haoxu CLI / Agent Bridge.
---

# 号续 Agent（查询 + 账号元数据写操作）

用 Shell 执行全局命令 `haoxu`，经本机 Agent Bridge 查询或更新号续数据。**0.4.0 起允许账号元数据与分组/标签写操作；其它写操作仍禁止。**

**Announce at start:** 「我在用 haoxu-agent skill 查询/更新号续数据。」

## 前置条件

| 条件 | 说明 |
|------|------|
| 号续运行 | 桌面端已启动并保持运行 |
| Agent Bridge 已开启 | 主账号或子账号均可：设置 → Agent Bridge → 启用，状态为「运行中」（开关与 API Key 本机共用） |
| 桌面端版本 | 需含**账号元数据写操作**能力的 Bridge（与 `@haoxu/cli` / `@haoxu/mcp` **0.4.0** 配套） |
| API Key | 在号续内生成；用 `haoxu auth set-key` 写入本机 |
| CLI 可用 | 已安装：`npm i -g @haoxu/cli@0.4.0`（或 `npx @haoxu/cli@0.4.0`） |

首次配置：

```bash
haoxu auth set-key
haoxu auth status --json
```

配置写入 `~/.haoxu/config.json`。默认 Bridge：`http://127.0.0.1:19321`。

## 能力边界

**允许（读）：** 查询账号与统计、稿件、存稿任务、发表任务、分组、标签、代理 IP、稿件分类、员工（主账号）、**公众号远端草稿箱与发表记录**。

**允许（写，0.4.0）：** 账号 `owner` / `remark1` / `remark2`、移组、设标签；分组与标签实体 CRUD。

**禁止：** 创建/启动存稿或发表、导入稿件、代理绑定/检测、统计刷新、账号检测、员工增删改、账号增删、删草稿/改私密等。

**删除须确认：** 删除分组/标签必须 `--confirm`（CLI）或 `confirm: true`（MCP），否则 `400 VALIDATION_ERROR`。

## 写操作权限（与 UI/IPC 一致）

| 操作 | 主账号 | 员工（可见账号） | 员工（不可见账号） |
|------|--------|------------------|-------------------|
| `PATCH` `owner` / `remark1` / `remark2` | ✅ | ✅（仅这三字段） | `404 NOT_FOUND` |
| `move-group` | ✅ | `403 FORBIDDEN` | `404` 若不可见，否则 `403` |
| `set-tags`（账号打标） | ✅（须会员标签额度） | ✅ 可见账号 + 会员额度 | `404` |
| 分组 create/update/delete | ✅ | `403` | — |
| 标签 create/update/delete | ✅（须会员额度） | ✅（须会员额度） | — |

会员不足写标签 → `403 FORBIDDEN`（如「请开通会员后再添加标签」）。

## 安装 CLI

```bash
npm install -g @haoxu/cli@0.4.0
haoxu --help
```

## 常用命令（请加 `--json`）

### 基础查询

```bash
haoxu auth status --json
haoxu accounts list --json
haoxu accounts get <账号ID> --json
haoxu manuscripts list --json
haoxu manuscripts get <稿件ID> --json --include content
haoxu draft-tasks list --json
haoxu draft-tasks get <任务ID> --json
haoxu publish-tasks list --json
haoxu publish-tasks get <任务ID> --json
```

### 新增域（0.2.0）

```bash
haoxu groups list --json
haoxu groups get <分组ID> --json
haoxu tags list --json
haoxu tags get <标签ID> --json
haoxu ip-proxies list --json
haoxu ip-proxies get <代理ID> --json
haoxu categories list --json
haoxu categories get <分类ID> --json
haoxu employees list --json    # 仅主账号
haoxu employees get <员工ID> --json
```

### 账号元数据写操作（0.4.0）

```bash
# 修改号主/备注（至少传一项）
haoxu accounts patch <id> --owner 张三 --json
haoxu accounts patch <id> --remark1 备注A --remark2 备注B --json

# 移组（ungrouped = 移出分组）
haoxu accounts move-group <id> --group <groupId|ungrouped> --json

# 设标签（全量替换，逗号分隔；空 = 清空）
haoxu accounts set-tags <id> --tags t1,t2 --json

# 分组 CRUD
haoxu groups create --name 新分组 --json
haoxu groups update <id> --name 改名 --json
haoxu groups delete <id> --confirm --json

# 标签 CRUD
haoxu tags create --name 新标签 --json
haoxu tags update <id> --name 改名 --json
haoxu tags delete <id> --confirm --json
```

### 公众号远端（0.3.0）

实时查询微信公众平台，**非**号续本地任务表。需**目标账号在线**且已登录公众平台（有可用 session）。

```bash
# 先查在线账号 ID
haoxu accounts list --quick-filter online --json

# 草稿箱关键词搜索
haoxu accounts mp-drafts <账号ID> --q 关键词 --json

# 贴图草稿
haoxu accounts mp-drafts <账号ID> --content-type note --json

# 今天发表记录
haoxu accounts mp-published <账号ID> --date-preset today --json

# 自定义日期范围
haoxu accounts mp-published <账号ID> --from 2026-07-01 --to 2026-07-17 --json
```

#### 草稿箱 `accounts mp-drafts`

| Flag | 取值 |
|------|------|
| `--q` | 关键词（透传微信草稿搜索） |
| `--content-type` | `article`（默认）\| `note` |
| `--page` / `--page-size` | 分页（pageSize 上限 20） |

#### 发表记录 `accounts mp-published`

| Flag | 取值 |
|------|------|
| `--q` | 关键词 |
| `--date-preset` | `today` \| `yesterday` \| `last_7_days` |
| `--from` / `--to` | `YYYY-MM-DD`，须成对；与 `--date-preset` **互斥** |
| `--page` / `--page-size` | 分页（pageSize 上限 20） |

**`scanExhausted`：** 带日期条件时 Bridge 会**有界跨页扫描**（最多 10 页微信列表）。若响应中 `scanExhausted: true`，表示达扫描上限提前停止，区间内结果**可能不完整**——应缩小日期范围或告知用户。

自然日（today 等）按**号续桌面端系统本地时区**计算。

## 列表过滤（英文码，与 UI 对齐）

多选 ID 用**逗号分隔**（CLI 用 `--group a,b`；HTTP 用 `groupId=a,b`）。省略或空 = 不限。

### 账号 `accounts list`

| Flag | 取值 |
|------|------|
| `--group` | 分组 id；哨兵 `ungrouped`（未分组） |
| `--tag` | 标签 id；哨兵 `untagged`（未打标） |
| `--type` | `all` \| `official` \| `mini_program` |
| `--quick-filter` | `all` \| `online` \| `offline` \| `renewedToday` \| `notRenewedToday` \| `needsAttention` \| `withIp` \| `direct` |
| `--remark` | 备注筛选项字面值（逗号多选） |
| `--q` | 搜索 name/wxid/owner/remark |

```bash
haoxu accounts list --quick-filter online --group g1,g2 --json
haoxu accounts list --tag untagged --quick-filter needsAttention --json
```

### 稿件 `manuscripts list`

| Flag | 取值 |
|------|------|
| `--category` | 分类 id；哨兵 `uncategorized` |
| `--assigned` | `all` \| `assigned` \| `unassigned` |
| `--saved` | `all` \| `saved` \| `unsaved` |
| `--cover` | `all` \| `set` \| `unset` |
| `--q` | 搜索 title / sourceFileName |

```bash
haoxu manuscripts list --assigned unassigned --cover unset --json
```

### 存稿任务 `draft-tasks list`

| Flag | 取值 |
|------|------|
| `--status` | `all` \| `pending` \| `running` \| `completed` \| `failed`（`completed` **含** `partial_completed`） |
| `--content-type` | `all` \| `article` \| `note` |
| `--completed` | `all` \| `today` \| `yesterday` \| `last_3_days` |
| `--publish` | `all` \| `published` \| `unpublished` |
| `--q` | 搜索账号名 |

```bash
haoxu draft-tasks list --status completed --completed today --json
```

### 发表任务 `publish-tasks list`

| Flag | 取值 |
|------|------|
| `--status` | `all` \| `pending` \| `running` \| `failed` \| `published` \| `scheduled`（`scheduled` = UI「已定时」） |
| `--q` | 搜索账号名 |

```bash
haoxu publish-tasks list --status scheduled --json
```

### 代理 IP `ip-proxies list`

| Flag | 取值 |
|------|------|
| `--status` | `all` \| `healthy` \| `slow` \| `failed` |
| `--group` | 分组 id（逗号多选）；哨兵 `ungrouped` |
| `--q` | 搜索 label/host/username/remark/绑定账号名 |

## 员工列表权限

- **仅主账号（owner）**可查询 `employees list|get`
- 员工子账号调用 → Bridge 返回 **403 FORBIDDEN**
- 未登录号续 → **503 APP_NOT_READY**

## 退出码

`0` 成功 | `1` 业务/参数 | `2` 鉴权 | `3` 连接失败

## 错误排查

| 现象 | 处理 |
|------|------|
| Connection refused / exit 3 | 启动号续并启用 Agent Bridge |
| 401 / exit 2 | `haoxu auth set-key` 或重新生成 Key |
| 503 APP_NOT_READY | 在号续内完成登录 |
| 503 ACCOUNT_NOT_READY | 目标账号离线或未登录公众平台 |
| 401 REMOTE_UNAUTHORIZED | 微信侧 session 失效，在号续内重新登录该账号 |
| 502 UPSTREAM_ERROR | 微信接口异常，稍后重试 |
| 403 FORBIDDEN（employees / move-group / groups） | 需主账号登录 |
| 403 FORBIDDEN（tags） | 会员标签额度不足 |
| 404 NOT_FOUND（mp-* / 不可见账号） | 账号不存在或员工不可见 |
| 400 VALIDATION_ERROR（delete） | 删除分组/标签须 `--confirm` |
| `haoxu: command not found` | `npm install -g @haoxu/cli@0.4.0` |
| 未知命令 / 过滤无效 | 升级桌面端与 CLI 至 0.4.0 |

## MCP（可选，0.4.0）

`npx -y @haoxu/mcp@0.4.0`，与 CLI 共用 API Key。

| 工具 | 说明 |
|------|------|
| `haoxu_status` | Bridge 与登录状态 |
| `haoxu_list_accounts` / `haoxu_get_account` | 账号（含过滤参数） |
| `haoxu_list_manuscripts` / `haoxu_get_manuscript` | 稿件 |
| `haoxu_list_draft_tasks` / `haoxu_get_draft_task` | 存稿任务 |
| `haoxu_list_publish_tasks` / `haoxu_get_publish_task` | 发表任务 |
| `haoxu_list_groups` / `haoxu_get_group` | 分组 |
| `haoxu_list_tags` / `haoxu_get_tag` | 标签 |
| `haoxu_list_ip_proxies` / `haoxu_get_ip_proxy` | 代理 IP |
| `haoxu_list_categories` / `haoxu_get_category` | 稿件分类 |
| `haoxu_list_employees` / `haoxu_get_employee` | 员工（主账号） |
| `haoxu_list_mp_drafts` | 公众号远端草稿箱（`accountId` + `q` / `contentType` / 分页） |
| `haoxu_list_mp_published` | 公众号远端发表记录（`accountId` + `q` / `datePreset` / `from`+`to` / 分页） |
| `haoxu_patch_account` | 修改号主/备注 |
| `haoxu_move_account_group` | 移组 |
| `haoxu_set_account_tags` | 设标签（全量替换） |
| `haoxu_create_group` / `haoxu_update_group` / `haoxu_delete_group` | 分组 CRUD（delete 须 `confirm: true`） |
| `haoxu_create_tag` / `haoxu_update_tag` / `haoxu_delete_tag` | 标签 CRUD（delete 须 `confirm: true`） |

MCP 参数名与 HTTP query/body 对齐（如 `quickFilter`、`groupId`、`assigned`、`completed`、`status`、`contentType`、`datePreset`）。

## 安装本 Skill

```bash
# Cursor
mkdir -p .cursor/skills/haoxu-agent
cp "$(npm root -g)/@haoxu/cli/skills/haoxu-agent/SKILL.md" .cursor/skills/haoxu-agent/

# Claude Code
mkdir -p ~/.claude/skills/haoxu-agent
cp "$(npm root -g)/@haoxu/cli/skills/haoxu-agent/SKILL.md" ~/.claude/skills/haoxu-agent/
```
