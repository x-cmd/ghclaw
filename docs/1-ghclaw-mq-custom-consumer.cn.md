---
x-title: ghclaw MQ 协议——构建你自己的消费者
x-desc: >-
  ghclaw 的 MQ 是一个只追加 TSV 文件——监听器与你的代码之间的公共接口。
  本文详解行格式、按类型的尾列、事件 JSON 反查，以及 Python / awk /
  shell 消费者写法，和用自己的处理器替换内置派发器的方法。
x-sidebar: ghclaw
x-keywords: ghclaw, mq, tsv, 事件队列, 自定义消费者, python, tailer, 监听器
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'ghclaw MQ 协议——构建你自己的消费者'
      inLanguage: 'cn'
      about: 'ghclaw MQ 协议与自定义消费者'
---

# ghclaw MQ 协议——构建你自己的消费者

`x ghclaw run` 内置的 AI 处理器（triage + `@x` 回复）只是
**一套默认最原始的流水线**——一种开箱即用的事件反应方式。
`ghclaw` 与你的代码之间真正的契约是一个文件：**MQ**，一个只
追加的 TSV 队列。

启动监听，让它持续累积事件，然后用**任何语言**读取 MQ，
实现你关心的特定事件与处理逻辑。本文是该协议的参考手册。
默认处理器本身会随版本演进——本文档化的 MQ 契约才是稳定
的部分。

> **配套阅读：** [ghclaw 着陆页](0-ghclaw-landing.cn.md)介绍
> 命令、AI 处理器与限制；本文深入数据契约本身。

## 一段话契约

- 每个 target 一个文件：
  `$X_CMD_ROOT_TMP/ghclaw/events/data/<target>/.mq/MQ.CURRENT/data.tsv`
  （`listen` 启动时会打印确切路径）。
- **只追加**：新事件只追加，不删除不改写。多个消费者可以各
  自带游标读取；重放就是重新读一遍。
- 每个事件一行，**TAB 分隔**，列内无裸 TAB/换行（正文里的空
  白已压平为单空格）。
- 消费端做**粗筛**所需的全部信息都在行内；需要完整上下文时
  才去查事件 JSON。

## 行格式

前 9 列定长，其后为按类型追加的尾列：

| # | 列 | 示例 | 含义 |
| --- | --- | --- | --- |
| 1 | `created_at` | `2026-09-17T11:23:53Z` | GitHub 服务端时间，ISO 8601 UTC |
| 2 | `id` | `15214943014` | 事件 id——去重/游标锚点 |
| 3 | `type` | `IssuesEvent` | PascalCase 事件类型 |
| 4 | `action` | `opened` | webhook action 的子集；无 action 的类型为空 |
| 5 | `repo` | `qiakai/test-ghclaw` | `owner/repo`——路由关键列 |
| 6 | `actor` | `lunrenyi` | 触发者；机器人带 `[bot]` 后缀 |
| 7 | `number` | `5` | issue / PR 号；无此概念时为空 |
| 8 | `title` | `test: event batch1` | issue/讨论/release 标题；无标题的类型为空（典型如 `PullRequestEvent`，payload 是浅对象） |
| 9 | `detail` | `label:bug` | 按类型的判别信息——见下表 |
| 10+ | 尾列 | `15218287560\tlooks good` | 按类型追加；**有正文时正文恒为最后一列**（全文） |

### 按类型的 `detail` 列

`detail` 列让一行队列裸读即知全情，不用打开任何文件：

| 类型 | `detail` |
| --- | --- |
| `PushEvent` | 分支名（tag 推送为 `tag:v1.2.0`） |
| `CreateEvent` / `DeleteEvent` | `ref:<名>,type:branch\|tag` |
| `PullRequestReviewEvent` | `state:approved` / `changes_requested` / `commented` / `dismissed` |
| `PullRequestReviewCommentEvent` | `path:<文件>` |
| `ReleaseEvent` | `tag:<tag_name>` |
| `GollumEvent` | `page:<第一页的页名>` |
| `PullRequestEvent` | `head->base` |
| labeled / unlabeled / assigned | `label:<名>` / `assignee:<login>` |
| `MemberEvent` | `member:<login>` |
| `ForkEvent` | `to:<fork 仓全名>` |
| `DiscussionEvent` | `disc:<标题>` |
| `CommitCommentEvent` | `commit:<sha 前 7 位>` |
| `IssueCommentEvent` | 为空——评论在 PR 上时追加 `,pr` 后缀 |
| `WatchEvent` / `PublicEvent` / `SponsorshipEvent` | 为空（自身已说明） |

### 按类型的尾列

| 类型 | 尾列 |
| --- | --- |
| `IssueCommentEvent` | `<评论id>`、`<正文>` |
| `PullRequestReviewCommentEvent` | `<评论id>`、`<正文>` |
| `CommitCommentEvent` | `<评论id>`、`<正文>` |
| `IssuesEvent` | `<正文>`（issue 正文） |
| 其他类型 | 无——行到第 9 列为止 |

第 2 列是**事件** id；评论类的*评论* id 在倒数第二列。两者
别混：评论 id 用于评论级 API 调用（reaction、回复），事件
id 用于去重。

## 事件 id 语义（比较之前先读）

GitHub 事件 id 是**按事件类型独立编号**的，不是全局单调——
`IssueCommentEvent` 在 ~152 亿号段，`PushEvent` 在 ~214 亿
号段。因此：

- **绝不可跨类型数值比较 id**——一次老推送的 id 比新评论还
  大。
- 唯一安全的操作是精确匹配。内置 cursor 去重正是这样做的：
  从新到旧扫，遇到与 cursor 相等的 id 即停。

## 反查：事件 JSON 文件

行内没带你要的信息时，完整事件原文一步可达：

```
$X_CMD_ROOT_TMP/ghclaw/events/data/<owner>/<repo>/<type>.<id>.json
```

路径由第 2、3、5 列直接拼出——O(1) 反查，无需搜索。注意点：

- 文件是**pretty-printed 全量事件 JSON**（与 Events API 返回
  同构）。
- 目录**跨 target 共享**：事件 id 全局唯一，单仓监听与组织
  监听抓到同一事件会写同一文件（覆盖幂等）。
- MQ 行里 JSON `null` 字段表现为空串，`true`/`false`/数字原
  样透传。
- 两个内容层面的坑值得记住：
  - `PullRequestEvent.payload.pull_request` 是**浅对象**
    （url / id / number / head / base）——没有 title 和
    state。需要详情时另调 REST API。
  - 一次 API 调用批量加 N 个标签会产生 N 个 `labeled` 事件，
    但每个事件的 `payload.label` 都是**批量里最后一个**标签。
    权威标签集在事件 JSON 的 `payload.issue.labels`。

## 配方 1——带游标的 Python 消费者

最小模式：记住字节偏移，增量读新行，处理，落盘偏移。内置
AI 处理器也只是这个模式的一个实例。

```python
#!/usr/bin/env python3
"""最小 ghclaw MQ 消费者——打印新的 issue/PR 评论。"""
import os

MQ      = os.path.expanduser("~/.x-cmd.root/tmp/ghclaw/events/data/x-cmd/x-cmd/.mq/MQ.CURRENT/data.tsv")
CURSOR  = os.path.expanduser("~/.cache/ghclaw-demo.cursor")

pos = int(open(CURSOR).read()) if os.path.exists(CURSOR) else 0

with open(MQ) as f:
    f.seek(pos)
    for line in f:                       # 迭代追加的行
        pos = f.tell()
        cols = line.rstrip("\n").split("\t")
        if len(cols) < 9:                # 非事件行
            continue
        created, eid, etype, action, repo, actor, number, title, detail = cols[:9]
        body = cols[-1] if len(cols) > 9 else ""
        if actor.endswith("[bot]"):      # 约定：跳过机器人
            continue
        if etype == "IssueCommentEvent" and action == "created":
            print(f"{repo}#{number} 新评论 by {actor}: {body[:80]}")
        with open(CURSOR, "w") as c:     # 至少一次语义；幂等自己保
            c.write(str(pos))
```

注意：

- **至少一次（at-least-once），不是恰好一次。** 若重放会造成
  影响，请自带幂等标记（内置处理器用评论页脚和 reaction 干
  的正是这件事）。
- MQ 真实路径从 `x ghclaw listen --repo x-cmd/x-cmd` 的输出
  拿——它会打印正在追加的文件。
- 所有 target 类型同此模式，只有 `data/<target>/` 目录不同。

## 配方 2——awk 一行流

粗筛设计在 awk 里最划算——零文件 IO，一行之内回答大多数
问题：

```sh
# 新开 issue，只看真人
awk -F'\t' '$3=="IssuesEvent" && $4=="opened" && $6 !~ /\[bot\]$/ \
            {print $5"#"$7, "-", $8}' data.tsv

# 今天谁 star 了仓库（WatchEvent 没有 action 列）
awk -F'\t' '$3=="WatchEvent" {print $1, $6}' data.tsv

# 任何含 "urgent" 的评论全文（正文 = 最后一列）
awk -F'\t' '$3=="IssueCommentEvent" && tolower($NF) ~ /urgent/ \
            {print $5"#"$7, $6":"; print $NF}' data.tsv
```

## 配方 3——用你自己的 shell 函数替换派发器

`x ghclaw consume` 通过回调处理 MQ 行。运行前 export
`___X_CMD_GHCLAW_TAILER_HANDLE`，每条原始行就改调你的函数，
不再走内置流水线——MQ 文件与其他一切都不变：

```sh
# my-handlers.sh —— 在运行 consume 前 source 本文件
my_dispatch(){
    ___x_cmd_ghclaw___mq_peel "$1" || return 0   # 设置 x_mq_* 全局变量
    [ -n "$x_mq_number" ] || return 0            # 共享粗筛守卫
    case "$x_mq_actor" in *"[bot]") return 0 ;; esac

    case "$x_mq_etype/$x_mq_action" in
        PullRequestEvent/opened)
            ghclaw:info "[MY] PR opened: $x_mq_repo#$x_mq_number ($x_mq_detail)"
            # ……你的逻辑：通知、挂 CI、跨平台转发……
            ;;
        ReleaseEvent/published)
            ghclaw:info "[MY] release: $x_mq_repo $x_mq_detail"
            ;;
    esac
}

export ___X_CMD_GHCLAW_TAILER_HANDLE=my_dispatch
x ghclaw consume --repo x-cmd/x-cmd
```

`mq_peel` 提供这些全局变量（dash 兼容，无需重复解析）：
`x_mq_created_at`、`x_mq_eid`、`x_mq_etype`、`x_mq_action`、
`x_mq_repo`、`x_mq_actor`、`x_mq_number`、`x_mq_title`、
`x_mq_detail`、`x_mq_tail`（第 10 列起；无则空）。

可移植 shell 注意点（ghclaw 自己踩过的坑）：

- 别用管道解析行——管道在子 shell 里跑，变量会丢。用
  `${rest%%"$ht"*}` / `${rest#*"$ht"}` 式参数展开剥列。
- `case` 的 glob 模式里，否定字符类必须**内联字面**写
  （`[!a-zA-Z0-9_-]`）——zsh 不会把变量里存的类再当模式解析。
- bot 过滤用 `*"[bot]"` 字面匹配，别写 `*"\[bot\]"`：模式里
  被引号包住的部分按纯字面参与匹配，反斜杠会照字面命中，
  `dependabot[bot]` 会漏网。

## 给默认流水线加一种新事件

内置派发器（`___x_cmd_ghclaw___consume_watchevent`）是一个很
小的 `case`——`IssuesEvent/opened` 跑 triage 再跑 reply，
`IssueCommentEvent/created` 跑 reply，其余只记日志忽略。改
造路径：拷贝模块的 `lib/consume/_index`，加你的分支和
handler 文件，再把 `___X_CMD_GHCLAW_TAILER_HANDLE` 指向你的
版本。一处判断，一处干活。

## 自定义消费者应遵守的约定

| 约定 | 原因 |
| --- | --- |
| 跳过 `actor` 以 `[bot]` 结尾的行 | 杜绝机器人互聊。 |
| 不足 9 列的行当非事件处理 | 防御意外追加的内容。 |
| 有尾列时正文才是最后一列 | 2026-09-23 之前的旧行无尾列，剥尾列前先判空。 |
| 自带游标，按至少一次语义设计 | MQ 只增不减；重放是特性不是故障。 |
| 持久化标记自己的工作（页脚/reaction/数据库行） | 重放与重启才不会重复动作。 |
| 第 2/3/5 列拼事件 JSON 路径 | 契约规定的反查方式；不要扫目录。 |

## 清理维护

MQ 与事件 JSON 文件只增不减——没有任何自动清理。方法如下：

- **按 target 重置**——`x ghclaw stop <target> --purge` 删除
  该 target 的 `.state` 和 `.mq`（之后新监听盲启动；共享
  JSON 保留）。
- **按仓清理**——事件文件跨 target 去重，所以只能按仓清理、
  绝不能按 target：`rm -rf data/<owner>/<repo>`。
- **截断 MQ**——停掉该 target，截断或归档 `data.tsv`，若要
  让消费者从头开始，一并删除 tailer 的 index
  （`data/<target>/.mq/MQ.CURRENT/index.txt`），然后重启。
  用字节偏移做游标的外部消费者也要同步重置。
- **保留策略**——用一个 cron 任务做上面这些事，就是目前的
  "保留策略"。

ghclaw 随 x-cmd 发布——升级 x-cmd 即升级 ghclaw；升级不影
响数据目录。

## 源码与资源

- **源码：** <https://github.com/x-bash/ghclaw>
  （`lib/awk/events.awk` 写 MQ；`lib/consume/_index` 剥列与派发）
- **着陆页：** [0-ghclaw-landing.cn.md](0-ghclaw-landing.cn.md)
- **事件源深入：** [2-ghclaw-events-api-playbook.cn.md](2-ghclaw-events-api-playbook.cn.md)
- **GitHub Events API：**
  <https://docs.github.com/en/rest/activity/events>
