---
name: 0-ghclaw-landing
description: ghclaw is an x-cmd module that polls the GitHub Events API (repo / org / user targets), appends events to an on-disk append-only MQ (TSV), and dispatches them to built-in AI agents — x ai triage (classify + label new issues) and x ai reply (@x keyword auto-reply). No webhook or admin rights needed; results post back via gh api. Custom consumers can read the MQ with their own cursor or replace the dispatcher via ___X_CMD_GHCLAW_TAILER_HANDLE.
type: summary
---

# Core Content

core_features:
  - Polls the GitHub Events API with conditional requests (ETag / 304 — idle polls cost no rate limit), dedup by event-id cursor
  - Every event lands as one line in an append-only MQ.tsv plus a pretty JSON file — debuggable, replayable, multi-consumer
  - Built-in AI triage for new issues (x ai triage): skips already-labeled issues, AI picks only from existing repo labels, posts priority/area/tldr + applies labels
  - Built-in AI auto-reply (x ai reply) on the @x keyword (GHCLAW_REPLY_KEYWORD to change), strict word boundaries, CJK-aware language detection
  - Three-layer anti-loop: [bot] actor filter, ghclaw footer check, eyes-reaction idempotency mark on the trigger object
  - Token hygiene: GH_TOKEN/GITHUB_TOKEN, gh via x env use gh, token passed per-invocation and never exported
  - Custom consumers: read the MQ TSV with your own cursor, or replace the per-line dispatcher with ___X_CMD_GHCLAW_TAILER_HANDLE
  - The built-in triage/reply handlers are a simple DEFAULT set (reference implementations, evolving) — the MQ contract is the stable interface
  - Prerequisites: GH_TOKEN (write access needed for the built-in reply/triage; read suffices for watch-only) and a user-configured x ai provider (e.g. x minimax apikey=<key>)
  - Long-running by design: listen = x worker background process managed via x ghclaw ls / stop [--purge]

# Commands

commands:
  - x ghclaw run --repo owner/repo [--interval 1min]     # foreground all-in-one: listen + consume + stop
  - x ghclaw listen --repo a/b | --org org | --user name # background listener appending to the MQ (one per target)
  - x ghclaw consume --repo a/b                          # foreground dispatcher to built-in AI handlers (Ctrl-C to exit)
  - x ghclaw ls                                          # list active listeners (target / pid / start time / state)
  - x ghclaw stop <target> [--purge]                     # stop; keep MQ+state unless --purge; stop --all for everything
  - Auth: export GH_TOKEN (or GITHUB_TOKEN)              # token must be able to read the watched repos

# Targets

targets:
  - --repo owner/repo: single repo, public or private as the token permits
  - --org <org>: whole organization, PUBLIC events only (private repos stay per-repo)
  - --user <name>: actor-centric activity stream — what the user triggered, not what happened in their repos
  - --interval: human time (30s, 1min, 2h30m), default 60s; targets normalized to lowercase

# Data Layout

storage:
  root: $X_CMD_ROOT_TMP/ghclaw/events/
  state: data/<target>/.state/                 # etag, lms, cursor, body, hdr
  mq: data/<target>/.mq/MQ.CURRENT/data.tsv    # append-only, one event per TAB-separated line
  events: data/<owner>/<repo>/<type>.<id>.json # full payload per event, shared across targets (ids globally unique)
  mq_columns: created_at, id, type, action, repo, actor, number, title, detail, + type-specific trailing columns (body always last when present)

# Event Coverage

events_api:
  source: GitHub Events API (~17 types), actions are a subset of the webhook set
  covered: IssuesEvent (opened/closed/reopened/assigned/labeled...), IssueCommentEvent (created), PullRequestEvent (opened/closed/merged...), PullRequestReviewEvent (state: approved/changes_requested/commented), PullRequestReviewCommentEvent, PushEvent (no commit list), ReleaseEvent (published), ForkEvent, WatchEvent, CreateEvent/DeleteEvent, CommitCommentEvent, DiscussionEvent, GollumEvent, MemberEvent, PublicEvent, SponsorshipEvent
  invisible: comment edited/deleted, check_run/check_suite/status (CI), workflow_run and other webhook-only types

# Lifecycle Semantics

lifecycle:
  - Fresh start = blind start: a new listener sets its cursor at the newest event and does not backfill history — ghclaw only sees events while it runs (offline = blind, like a webhook)
  - The MQ never shrinks; the built-in tailer resumes from its own index, and handlers still carry idempotency marks (footers, eyes reaction) since the MQ may be re-read by any tool
  - Each poll reads only the first page (100 events); a bigger burst loses its tail, surfaced as a cursor-gap warning
  - Stop keeps data unless --purge; a stopped listener is blind (window events roll past)
  - Polls align to interval boundaries to be kind to the rate limit
  - Gap detection: cursor missing from the window → batch treated as new + warning logged

# Use Cases

use_cases:
  - AI-assisted GitHub issue triage + labeling without running a server or configuring webhooks
  - Auto-reply to issues/PR comments that mention @x (or a custom keyword)
  - Watching repos you do not admin (polling needs read access only)
  - One MQ file feeding multiple consumers (built-in AI, your own scripts, awk/python cursors)
  - Security- and community-sensitive feeds: MemberEvent (collaborator changes), PublicEvent (repo went public), WatchEvent/ForkEvent

# Limits

limits:
  - Poll latency (default 60s), not sub-second; busy orgs may need 10-30s intervals
  - Events API can drop events under load; window is ~300 events / 30 days
  - Push events lag 10-40 minutes through GitHub's feed pipeline
  - Org watching is public-events-only; --user misses other people's activity in that user's repos
  - Not for CI-driven automation (no check_run/status in the Events API)
  - Rate budget: 5000 requests/hour authenticated; ~60/hour per target at 60s interval; MQ/JSON grow unbounded — housekeeping recipes in 1-ghclaw-mq-custom-consumer
  - Notifications-based @mention channel is deferred until demand shows up
  - Troubleshooting quick table in the landing page maps every common log line (MQ not found, cursor gap, missing token/provider, keyword non-triggers) to its fix

# Related Resources

official:
  source: https://github.com/x-bash/ghclaw
  module: https://github.com/x-cmd/x-cmd (mod/ghclaw/)
related:
  - name: GitHub Events API
    url: https://docs.github.com/en/rest/activity/events
  - name: x ai
    url: https://x-cmd.com/mod/ai
  - name: x env (gh)
    url: https://x-cmd.com/mod/env

# Summary

ghclaw is a "claw design" for GitHub events shipped as the x-cmd module `x ghclaw`: a background listener polls the GitHub Events API for a repo / org / user target, dedups by event-id cursor behind ETag/304 conditional requests, and appends each new event to an on-disk append-only MQ (one TSV line + one JSON file per event). A foreground consumer dispatches MQ lines to built-in AI agents — x ai triage classifies and labels new issues (only from the repo's existing label vocabulary), and x ai reply answers comments containing the @x keyword — posting results back through gh api with footer and reaction idempotency guards. No webhook, no server, no admin rights needed; ghclaw ships with x-cmd. External tools can consume the same MQ with their own cursor, or replace the dispatcher wholesale via ___X_CMD_GHCLAW_TAILER_HANDLE. Limits come from the Events API itself: polling latency, possible event loss, no CI events, invisible comment edits/deletes.
