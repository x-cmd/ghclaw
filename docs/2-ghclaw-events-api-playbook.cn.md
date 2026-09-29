---
x-title: GitHub Events API 实战手册——ghclaw 能看到什么、看不到什么
x-desc: >-
  ghclaw 的事件源是 GitHub Events API——webhook 事件体系的一个
  小型只读投影。本文给出业务事件判断矩阵、延迟与丢失特性、
  repo / org / user 三种端点语义、events 丢失时的快照兜底方案，
  以及何时应该改用 webhook 通道。
x-sidebar: ghclaw
x-keywords: github events api, webhook, 轮询, 事件覆盖, check_run, notifications, 组织事件
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'GitHub Events API 实战手册——ghclaw 能看到什么、看不到什么'
      inLanguage: 'cn'
      about: 'ghclaw 事件源覆盖与限制'
---

# GitHub Events API 实战手册

`ghclaw` 投递的每个事件都来自同一来源：GitHub **Events
API**，按 target 轮询。这个 API 是 GitHub 全部动态的一个
小型、老式、只读投影——搞清楚它投影了什么、丢掉了什么，
就知道 `ghclaw` 能驱动哪些自动化、哪些该换别的工具。

> **配套阅读：** [着陆页](0-ghclaw-landing.cn.md) ·
> [MQ 协议](1-ghclaw-mq-custom-consumer.cn.md)

## 三个事件世界

GitHub 有三套相互重叠的事件体系，混淆它们是第一大设计
错误（比如以为 Events API 能提供 Actions `on:` 承诺的
事件——并不能）：

```
webhook 事件全集（40+ 种，snake_case）
├── Actions `on:` 触发器（绝大多数 webhook + 4 个合成触发器）
└── Events API 类型（约 17 种，PascalCase + Event 后缀，
    action 是 webhook action 的【子集】）
```

| 体系 | 形态 | 说明 |
| --- | --- | --- |
| Webhook | 40+ 类型、action 全集（`created`/`edited`/`deleted`/`synchronize`……）、payload 完整（`commits[]`、check run） | 最丰富的事件流；每个被监听仓库需要公网端点 + 管理员权限 |
| Actions `on:` | webhook 列表 + 4 个合成触发器（`schedule`、`workflow_dispatch`、`workflow_call`、`repository_dispatch`） | 这 4 个合成触发器不存在于任何事件流 |
| Events API | 约 17 种类型、action 子集 | `ghclaw` 轮询的对象；被监听仓库零配置 |

**只在** Events API 有的类型：`SponsorshipEvent`、
`MemberEvent`（webhook 里虽有，但 Actions 不能用作触发器）。
**只在** webhook 世界、因此对 `ghclaw` 不可见的类型：
`check_run`、`check_suite`、`status`、`workflow_run`、
`deployment`、`label`、`milestone`、`merge_group`、
`branch_protection_rule`、`dependabot_alert` 等 25+ 种。

## 业务事件判断矩阵

只有 Events API 可用时，怎么判断具体业务事件发生了
（`ghclaw` 分类器编码的正是这套规则）：

| 业务事件 | 首选信号 | 兜底 / 备注 |
| --- | --- | --- |
| issue 新开 | `IssuesEvent` `opened` | issues 快照出现新 number |
| issue 关闭 | `IssuesEvent` `closed` | `state_reason` 区分完成/未计划 |
| issue 重开 | `IssuesEvent` `reopened` | 快照状态迁移 |
| issue 打标签/指派 | `IssuesEvent` `labeled`/`assigned` + payload | 批量打标签时每个事件重复最后一个标签 |
| issue 新评论 | `IssueCommentEvent` `created` | issue payload 的 `pull_request` 字段区分 issue/PR |
| 评论**被编辑** | **不可见** | 最接近的——comments 端点 `updated_at` > `created_at` |
| 评论**被删除** | **任何端点都看不到** | 只能表现为计数对不上 |
| PR 开/合并/关 | `PullRequestEvent` `opened`/`merged`/`closed` | pulls 快照（payload 本身是浅对象） |
| PR review 提交 | `PullRequestReviewEvent`——结论看 `review.state`：approved / changes_requested / commented | action 是 created/updated/dismissed（不是 webhook 的 submitted） |
| PR 行内评论 | `PullRequestReviewCommentEvent` `created` | detail 列有 `path:` |
| PR 转 ready | **events 里没有** | pulls 快照 `draft` true→false |
| push | `PushEvent`（无 action；ref/before/after） | 提交明细需另拉 compare |
| release 发布 | `ReleaseEvent` `published` | payload 有 `prerelease` |
| star / fork | `WatchEvent` `started` / `ForkEvent` `forked` | — |
| 建删分支/tag | `CreateEvent` / `DeleteEvent` | `ref:` + `type:branch\|tag` |
| wiki 改动 | `GollumEvent` | 页面删除不报告 |
| 新增协作者 | `MemberEvent` `added` | 触发于邀请*被接受*时 |
| 私有仓转公开 | `PublicEvent` | 安全敏感 |
| 新讨论 | `DiscussionEvent` `created` | 讨论**的评论**没有任何事件 |
| **有人 @ 我** | **没有此信号**——但在被监听的仓里评论事件本身可见；可在正文（MQ 最后一列）关键词匹配 `@你` | mention *通知语义* 只在 Notifications API（`reason=mention`） |
| **请求我 review** | **events 里没有** | Notifications API `reason=review_requested` |
| **CI 结果** | **events 里没有** | Notifications API `reason=ci_activity` |

## 延迟与丢失特性

实测行为，做触发器设计前先内化：

| 方面 | 现实 |
| --- | --- |
| API / 网页操作 | 秒级 ~ 1 分钟——交互式自动回复够用 |
| git 推送（`PushEvent`） | feed 管道延迟 **10~40 分钟**；偶发丢失 |
| 事件丢失 | 官方明确高负载不保证送达；每条时间线窗口 ~300 条 / 30 天 |
| 每轮翻页 | 只读第一页（100 条）——单轮爆发超出会丢尾部，记为 cursor 缺口告警 |
| 能否做审计依据 | **不能。** 把事件流当"最终信息化"，不是账本 |

快照端点（`/issues`、`/pulls`、`/issues/{n}/comments`、
`/pulls/{n}/comments`、`/releases`）是设计好的兜底：ETag
判变后与自己上次存的快照 diff——状态迁移、评论计数差、
时间戳启发式（`updated_at` > `created_at`）能重建大部分
events 看不到的东西。代价是每端点一次请求，所以 ghclaw
主线保持 events-only，对账留给自定义消费者和路线图。

## 端点范围——repo、org、user

三条时间线，三种语义（`ghclaw` 以 `--repo`、`--org`、
`--user` 暴露）：

| 端点 | 回答 | 坑 |
| --- | --- | --- |
| `/repos/{o}/{r}/events` | "这个仓发生了什么" | 公共或私有，取决于 token 可读范围 |
| `/orgs/{org}/events` | "这个组织名下所有仓发生了什么" | **仅公共事件**——端点名字就叫 list-public-organization-events；私有仓仍要逐仓。全组织共享一个 300 条窗口：活跃仓会把安静仓的事件挤出窗口（繁忙组织把间隔缩到 10~30s） |
| `/users/{name}/events` | "这个人在 GitHub 上干了什么" | 以人为中心：只含该用户*触发*的事件（跨全站）。其仓库里其他人的活动不可见——替代不了逐仓监听。**监听自己也抓不到 @你的消息**（触发者是评论者而不是你）——要监听相关仓库并在正文里匹配 |

组织级 1 请求 vs N 条单仓监听线：

| 维度 | 组织级 1 请求 | N 条单仓轮询 |
| --- | --- | --- |
| 每周期请求数 | 1 | N |
| 时间线 | 一条，跨仓有序 | N 条各自独立 |
| 300 条窗口 | 共享（活跃挤安静） | 每仓独立 |
| 缺口爆炸半径 | 一个缺口波及所有仓 | 只影响该仓 |
| 新增公共仓 | 自动覆盖 | 要加监听 |
| 私有仓 | 看不到 | 逐仓可见 |
| 间隔 | 全组织一刀切 | 可按仓定 |

**个人账号没有**等价批量端点——想监听"某人名下所有仓"，
只能一个仓一条监听线。

## 轮询 vs webhook——诚实的对照

| 维度 | 轮询（ghclaw 的做法） | webhook |
| --- | --- | --- |
| 接入成本 | 零——有读权限即可 | 公网 HTTPS 端点 + 逐仓管理员（或组织 webhook / GitHub App） |
| 能监听别人的仓 | 能 | 不能 |
| 延迟 | 轮询间隔（秒~分钟级） | 通常 <1s~数秒，无 SLA |
| 送达 | 会丢但*最终可续*（ETag/cursor；缺口可检测告警） | at-most-once、无顺序、10 秒 ack 窗口——端点宕机即丢 |
| 离线行为 | 停机即盲、从"现在"起算（新监听不回填） | 端点宕机期间永久丢失 |
| 事件类型 | 约 17 种、action 子集 | 40+ 种、action 全集 |
| 速率成本 | 每周期 1 次条件请求（304 免费） | 送达本身免费 |
| 攻击面 | 无（纯出站） | 验签、防重放、secret 轮换 |
| 审计痕迹 | 自留 MQ + JSON 文件 | GitHub Recent Deliveries 面板 |

互补式设计——也是 `ghclaw` 的路线图方向——是**混合方案**：
webhook（前面挂一个小的接收端，如 Cloudflare Worker，验
`X-Hub-Signature-256`、落盘即返、10 秒内 ack）把你自己管
理的仓库的事件写入**同一个 MQ 格式**；轮询覆盖其余一切，
并为 webhook 的丢失做对账层。webhook"实时但脆弱"与轮询
"慢但皮实"正好互补短板。

## 值得盯防的安全类事件

Events API 里有两种纯安全信号，天然适合告警型消费者
（关键点：`PublicEvent`、`MemberEvent` 在 Actions `on:` 里
都用不了）：

- `PublicEvent`——私有仓刚转公开。立即告警；这很少是故
  意的。
- `MemberEvent`——新增协作者（邀请被接受时触发）。审计
  "谁进了哪个仓库"。
- `GollumEvent`——敏感页面（部署文档等）的 wiki 改动；注
  意它看不到页面删除。

## ghclaw 用这些信息做了什么

- **主线**：只轮询 events 时间线——每周期一次条件请求，
  cursor 去重，事件追加进 MQ。
- **分类与派发**：`IssuesEvent/opened` → AI 分类 + `@x`
  回复；`IssueCommentEvent/created` → `@x` 回复。
- **其余一切**：只记日志，留在 MQ 里等你的消费者——这就
  是 [MQ 协议](1-ghclaw-mq-custom-consumer.cn.md)的用武
  之地。

## 源码与资源

- **源码与设计笔记：** <https://github.com/x-bash/ghclaw>
  （`.x-cmd/story/` 存有本文取材的 API 调研与轮询/webhook
  权衡研究）
- **GitHub Events API：**
  <https://docs.github.com/en/rest/activity/events>
- **Webhook 事件与 payload：**
  <https://docs.github.com/en/webhooks/webhook-events-and-payloads>
- **Notifications API：**
  <https://docs.github.com/en/rest/activity/notifications>
