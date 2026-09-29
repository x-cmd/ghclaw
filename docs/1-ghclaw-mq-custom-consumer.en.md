---
x-title: The ghclaw MQ Protocol — Build Your Own Consumer
x-desc: >-
  The ghclaw MQ is an append-only TSV file — the public interface
  between the listener and your code. Line format, per-type
  trailing columns, event-JSON reverse lookup, and recipes for
  Python / awk / shell consumers, plus replacing the built-in
  dispatcher with your own handler.
x-sidebar: ghclaw
x-keywords: ghclaw, mq, tsv, event queue, custom consumer, python, tailer, watcher
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'ghclaw MQ protocol — build your own consumer'
      inLanguage: 'en'
      about: 'ghclaw MQ protocol and custom consumers'
---

# The ghclaw MQ Protocol — Build Your Own Consumer

`x ghclaw run`'s built-in AI handlers (triage + `@x` reply) are
just the **default primitive pipeline** — one way to react to
events, shipped so the common case works out of the box. The
real contract between `ghclaw` and your code is a single file:
the **MQ**, an append-only TSV queue.

Start a listener, let it accumulate events, then read the MQ
from **any language** and implement exactly the events and logic
you care about. This article is the protocol reference. The
default handlers themselves evolve across releases — the MQ
contract documented here is the stable part.

> **Companion:** [ghclaw landing page](0-ghclaw-landing.en.md)
> covers the commands, AI handlers and limits. Here we go deep
> on the data contract.

## The contract in one paragraph

- One file per target:
  `$X_CMD_ROOT_TMP/ghclaw/events/data/<target>/.mq/MQ.CURRENT/data.tsv`
  (the `listen` command prints the exact path on start).
- **Append-only**: new events are appended; nothing is deleted
  or rewritten. Multiple consumers can read with independent
  cursors; replay is just re-reading.
- One event per line, **TAB-separated**, no bare TAB or newline
  inside a column (whitespace in bodies is flattened to single
  spaces).
- Everything a consumer needs for a **coarse filter** is on the
  line itself. Reach for the event JSON only when you need full
  context.

## Line format

Nine fixed leading columns, then type-specific trailing columns:

| # | Column | Example | Meaning |
| --- | --- | --- | --- |
| 1 | `created_at` | `2026-09-17T11:23:53Z` | GitHub server time, ISO 8601 UTC |
| 2 | `id` | `15214943014` | Event id — the dedup/cursor anchor |
| 3 | `type` | `IssuesEvent` | PascalCase event type |
| 4 | `action` | `opened` | Webhook-action subset; empty when the type has none |
| 5 | `repo` | `qiakai/test-ghclaw` | `owner/repo` — the routing key |
| 6 | `actor` | `lunrenyi` | Who triggered it; bots end in `[bot]` |
| 7 | `number` | `5` | Issue / PR number; empty when N/A |
| 8 | `title` | `test: event batch1` | Issue / discussion / release title; empty for types without one (notably `PullRequestEvent`, whose payload is shallow) |
| 9 | `detail` | `label:bug` | Per-type discriminator — see table below |
| 10+ | trailing | `15218287560\tlooks good` | Type-specific; **when a body is present it is always the LAST column** (full text) |

### The `detail` column, per type

`detail` exists so a queue row reads as a full story without
opening any other file:

| Type | `detail` |
| --- | --- |
| `PushEvent` | branch name (`tag:v1.2.0` for tags) |
| `CreateEvent` / `DeleteEvent` | `ref:<name>,type:branch\|tag` |
| `PullRequestReviewEvent` | `state:approved` / `changes_requested` / `commented` / `dismissed` |
| `PullRequestReviewCommentEvent` | `path:<file>` |
| `ReleaseEvent` | `tag:<tag_name>` |
| `GollumEvent` | `page:<first page name>` |
| `PullRequestEvent` | `head->base` |
| labeled / unlabeled / assigned | `label:<name>` / `assignee:<login>` |
| `MemberEvent` | `member:<login>` |
| `ForkEvent` | `to:<fork full_name>` |
| `DiscussionEvent` | `disc:<title>` |
| `CommitCommentEvent` | `commit:<sha, 7 chars>` |
| `IssueCommentEvent` | empty — plus `,pr` suffix when the comment is on a PR |
| `WatchEvent` / `PublicEvent` / `SponsorshipEvent` | empty (self-explanatory) |

### Trailing columns, per type

| Type | Trailing columns |
| --- | --- |
| `IssueCommentEvent` | `<comment_id>`, `<body>` |
| `PullRequestReviewCommentEvent` | `<comment_id>`, `<body>` |
| `CommitCommentEvent` | `<comment_id>`, `<body>` |
| `IssuesEvent` | `<body>` (the issue body) |
| all others | none — the line ends at column 9 |

Column 2 is the **event** id; for comment types the *comment*
id sits in the second-to-last column. Don't confuse them:
comment ids anchor comment-level API calls (reactions, replies),
event ids anchor dedup.

## Event-id semantics (read before you compare)

GitHub event ids are **numbered per event type**, not globally
monotonic — `IssueCommentEvent` lives around `15.2e9` while
`PushEvent` sits around `21.4e9`. So:

- **Never compare ids numerically across types** — an old push
  has a bigger id than a brand-new comment.
- Exact-match equality is the only safe operation. That is
  exactly how the built-in cursor dedup works: scan newest →
  oldest, stop at the id equal to the cursor.

## Reverse lookup: the event JSON files

When a line doesn't carry what you need, the full event payload
is one file away:

```
$X_CMD_ROOT_TMP/ghclaw/events/data/<owner>/<repo>/<type>.<id>.json
```

The path is built directly from columns 2, 3 and 5 — an O(1)
lookup, no search. Notes:

- Files hold the **pretty-printed full event JSON** (the same
  shape the Events API returns).
- The directory is **shared across targets**: event ids are
  globally unique, so a repo listener and an org listener
  writing the same event produce the same file (overwrite is
  idempotent).
- A JSON `null` field surfaces as an empty string in the MQ
  line, and `true`/`false`/numbers pass through as-is.
- Two content caveats worth remembering:
  - `PullRequestEvent.payload.pull_request` is a **shallow**
    object (url / id / number / head / base) — no title, no
    state. Fetch details via the REST API when you need them.
  - A batch-add of N labels produces N `labeled` events whose
    `payload.label` is always the **last** label of the batch.
    The authoritative label set is `payload.issue.labels` in
    the event JSON.

## Recipe 1 — Python consumer with its own cursor

The minimal pattern: remember a byte offset, read new lines as
they appear, act, persist the offset. The built-in AI handlers
are just one instance of this pattern.

```python
#!/usr/bin/env python3
"""Minimal ghclaw MQ consumer — prints new issue/PR comments."""
import os

MQ      = os.path.expanduser("~/.x-cmd.root/tmp/ghclaw/events/data/x-cmd/x-cmd/.mq/MQ.CURRENT/data.tsv")
CURSOR  = os.path.expanduser("~/.cache/ghclaw-demo.cursor")

pos = int(open(CURSOR).read()) if os.path.exists(CURSOR) else 0

with open(MQ) as f:
    f.seek(pos)
    for line in f:                       # iterates appended lines
        pos = f.tell()
        cols = line.rstrip("\n").split("\t")
        if len(cols) < 9:                # not an event line
            continue
        created, eid, etype, action, repo, actor, number, title, detail = cols[:9]
        body = cols[-1] if len(cols) > 9 else ""
        if actor.endswith("[bot]"):      # convention: skip bots
            continue
        if etype == "IssueCommentEvent" and action == "created":
            print(f"{repo}#{number} new comment by {actor}: {body[:80]}")
        with open(CURSOR, "w") as c:     # at-least-once; keep your own guards
            c.write(str(pos))
```

Notes:

- **At-least-once, not exactly-once.** Keep your own idempotency
  mark if reprocessing would hurt (the built-in handlers use
  comment footers and reactions for exactly this).
- Get the real MQ path from the output of
  `x ghclaw listen --repo x-cmd/x-cmd` — it prints the file it
  is appending to.
- The same pattern works for every target kind; only the
  `data/<target>/` directory changes.

## Recipe 2 — awk one-liners

The coarse-filter design pays off most in awk — no file IO, the
line alone answers most questions:

```sh
# New issues, human-authored only
awk -F'\t' '$3=="IssuesEvent" && $4=="opened" && $6 !~ /\[bot\]$/ \
            {print $5"#"$7, "-", $8}' data.tsv

# Who starred the repo today (WatchEvent has no action column)
awk -F'\t' '$3=="WatchEvent" {print $1, $6}' data.tsv

# Full text of any comment mentioning "urgent" (body = last column)
awk -F'\t' '$3=="IssueCommentEvent" && tolower($NF) ~ /urgent/ \
            {print $5"#"$7, $6":"; print $NF}' data.tsv
```

## Recipe 3 — replace the dispatcher with your shell function

`x ghclaw consume` processes MQ lines through a callback. Export
`___X_CMD_GHCLAW_TAILER_HANDLE` before running it and every raw
line goes to your function instead of the built-in pipeline —
same MQ file, same everything else:

```sh
# my-handlers.sh — source this before running consume
my_dispatch(){
    ___x_cmd_ghclaw___mq_peel "$1" || return 0   # sets x_mq_* globals
    [ -n "$x_mq_number" ] || return 0            # shared coarse guard
    case "$x_mq_actor" in *"[bot]") return 0 ;; esac

    case "$x_mq_etype/$x_mq_action" in
        PullRequestEvent/opened)
            ghclaw:info "[MY] PR opened: $x_mq_repo#$x_mq_number ($x_mq_detail)"
            # ... your logic: notify, queue CI, cross-post ...
            ;;
        ReleaseEvent/published)
            ghclaw:info "[MY] release: $x_mq_repo $x_mq_detail"
            ;;
    esac
}

export ___X_CMD_GHCLAW_TAILER_HANDLE=my_dispatch
x ghclaw consume --repo x-cmd/x-cmd
```

`mq_peel` gives you these globals (dash-compatible; no
re-parsing needed): `x_mq_created_at`, `x_mq_eid`, `x_mq_etype`,
`x_mq_action`, `x_mq_repo`, `x_mq_actor`, `x_mq_number`,
`x_mq_title`, `x_mq_detail`, `x_mq_tail` (columns 10+; empty
when absent).

Portable-shell notes (learned the hard way in ghclaw itself):

- Don't parse the line through a pipe — a pipeline runs in a
  subshell and your variables vanish. Peel with
  `${rest%%"$ht"*}` / `${rest#*"$ht"}`-style parameter expansion.
- In `case` glob patterns, the negated character class must be
  written inline (`[!a-zA-Z0-9_-]`) — zsh won't re-interpret a
  class stored in a variable.
- Match bot actors literally with `*"[bot]"`, not `*"\[bot\]"`:
  quoted text in a pattern is literal, so the backslash would be
  matched verbatim and `dependabot[bot]` would slip through.

## Adding a new event to the default pipeline

The built-in dispatcher (`___x_cmd_ghclaw___consume_watchevent`)
is a small `case` — `IssuesEvent/opened` runs triage then reply,
`IssueCommentEvent/created` runs reply, everything else is
logged and ignored. The fork-and-patch path: copy the module's
`lib/consume/_index`, add your branch and handler file, and
point `___X_CMD_GHCLAW_TAILER_HANDLE` at your version. One place
judges, one place works.

## Conventions to honor in your own consumers

| Convention | Why |
| --- | --- |
| Skip `actor` ending in `[bot]` | No bot ping-pong. |
| Treat lines without 9 columns as non-events | Robustness against stray appends. |
| Body = last column only when present | Pre-2026-09-23 rows have no trailing columns; check before peeling. |
| Keep your own cursor; expect at-least-once | The MQ never shrinks; replays are a feature. |
| Mark your work durably (footer / reaction / DB row) | So replays and restarts don't double-act. |
| Columns 2/3/5 → event JSON path | The documented reverse-lookup; don't scan directories. |

## Housekeeping

The MQ and the event JSON files grow forever — nothing is
pruned automatically. Recipes:

- **Per-target reset** — `x ghclaw stop <target> --purge`
  removes that target's `.state` and `.mq` (a fresh listener
  then starts blind; the shared JSON stays).
- **Per-repo cleanup** — event files are deduplicated across
  targets, so prune by repo, never by target:
  `rm -rf data/<owner>/<repo>`.
- **Truncate the MQ** — stop the target, truncate or archive
  `data.tsv`, delete the tailer index
  (`data/<target>/.mq/MQ.CURRENT/index.txt`) if consumers
  should start over, then restart. External consumers keeping
  byte-offset cursors must reset those too.
- **Retention** — a cron job doing the above is the intended
  "retention policy" for now.

ghclaw ships with x-cmd — upgrading x-cmd upgrades it; the data
directory is untouched by upgrades.

## Source & Resources

- **Source:** <https://github.com/x-bash/ghclaw>
  (`lib/awk/events.awk` writes the MQ; `lib/consume/_index`
  peels and dispatches it)
- **Landing page:** [0-ghclaw-landing.en.md](0-ghclaw-landing.en.md)
- **Event-source deep dive:** [2-ghclaw-events-api-playbook.en.md](2-ghclaw-events-api-playbook.en.md)
- **GitHub Events API:** <https://docs.github.com/en/rest/activity/events>
