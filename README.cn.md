# x-cmd/ghclaw — `ghclaw` 着陆页

`ghclaw` 是一种用于 **GitHub 事件** 的 "claw 设计"—— 一个
x-cmd 模块：轮询 GitHub Events API，把事件落盘为只追加的 MQ，
再派发给 AI 辅助的分类与回复（一套简单默认方案）或你自己的
消费者。

> 🌐 **English version: [README.md](./README.md)** — same
> content, English front matter.

本仓库托管 `ghclaw` 的标准文档 —— 它是什么、claw 设计如何工
作、如何基于 MQ 构建你自己的消费者。文章为内容导向，欢迎任
何人提交 **修改 PR** — 见 [`CONTRIBUTING.md`](./CONTRIBUTING.md)。

## 仓库结构

```
x-cmd/ghclaw/
├── README.md                 # 本文件（英文）
├── README.cn.md              # 中文版
├── CONTRIBUTING.md           # 文章写作流程 + frontmatter 规范
├── SKILL.md                  # AI agent 配方
├── LICENSE                   # Apache-2.0
└── docs/
    ├── 0-ghclaw-landing.{en,cn}.md       # 着陆页
    ├── 0-ghclaw-landing.llms.md
    ├── 0-ghclaw-landing.faq.yml
    ├── 1-ghclaw-mq-custom-consumer.*     # MQ 协议与自定义消费者
    └── 2-ghclaw-events-api-playbook.*    # 事件源覆盖与限制
```

每个槽位 `n-<slug>` 都是标准的四文件集合 —— `{en,cn}.md` 加
`.llms.md` 加 `.faq.yml`。前缀数字是阅读顺序：`0-` 是着陆页；
`1-` 讲解面向自定义消费者的 MQ 协议；`2-` 梳理 GitHub Events
API 能给什么、给不了什么。后续槽位（3-、……）可用于更深的话
题（webhook 通道、GitLab/Gitea 变体）。

## 姊妹仓库

- [`x-cmd/cve`](https://github.com/x-cmd/cve) — 专题文库模式参考。
- [`x-cmd/terminal`](https://github.com/x-cmd/terminal) — 终端专题文库。
- [`x-cmd/browser`](https://github.com/x-cmd/browser) — 浏览器专题文库。
- [`x-cmd/x-cmd`](https://github.com/x-cmd/x-cmd) — 模块源码（`mod/ghclaw/`）。
- [`x-bash/ghclaw`](https://github.com/x-bash/ghclaw) — 源码级参考。

## 许可

Apache License 2.0 — 见 [`LICENSE`](./LICENSE)。