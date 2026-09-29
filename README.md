# x-cmd/ghclaw — `ghclaw` landing page

`ghclaw` is a "claw design" for **GitHub events** — an x-cmd
module that polls the GitHub Events API into an on-disk
append-only MQ, then dispatches events to AI-assisted triage
and reply (a simple default set) or to your own consumers.

> 🌐 **中文版：[README.cn.md](./README.cn.md)** — same content,
> Chinese front matter.

This repo hosts the canonical documentation for `ghclaw` —
what it is, how the claw design works, and how to build your
own consumers on the MQ. Articles are content-only and open
for **modification PRs** from anyone — see
[`CONTRIBUTING.md`](./CONTRIBUTING.md).

## What's in this repo

```
x-cmd/ghclaw/
├── README.md                 # this file (English)
├── README.cn.md              # Chinese version
├── CONTRIBUTING.md           # article workflow + frontmatter spec
├── SKILL.md                  # AI-agent recipe
├── LICENSE                   # Apache-2.0
└── docs/
    ├── 0-ghclaw-landing.{en,cn}.md       # the landing page
    ├── 0-ghclaw-landing.llms.md
    ├── 0-ghclaw-landing.faq.yml
    ├── 1-ghclaw-mq-custom-consumer.*     # the MQ protocol + custom consumers
    └── 2-ghclaw-events-api-playbook.*    # event-source coverage & limits
```

Each slot `n-<slug>` is a canonical four-file set — `{en,cn}.md`
plus `.llms.md` plus `.faq.yml`. The leading integer is the
reading order: `0-` is the landing page; `1-` documents the
MQ protocol for building custom consumers; `2-` maps what the
GitHub Events API can and cannot deliver. Later slots (3-, …)
can be added for deeper topics (the webhook channel,
GitLab/Gitea variants).

## Sister repos

- [`x-cmd/cve`](https://github.com/x-cmd/cve) — topic library pattern reference.
- [`x-cmd/terminal`](https://github.com/x-cmd/terminal) — terminal topic library.
- [`x-cmd/browser`](https://github.com/x-cmd/browser) — browser topic library.
- [`x-cmd/x-cmd`](https://github.com/x-cmd/x-cmd) — module source (`mod/ghclaw/`).
- [`x-bash/ghclaw`](https://github.com/x-bash/ghclaw) — the source-level reference.

## License

Apache License 2.0 — see [`LICENSE`](./LICENSE).