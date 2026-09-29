---
name: ghclaw
description: ghclaw — a claw design for GitHub events, shipped as the x-cmd module `x ghclaw`. It polls the GitHub Events API (repo / org / user targets) into an on-disk append-only MQ (TSV + per-event JSON) and dispatches events to handlers — the built-in x ai triage/@x reply being a simple DEFAULT set; the main path is user-defined consumers reading the MQ. Use when the user asks about "ghclaw", "GitHub event listener", "AI-assisted GitHub triage", "GitHub event queue", or "claw design".
metadata: type=landing-page, source=team-curated, schema=4-tuple-md, refresh=manual, license=apache-2.0, scope=ghclaw-design
---

# x-cmd/ghclaw — using the docs

## 1. Read on the website

The docs are published at <https://x-cmd.com/ghclaw>.

## 2. Use the raw files

Plain markdown + YAML, served over HTTPS from GitHub. Fetch
directly from `https://raw.githubusercontent.com/x-cmd/ghclaw/main/...`
— do not route through any CDN or proxy.

```sh
curl -fsSL "https://raw.githubusercontent.com/x-cmd/ghclaw/main/docs/0-ghclaw-landing.en.md"
```

## Article schema

Every slot `n-<slug>` is a canonical four-file set, kept in
sync (`en`↔`cn` in the same commit):

| File | Shape | Purpose |
| --- | --- | --- |
| `n-<slug>.en.md` | Markdown with YAML frontmatter. | Canonical English article. |
| `n-<slug>.cn.md` | Same as above, in Chinese. | Chinese translation. |
| `n-<slug>.llms.md` | YAML frontmatter + flat structured sections. | LLM-friendly summary. |
| `n-<slug>.faq.yml` | Bilingual Q&A with `confidence` and `reference`. | FAQ + JSON-LD. |

Current slots: `0-ghclaw-landing` (the product), `1-ghclaw-mq-custom-consumer`
(the MQ protocol + custom consumers), `2-ghclaw-events-api-playbook`
(event-source coverage & limits).

## What `ghclaw` is

`ghclaw` is a "claw design" for **GitHub events**, shipped as
the x-cmd module `x ghclaw`. A background `listen` worker
polls the GitHub **Events API** per target (`--repo owner/repo`,
`--org <org>` = public events only, `--user <name>` =
actor-centric stream) with ETag/304 conditional requests,
dedups by event-id cursor, and appends each new event to an
on-disk **append-only MQ** (`data/<target>/.mq/MQ.CURRENT/data.tsv`,
one TSV line per event) plus a full JSON per event
(`data/<owner>/<repo>/<type>.<id>.json`). A `consume` tailer
dispatches MQ lines to handlers.

The built-in handlers — `x ai triage` (classify + label new
issues) and `x ai reply` (`@x` keyword auto-reply) — are a
**simple default set** (reference implementations, evolving).
The stable contract is the MQ itself: users read it from
Python/awk/anything with their own cursor, or replace the
dispatcher wholesale via `___X_CMD_GHCLAW_TAILER_HANDLE`.
Fresh listeners start **blind** (cursor at the newest event,
no history backfill). Built-in handlers are idempotent via
`ghclaw` footers and 👀 reactions; tokens are never exported.

The "claw" metaphor: each event is grabbed by a *claw* that
sorts it, classifies it, and either drops it, replies to it,
or escalates it.

## Sources

- <https://github.com/x-cmd/ghclaw> — this repo (docs).
- <https://x-cmd.com/ghclaw> — published site.
- [`x-bash/ghclaw`](https://github.com/x-bash/ghclaw) — source-level reference (`.x-cmd/story/` holds the design notes the docs are verified against).
- [`x-cmd/x-cmd`](https://github.com/x-cmd/x-cmd) — module source (`mod/ghclaw/`).
- [`x-cmd/cve/SKILL.md`](https://github.com/x-cmd/cve/blob/main/SKILL.md) — parallel topic-library pattern.
- [`x-cmd/terminal/SKILL.md`](https://github.com/x-cmd/terminal/blob/main/SKILL.md) — parallel topic-library pattern.
- [`x-cmd/browser/SKILL.md`](https://github.com/x-cmd/browser/blob/main/SKILL.md) — parallel topic-library pattern.
