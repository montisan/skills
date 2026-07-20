---
name: haoxu-agent
description: 通过 haoxu CLI 查询号续账号、稿件、存稿任务、发表任务、分组/标签/代理 IP/稿件分类/员工，支持公众号远端草稿箱与发表记录查询，以及账号元数据写操作（号主/备注、移组、设标签、分组/标签 CRUD）、代理写操作（代理库 CRUD/绑定/检测/Excel 导入、异步 Job）、统计刷新/账号检测/未读刷新（0.6.0）、稿件路径/链接导入（0.7.0）与存稿/发表写操作（0.8.0）。Use when the user asks to query or update HaoXu data, list accounts/manuscripts/draft tasks/publish tasks/groups/tags/ip-proxies/categories/employees, query WeChat MP drafts/published posts, patch account owner/remarks, move groups, set tags, manage proxies, bind proxy, test proxy, import proxies, detect account proxy, refresh account metrics, detect account, refresh unread, import manuscripts from paths or links, create/start/cancel/delete draft tasks, create/start publish tasks from batch draft, or mentions haoxu CLI / Agent Bridge.
---

# 号续 Agent（查询 + 账号元数据写 + 代理写 + 统计刷新）

用 Shell 执行全局命令 `haoxu`，经本机 Agent Bridge 查询或更新号续数据。**0.4.0** 起允许账号元数据与分组/标签写操作；**0.5.0** 起允许代理库写、账号绑定代理、**代理检测**与 Excel 导入（异步 Job）；**0.6.0** 起允许统计刷新、**账号检测**（非代理检测）与未读刷新；**0.7.0** 起允许本机路径（含 zip）与公众号链接导入稿件；**0.8.0** 起允许存稿 create/start/cancel/delete 与发表 from-batch-draft / start / start-batch / start-all-pending。**不支持飞书导入**。

适用于任何能跑终端的 Agent（Cursor、Claude Code、Codex、WorkBuddy、QoderWork、QClaw、OpenClaw、Trae 等），不限单一产品。

**Announce at start:** 「我在用 haoxu-agent skill 查询/更新号续数据。」

## 前置条件

| 条件 | 说明 |
|------|------|
| 号续运行 | 桌面端已启动并保持运行 |
| Agent Bridge 已开启 | 主账号或子账号均可：设置 → Agent Bridge → 启用，状态为「运行中」（开关与 API Key 本机共用） |
| 桌面端版本 | 需含对应 Bridge 能力：账号元数据写 **0.4.0**；代理写与 Job **0.5.0**；统计刷新/账号检测/未读 **0.6.0**；稿件路径/链接导入 **0.7.0**；存稿/发表写 **0.8.0**（与 `@haoxu/cli` / `@haoxu/mcp` 同版本配套） |
| API Key | 在号续内生成；用 `haoxu auth set-key` 写入本机 |
| CLI 可用 | 已安装：`npm i -g @haoxu/cli@0.8.0`（或 `npx @haoxu/cli@0.8.0`） |
| 本 Skill | 推荐：`npx skills add montisan/skills --skill haoxu-agent -y -g`（见文末「安装本 Skill」） |

首次配置：

```bash
haoxu auth set-key
haoxu auth status --json
```

配置写入 `~/.haoxu/config.json`。默认 Bridge：`http://127.0.0.1:19321`。完整小白教程：仓库 `Desktop/docs/agent/haoxu-agent-guide.md` §1–§2。

## 能力边界

**允许（读）：** 查询账号与统计、稿件、存稿任务、发表任务、分组、标签、代理 IP、稿件分类、员工（主账号）、**公众号远端草稿箱与发表记录**。

**允许（写，0.4.0）：** 账号 `owner` / `remark1` / `remark2`、移组、设标签；分组与标签实体 CRUD。

**允许（写，0.5.0）：** 代理库 CRUD / 批量创建 / 移组 / 入库 test；账号绑定代理；无库连通性 test；单账号与批量「**代理检测**」（`proxy-detect`，非账号检测）；Excel 导入（异步 Job）。

**允许（写，0.6.0）：** 单账号与批量统计刷新、**账号检测**（`detect`，非 `proxy-detect`）、未读刷新。查统计仍用 `accounts list|get`，勿把 refresh 当查询接口。

**允许（写，0.7.0）：** 本机**绝对路径**（含 zip/docx/html/md/txt 等）与**公众号文章链接**导入稿件（同步返回 `{ imported, errors }`）。号续与 Agent **须同机**，路径须号续进程可读；Bridge 不弹文件对话框。**不支持飞书**、base64 上传或分类 CRUD。

**允许（写，0.8.0）：** 存稿任务 create/start/cancel/delete（create 可选 `--start` / `start: true`）；发表任务 from-batch-draft create、单条 start、start-batch、start-all-pending（create 可选定时 `schedule` 与 `--start`）。启动后**无 Job**，须轮询 `draft-tasks get` / `publish-tasks get` 查看 `status` 直至终态。

**禁止（不在范围）：** from-drafts / from-first-drafts 创建发表、飞书导入、删发表任务、员工增删改、账号增删、删草稿/改私密等。

**删除须确认：** 删除分组/标签/代理节点/存稿任务必须 `--confirm`（CLI）或 `confirm: true`（MCP），否则 `400 VALIDATION_ERROR`。

**异步 Job：** Excel 导入、批量代理检测、批量统计刷新/账号检测/未读刷新返回 `jobId`；用 `haoxu jobs get <id>` 轮询。Job 存于号续进程内存，完成后约 **1 小时** TTL 自动清理；**号续退出或重启后 Job 不可恢复**（旧 jobId → 404）。

## 写操作权限（与 UI/IPC 一致）

| 操作 | 主账号 | 员工（可见账号） | 员工（不可见账号） |
|------|--------|------------------|-------------------|
| `PATCH` `owner` / `remark1` / `remark2` | ✅ | ✅（仅这三字段） | `404 NOT_FOUND` |
| `move-group` | ✅ | `403 FORBIDDEN` | `404` 若不可见，否则 `403` |
| `set-tags`（账号打标） | ✅（须会员标签额度） | ✅ 可见账号 + 会员额度 | `404` |
| 分组 create/update/delete | ✅ | `403` | — |
| 标签 create/update/delete | ✅（须会员额度） | ✅（须会员额度） | — |

会员不足写标签 → `403 FORBIDDEN`（如「请开通会员后再添加标签」）。

## 代理写操作权限（0.5.0，与 UI/IPC 一致）

| 操作 | 主账号 | 员工 |
|------|--------|------|
| 代理 create / batch-create | ✅ | ✅（分组须授权） |
| 代理 update | ✅ | 见 IPC（备注始终可；改 groupId 禁；其它字段须会员额度且节点在授权组） |
| 代理 delete / move-group | ✅ | ❌ `403` |
| 入库节点 test | ✅ | ✅ 且节点可见；不可见 → `404` |
| bind-proxy | ✅ | ✅ 账号可见 + managed 规则 |
| `proxy test`（无库） | ✅ | ✅ |
| 单账号 / 批量 proxy detect | ✅ | ✅ 仅可见账号 |
| Excel import | ✅ | ✅（同 IPC 导入权限） |
| 统计刷新 / 账号 detect / 未读 | ✅ | ✅ 可见账号 | 不可见 → `404` |
| import-paths / import-links | ✅ | ✅（对齐 IPC：会话 owner = userId） | — |

## 安装 CLI

```bash
npm install -g @haoxu/cli@0.8.0
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

### 代理写操作（0.5.0）

```bash
# 创建 / 更新 / 删除（删除须 --confirm）
haoxu ip-proxies create --host 1.2.3.4 --port 1080 --type socks5 --label 节点A --json
haoxu ip-proxies update <id> --label 改名 --json
haoxu ip-proxies delete <id> --confirm --json

# 批量创建（JSON 文件含 items 数组）
haoxu ip-proxies batch-create --file ./proxies.json --json

# 移组
haoxu ip-proxies move-group --ids id1,id2 --group <groupId|ungrouped> --json

# 入库节点连通性 test（失败仍 200，状态写入 DTO）
haoxu ip-proxies test <id> --json

# Excel 导入（异步，返回 jobId）
haoxu ip-proxies import --file ./proxies.xlsx --group <groupId> --json
haoxu jobs get <jobId> --json

# 账号绑定代理
haoxu accounts bind-proxy --accounts a1,a2 --mode managed --ip-proxy-id <id> --json
haoxu accounts bind-proxy --accounts a1 --mode direct --json
haoxu accounts bind-proxy --accounts a1 --mode custom --payload-file ./proxy.json --json

# 单账号代理检测（对齐 UI「代理检测」）
haoxu accounts proxy-detect <accountId> --json

# 批量代理检测（异步；--poll 自动轮询 Job，间隔 1s，超时 120s）
haoxu accounts proxy-detect-batch --accounts a1,a2 --poll --json

# 无库连通性 test（不入库）
haoxu proxy test --host 1.2.3.4 --port 1080 --type socks5 --username u --password p --json
```

### 统计刷新 / 账号检测 / 未读（0.6.0）

查统计仍用 `accounts list|get`；下列为触发刷新/检测的写操作。**账号检测**（`detect`）≠ **代理检测**（`proxy-detect`）。

```bash
# 单账号统计刷新（返回更新后的账号 DTO）
haoxu accounts refresh-metrics <id> [--no-force] --json

# 批量统计刷新（异步；省略 --accounts 则处理全部可见可刷新账号）
haoxu accounts refresh-metrics-batch [--accounts a1,a2] [--poll] [--json]

# 单账号检测（公众号有效性等）
haoxu accounts detect <id> [--no-force] --json

# 批量账号检测（异步）
haoxu accounts detect-batch [--accounts a1,a2] [--poll] [--json]

# 单账号未读刷新
haoxu accounts refresh-unread <id> [--no-force] --json

# 批量未读刷新（异步）
haoxu accounts refresh-unread-batch [--accounts a1,a2] [--poll] [--json]
```

默认 `force: true`；传 `--no-force` 时仅刷新符合自动策略的账号。批量 `--poll` 间隔 1s，超时 120s。

### 稿件导入（0.7.0）

路径须为**本机绝对路径**（如 `/Users/me/article.zip`）；号续与 Agent 同机，Bridge 直接读盘。**不支持飞书**。

```bash
# 从本机路径导入（含 zip；可选分类与去重标题）
haoxu manuscripts import-paths --paths /abs/a.docx,/abs/b.zip [--category <id>] [--strip-duplicate-title] --json

# 从公众号链接导入
haoxu manuscripts import-links --urls https://mp.weixin.qq.com/s/xxx,https://mp.weixin.qq.com/s/yyy [--category <id>] --json
```

同步返回 `{ imported, errors }`；部分失败仍 exit 0（成功进 `imported`，失败进 `errors`）。可选 `--category` 须为已存在分类 ID（`categories list` 查询）；不做分类 CRUD。

### 存稿 / 发表写操作（0.8.0）

复杂存稿配置优先 `--payload-file`（JSON 对齐 create body，可含 `start`；**payload 覆盖同名 CLI flags**）。启动后 fire-and-forget，**须轮询 GET** 直至终态：

```bash
# 创建存稿（可选 --start 创建后立即启动）
haoxu draft-tasks create --account <账号ID> --manuscripts m1,m2 [--payload-file ./draft.json] [--start] --json
haoxu draft-tasks start <任务ID> --json
haoxu draft-tasks cancel <任务ID> --json
haoxu draft-tasks delete <任务ID> --confirm --json

# 轮询进度
haoxu draft-tasks get <任务ID> --json

# 从已完成存稿创建发表（可选定时 + --start）
haoxu publish-tasks from-batch-draft --tasks d1,d2 [--schedule-date <label> --schedule-hour <h> --schedule-minute <m>] [--start] --json
haoxu publish-tasks start <任务ID> --json
haoxu publish-tasks start-batch --ids p1,p2 --json
haoxu publish-tasks start-all-pending --json

# 轮询发表进度
haoxu publish-tasks get <任务ID> --json
```

员工仅可操作**可见账号**下任务；不可见 → `404 NOT_FOUND`。账号离线/非公众号不可存稿 → `503 ACCOUNT_NOT_READY`。

响应 DTO **不含 password**；创建/更新请求体可含 password。

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
| 403 FORBIDDEN（employees / move-group / groups / 代理 delete·move-group） | 需主账号登录 |
| 403 FORBIDDEN（tags / 代理 update 额度） | 会员额度不足 |
| 404 NOT_FOUND（mp-* / 不可见账号·代理） | 资源不存在或员工不可见 |
| 400 VALIDATION_ERROR（delete） | 删除分组/标签/代理/存稿任务须 `--confirm` |
| 404 NOT_FOUND（jobs） | jobId 不存在、已 TTL 清理，或号续已重启 |
| `haoxu: command not found` | `npm install -g @haoxu/cli@0.8.0` |
| 未知命令 / 过滤无效 | 升级桌面端与 CLI 至 0.8.0 |
| 路径不可读 / 导入失败 | 确认绝对路径、号续与 Agent 同机、文件扩展名支持 |

## MCP（可选，0.8.0）

`npx -y @haoxu/mcp@0.8.0`，与 CLI 共用 API Key。

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
| `haoxu_create_ip_proxy` / `haoxu_update_ip_proxy` / `haoxu_delete_ip_proxy` | 代理 CRUD（delete 须 `confirm: true`） |
| `haoxu_batch_create_ip_proxies` / `haoxu_move_ip_proxies` | 批量创建 / 移组 |
| `haoxu_test_ip_proxy` | 入库节点 test |
| `haoxu_import_ip_proxies` | Excel 导入（异步，返回 `jobId`） |
| `haoxu_bind_accounts_proxy` | 账号绑定代理 |
| `haoxu_detect_account_proxy` / `haoxu_detect_account_proxy_batch` | 单账号 / 批量**代理**检测（批量异步） |
| `haoxu_refresh_account_metrics` / `haoxu_refresh_account_metrics_batch` | 单账号 / 批量统计刷新（批量异步） |
| `haoxu_detect_account` / `haoxu_detect_account_batch` | 单账号 / 批量**账号**检测（非代理；批量异步） |
| `haoxu_refresh_account_unread` / `haoxu_refresh_account_unread_batch` | 单账号 / 批量未读刷新（批量异步） |
| `haoxu_test_proxy_connectivity` / `haoxu_proxy_lookup_exit_ip` | 无库 test / 查出口 IP |
| `haoxu_get_job` | 查询异步 Job 状态 |
| `haoxu_import_manuscripts_from_paths` | 本机绝对路径导入稿件（同步 `{ imported, errors }`） |
| `haoxu_import_manuscripts_from_links` | 公众号链接导入稿件（同步 `{ imported, errors }`） |
| `haoxu_create_batch_draft_task` | 创建存稿任务（可选 `start`） |
| `haoxu_start_batch_draft_task` | 启动存稿任务 |
| `haoxu_cancel_batch_draft_task` | 取消存稿任务 |
| `haoxu_delete_batch_draft_task` | 删除存稿任务（须 `confirm: true`） |
| `haoxu_create_publish_tasks_from_batch_draft` | 从存稿创建发表（可选 `schedule`、`start`） |
| `haoxu_start_publish_task` | 启动单条发表任务 |
| `haoxu_start_publish_tasks_batch` | 批量启动发表任务 |
| `haoxu_start_all_pending_publish_tasks` | 启动全部可见待执行发表任务 |

MCP 参数名与 HTTP query/body 对齐（如 `quickFilter`、`groupId`、`assigned`、`completed`、`status`、`contentType`、`datePreset`）。`haoxu_import_ip_proxies`、`haoxu_detect_account_proxy_batch` 及 Phase C 批量工具返回 `jobId`，用 `haoxu_get_job` 轮询；Job 内存存储，号续重启后丢失。稿件导入与存稿/发表写为**同步** fire-and-forget；启动后用 `haoxu_get_draft_task` / `haoxu_get_publish_task` 轮询进度。

## 安装本 Skill / CLI / MCP

**推荐（全局 Skill）：**

```bash
npx skills add montisan/skills --skill haoxu-agent -y -g
npm install -g @haoxu/cli@0.8.0
# 可选 MCP：
npm install -g @haoxu/mcp@0.8.0
```

装完后请用户：号续 → 设置 → Agent Bridge → 启用 → 生成 Key → `haoxu auth set-key` → `haoxu auth status --json`。然后**重启当前 Agent / 开新对话**。

**Agent 代装：** 用户可把教程 `docs/agent/haoxu-agent-guide.md` §2 发给**任意**能执行终端的 Agent（含 WorkBuddy、QoderWork、QClaw、OpenClaw、Trae 等）；或由你直接执行上述命令后协助配置 Key。完整小白步骤见该教程 §1。

**备选（从 npm 包复制 Skill）：** 拷到当前产品要求的 skills 目录，例如：

```bash
mkdir -p ~/.claude/skills/haoxu-agent   # 或 .cursor/skills/haoxu-agent 等
cp "$(npm root -g)/@haoxu/cli/skills/haoxu-agent/SKILL.md" ~/.claude/skills/haoxu-agent/
```
