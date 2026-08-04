---
name: mpbook-agent
description: 通过 mpbook CLI 读写本机「公号书」（MPBook）桌面端的数据 —— 读取已抓取的公众号文章正文做跨文分析/选题/二创，查询与维护书架、分组、标签、笔记、收藏，触发抓取、导出、AI 摘要与 WebDAV 同步。Use when the user asks to read or analyze articles saved in MPBook, list/search their 公众号 books or articles, get article content or markdown, manage book groups/tags/notes/favorites, add a 公众号 to the shelf, trigger downloads or exports, run AI summaries, or mentions mpbook CLI / 公号书 / Agent Bridge.
---

# 公号书 Agent 接入

用 `mpbook` 命令行读写本机公号书。**最大价值是读正文** —— 用户书架里沉淀着成百上千篇已抓取入库的公众号文章，你能直接读到，做跨文分析、找选题、写作参考。

## 开始前必做

```bash
mpbook auth status --json
```

不通就照它的提示解决，**别继续往下猜**。四种失败各有不同解法：

| 退出码 | 含义 | 怎么办 |
|---|---|---|
| 3 | 连不上 | 公号书没启动，或设置里没开 Agent Bridge |
| 2 + `UNAUTHORIZED` | Key 不对 | 让用户到「设置 → Agent 接入」重新生成，再 `mpbook auth set-key` |
| 2 + `VIP_REQUIRED` | 非会员 | Bridge 仅会员可用，需开通会员 |
| 4 + `APP_NOT_READY` | 没登录 | 让用户在公号书里登录 |

**架构**：CLI 只连**本机** `127.0.0.1:19322`，不是云端服务，数据不出本机。公号书关掉就用不了。

## 读正文：本 Skill 的核心

```bash
mpbook articles content <articleId> --format text --max-chars 3000
```

不加 `--json` 时**直接输出裸正文**，可以管道、可以重定向。这是全 CLI 唯一一处不输出 JSON 的命令。

三种格式的取舍：

| format | 特点 | 什么时候用 |
|---|---|---|
| `text` | 纯文本，最省 token | **批量分析首选** |
| `md`（默认） | 保结构，图片是可用的远端 URL，站内链接保留 | 需要引用结构或图片时 |
| `html` | 正文 HTML | 你要自己解析时 |

### 批量分析的正确流程

**不要**对整本书逐篇拉全文 —— 一本书可能有两千多篇，会直接打爆你的上下文。

```bash
# 1. 先拿列表（只有元数据，很轻）
mpbook articles list --book <bookId> --status downloaded --limit 50 --json

# 2. 从标题/日期里挑出目标，只对挑中的取正文
mpbook articles content <id> --format text --max-chars 3000
```

`--max-chars` 会截断并在 `--json` 输出里给 `truncated: true` 与 `contentLength`（截断前的完整长度），你能知道自己拿到的是不是全部。

### 找文章的三种方式

```bash
mpbook search articles --q 关键词 --json                    # 全库搜索，带 snippet
mpbook search articles --q 关键词 --book <bookId> --json     # 书内搜索
mpbook articles list --book <id> --from 2026-01-01 --to 2026-06-30 --json
```

搜索结果统一是 `{ hits: [...] }`，书内和全库形状一致。

## 命令速查

```bash
# 书架
mpbook books list [--type account|collection|album] [--group <id>|ungrouped] [--q 关键词] --json
mpbook books get <id> --json
mpbook groups list --json

# 文章
mpbook articles list --book <id> [--status downloaded|pending|deleted|unavailable|all] \
                     [--original original|non_original] [--tag a,b] [--from Y-M-D] [--to Y-M-D] --json
mpbook articles get <id> --json
mpbook articles content <id> [--format text|md|html] [--max-chars N]

# 笔记与收藏
mpbook notes list [--book <id>] [--article <id>] [--q 关键词] --json
mpbook favorites list [--tag x] [--q 关键词] --json
mpbook favorites parse-url --url <公众号文章链接> --json
mpbook favorites create --url <链接> --json
mpbook favorites ensure-local <favoriteId> --json     # 拉正文入库，之后可读 content

# 打标签（会自动补进该书的标签词表）
mpbook articles set-tags <id> --tags 增长,案例 --json          # 全量替换
mpbook articles add-tags --ids a1,a2 --tags 增长 --json        # 批量追加

# 加公众号到书架
mpbook books search-account --q 公众号名 --json                # 拿 fakeid
mpbook books create --type account --fakeid <f> --name <n> --account <a> --json
mpbook downloads start --books <bookId> --json                 # 再抓取

# 长任务
mpbook downloads list --json
mpbook exports start --book <id> --format md --mode all --target-dir /绝对/路径 --json
mpbook ai summarize --articles a1,a2,a3 --poll --json
mpbook jobs get <jobId> --json
```

全部命令加 `--help` 可看详细用法。

## 硬约束（违反会失败或造成损失）

**删除类必须加 `--confirm`。** `books delete` / `groups delete` / `articles delete` / `articles batch-delete` / `notes delete` / `favorites delete` / `books tag-vocab-delete` / `favorite-tags delete` 缺 `--confirm` 时 CLI 本地就报错、不发请求。

**删之前先把待删对象列出来给用户确认**，别直接带 `--confirm` 执行。删书会级联删掉整本文章与本地正文，**不可恢复**。

**路径必须是本机绝对路径。** `--target-dir` / `--target-path` 走的是公号书直接读写磁盘，所以公号书与你必须在**同一台机器**上。相对路径会被拒。

**长任务要轮询。** `ai summarize` 多篇时返回 `{ jobId }`，用 `mpbook jobs get <id>` 或加 `--poll`。**Job 存于公号书进程内存，重启后 jobId 失效**（会返回 404 并说明原因）—— 那不是你 id 写错了，别反复重试。

**撞上微信环境验证（`CAPTCHA_REQUIRED`，423）时不要重试。** 告诉用户到公号书桌面端完成验证，你在命令行里绕不过去。

## 能力边界

**不支持**（有意排除，别去试）：

- 注册、登录、修改密码、退出软件
- 微信扫码授权公众号（`books create --type account` 用的是**已授权**的采集号去搜，不是新授权）
- 从传输助手导入收藏
- 修改设置、WebDAV 配置（会涉及明文密码与 API Key）
- 存储位置迁移
- 软件更新的下载与安装

**`--status read` / `--status unread` 不支持。** 已读状态存在桌面端本地（localStorage），Bridge 读不到。传了会回 400 并解释原因。

**仅会员可用。** 非会员所有请求返回 `403 VIP_REQUIRED`。

**存储迁移期间所有写操作返回 `409 LIBRARY_LOCKED`。** 那是正常保护，等迁移完成即可。

## 退出码

| 码 | 含义 |
|---|---|
| 0 | 成功 |
| 1 | 业务/参数错误 |
| 2 | 鉴权失败（Key 无效 / 非会员） |
| 3 | 连不上公号书 |
| 4 | 未就绪（未登录 / 公众号登录态失效） |

## 实战提示词示例

用户说「帮我把 XX 公众号最近 10 篇总结一下」，推荐这么做：

```bash
mpbook books list --q XX --json                                    # 找到书
mpbook articles list --book <id> --status downloaded --limit 10 --json   # 拿最近 10 篇
mpbook articles content <每篇的 id> --format text --max-chars 3000       # 逐篇取正文
```

然后你自己总结。**也可以用公号书自带的 AI**（`mpbook ai summarize --articles ... --poll`），结果会落库、在桌面端也能看到 —— 但那会消耗用户配置的 AI 额度，**先问用户要哪种**。

用户说「找出我收藏里所有讲增长的」：

```bash
mpbook search favorites --q 增长 --json
```

用户说「把这个号加进书架并抓下来」：

```bash
mpbook books search-account --q <号名> --json      # 先搜，拿 fakeid，报给用户确认是哪个
mpbook books create --type account --fakeid <f> --name <n> --account <a> --json
mpbook downloads start --books <新书 id> --json
mpbook downloads list --json                       # 轮询进度
```

## 别做的事

- **别把 API Key 写进代码仓库或任何会被提交的文件。** 它在 `~/.mpbook/config.json`，或用 `MPBOOK_API_KEY` 环境变量。
- **别臆造不存在的命令。** 不确定就 `mpbook <域> --help`。
- **别对整本书逐篇拉正文。** 先列表后按需取。
- **别把微信提示页当正文。** 违规下架/群发失败的文章，`content` 会照常返回 200 但带 `unavailableReason` 与 `unavailableText` —— 那是提示页不是真正文，要如实告诉用户。
