---
name: 2-ghclaw-events-api-playbook
description: Everything ghclaw can and cannot see via the GitHub Events API — the ~17 event types are a small read-only projection of the 40+ webhook universe with a subset of actions. Business-event judgment matrix (comment edited/deleted invisible, no check_run/check_suite/status/CI, no @mention/review-request), delay and loss characteristics (push 10-40 min, ~300 events/30 days window, 100 per poll), repo/org/user endpoint scopes (org = public events only, user = actor-centric, no bulk endpoint for personal accounts), snapshot endpoints as fallback, polling vs webhook tradeoff and the hybrid design, and the security-relevant PublicEvent/MemberEvent/GollumEvent signals.
type: summary
---

# Core Content

core_features:
  - Events API = ~17 PascalCase types, actions are a strict subset of the webhook set; Actions `on:` additionally has 4 synthetic triggers (schedule, workflow_dispatch, workflow_call, repository_dispatch) that exist in no stream
  - Visible-only-in-Events-API: SponsorshipEvent, MemberEvent (webhooks have them but Actions cannot trigger on them)
  - Invisible to ghclaw (webhook-only): check_run, check_suite, status, workflow_run, deployment, label, milestone, merge_group, branch_protection_rule, dependabot_alert, ~25 more
  - Comment edited is invisible (timestamp heuristic at best); comment deleted is invisible everywhere; @mention / review request / CI result live only in the Notifications API (reason=mention / review_requested / ci_activity) — and watching your own user stream does not catch mentions (actor-centric); in a watched repo, @me can be approximated by keyword-matching the comment body (last MQ column)
  - Delay gradient: API/web actions seconds~1min; git pushes 10-40 min through the feed pipeline; stream lossy under load, ~300 events / 30 days window, only the first 100-event page per poll — not audit-grade

# Endpoint Scopes

scopes:
  - /repos/{o}/{r}/events — one repo, public or private as the token permits
  - /orgs/{org}/events — whole org, PUBLIC events only, one shared 300-event window (a loud repo evicts a quiet repo's events); private repos stay per-repo
  - /users/{name}/events — actor-centric personal stream (what the person did, anywhere); misses everyone else's activity in their repos; no bulk endpoint exists for a personal account's repos

# Fallbacks

fallbacks:
  - Snapshot endpoints (/issues, /pulls, issue-comments, pr-comments, /releases) + ETag diff reconstruct state transitions, comment-count deltas and edit heuristics — the designed fallback when events lose data
  - ghclaw main line stays events-only (1 request per cycle); reconciliation belongs to custom consumers / roadmap

# Polling vs Webhook

tradeoff:
  - polling: zero setup, works on repos you don't own, lossy but resumable, ~17 types, no attack surface
  - webhook: real-time, 40+ types full actions, but needs a public endpoint + per-repo admin, at-most-once with a 10s ack window, offline = permanent loss
  - hybrid (ghclaw roadmap): a small receiver (e.g. Cloudflare Worker verifying X-Hub-Signature-256, persist-then-ack) feeds the same MQ for managed repos; polling covers the rest and reconciles webhook loss

# Security Signals

security_events:
  - PublicEvent — private repo went public; alert immediately
  - MemberEvent — collaborator added (fires on invitation acceptance); audit who joined which repo
  - GollumEvent — wiki edits on sensitive pages; cannot see page deletions

# Related Resources

official:
  source: https://github.com/x-bash/ghclaw
  design_notes: .x-cmd/story/
related:
  - name: GitHub Events API
    url: https://docs.github.com/en/rest/activity/events
  - name: Webhook events and payloads
    url: https://docs.github.com/en/webhooks/webhook-events-and-payloads
  - name: Notifications API
    url: https://docs.github.com/en/rest/activity/notifications

# Summary

ghclaw drinks from the GitHub Events API, a small old read-only projection of the webhook universe: ~17 types with a subset of actions. The judgment matrix says what is visible (issue/PR/comment/push/release/star/fork lifecycles), what is partially visible (comment edits via timestamps; PR ready-for-review via the pulls snapshot) and what is invisible (comment deletes, CI results, @mentions, review requests — the last three only exist in the Notifications API). Reality check before designing triggers: pushes lag 10-40 minutes, streams lose events under load within a ~300-event / 30-day window, and each poll reads only the first 100 events. Endpoint scopes differ fundamentally: org watching covers public events only, user watching is actor-centric, and personal accounts have no bulk endpoint. Webhooks beat polling on latency and coverage but need a public endpoint plus per-repo admin and lose data when offline — the documented endgame is a hybrid where webhook receivers write the same MQ and polling reconciles. Security-flavored types worth alerting on: PublicEvent, MemberEvent, GollumEvent.
