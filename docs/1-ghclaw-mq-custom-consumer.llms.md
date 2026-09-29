---
name: 1-ghclaw-mq-custom-consumer
description: The ghclaw MQ protocol reference — the append-only TSV file is the public interface between the listener and your code. Line format (9 fixed columns + type-specific trailing columns, body always last), per-type detail column, event-JSON reverse lookup via data/<owner>/<repo>/<type>.<id>.json, event-id semantics (per-type numbering, exact match only), and consumer recipes for Python (own cursor), awk one-liners, and shell (___X_CMD_GHCLAW_TAILER_HANDLE replaces the built-in dispatcher; mq_peel globals x_mq_*).
type: summary
---

# Core Content

core_features:
  - MQ = one append-only TSV per target at data/<target>/.mq/MQ.CURRENT/data.tsv; append-only means multi-consumer, replayable, cursor-based reads in any language
  - 9 fixed columns (created_at, id, type, action, repo, actor, number, title, detail) + type-specific trailing columns; when a body is present it is ALWAYS the last column
  - detail column carries per-type discriminators (branch, tag:, state:, path:, head->base, label:, member:, to:, page:, disc:, commit:; IssueCommentEvent gets ',pr' suffix on PRs)
  - Comment types add <comment_id> + <body> as trailing columns; IssuesEvent adds <body>; comment id sits in the second-to-last column (event id is column 2 — don't confuse)
  - Full payload reverse-lookup: data/<owner>/<repo>/<type>.<id>.json built from columns 2/3/5 (O(1), shared across targets, ids globally unique)
  - Event ids are numbered per type — exact-match equality only, never numeric comparison

# Consumer Recipes

recipes:
  - python: remember a byte offset as your cursor, iterate appended lines, split on TAB, filter by type/action/actor, act, persist offset; expect at-least-once and keep your own idempotency mark
  - awk: -F'\t' filters on $3 type / $4 action / $6 actor; $NF is the body; zero file IO needed for coarse filtering
  - shell: export ___X_CMD_GHCLAW_TAILER_HANDLE=<func> before 'x ghclaw consume'; the func receives each raw line and can use ___x_cmd_ghclaw___mq_peel to get x_mq_created_at/eid/etype/action/repo/actor/number/title/detail/tail globals
  - fork-and-patch: the built-in dispatcher is a small case on etype/action — copy lib/consume/_index, add your branch + handler, point the env var at it

# Conventions

conventions:
  - Skip actors ending in [bot] (literal glob *"[bot]")
  - Treat lines with fewer than 9 columns as non-events
  - Pre-2026-09-23 rows have no trailing columns — check before peeling the body
  - Keep your own cursor; durable marks (footer / reaction / DB row) make replays safe
  - Shell pitfalls: no pipe-subshell peeling (variables vanish); zsh needs inline negated glob classes; case patterns quote literals
  - Content caveats: PullRequestEvent payload is shallow (no title/state); batch labeling repeats the last label in every event

# Housekeeping

housekeeping:
  - MQ and event JSON grow forever; nothing is pruned automatically
  - Per-target reset: x ghclaw stop <target> --purge removes .state and .mq (shared JSON untouched)
  - Shared JSON is pruned per repo only (rm -rf data/<owner>/<repo>), never per target
  - MQ truncation: stop target, truncate data.tsv, delete the tailer index.txt if consumers should restart, reset external byte-offset cursors
  - ghclaw ships with x-cmd; upgrading x-cmd upgrades it; data untouched by upgrades

# Related Resources

official:
  source: https://github.com/x-bash/ghclaw
  mq_writer: lib/awk/events.awk
  dispatcher: lib/consume/_index
related:
  - name: ghclaw landing page
    url: docs/0-ghclaw-landing.en.md
  - name: GitHub Events API
    url: https://docs.github.com/en/rest/activity/events

# Summary

The ghclaw MQ is a deliberately boring append-only TSV file — the real interface under the built-in AI handlers. Each event is one TAB-separated line: created_at, id, type, action, repo, actor, number, title, detail, plus type-specific trailing columns with the body always last when present. Consumers coarse-filter from the line alone (type/action/actor/title/keywords), then reverse-lookup the full event JSON at data/<owner>/<repo>/<type>.<id>.json when needed. Event ids are per-type numbered, so only exact matching is meaningful. You can read the MQ from Python with your own byte-offset cursor, filter it with awk one-liners, or replace the whole per-line dispatcher via ___X_CMD_GHCLAW_TAILER_HANDLE using the mq_peel globals — the MQ file itself is never touched either way. Honor the conventions: skip [bot] actors, treat short lines as non-events, design for at-least-once, and mark your work durably so replays never double-act.
