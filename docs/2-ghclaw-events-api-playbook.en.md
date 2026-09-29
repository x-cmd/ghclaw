---
x-title: The GitHub Events API Playbook — What ghclaw Can and Cannot See
x-desc: >-
  ghclaw drinks from the GitHub Events API, a small read-only
  projection of the webhook universe. This playbook maps the
  coverage — which business events are visible, how to judge
  them, the delay and loss characteristics, the repo / org /
  user endpoint scopes, snapshot fallbacks for what events miss,
  and when a webhook channel is the better tool.
x-sidebar: ghclaw
x-keywords: github events api, webhook, polling, event coverage, check_run, notifications, org events
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'GitHub Events API playbook — what ghclaw can and cannot see'
      inLanguage: 'en'
      about: 'ghclaw event source coverage and limits'
---

# The GitHub Events API Playbook

Every event `ghclaw` delivers comes from one source: the GitHub
**Events API**, polled per target. That API is a small, old,
read-only projection of everything that happens on GitHub —
knowing exactly what it projects (and what it drops) tells you
which automations `ghclaw` can power and which need a different
tool.

> **Companion:** [landing page](0-ghclaw-landing.en.md) ·
> [MQ protocol](1-ghclaw-mq-custom-consumer.en.md)

## The three event worlds

GitHub has three overlapping event systems. Confusing them is
the #1 design mistake (e.g. assuming the Events API delivers
what Actions `on:` promises — it does not):

```
webhook event universe (40+ types, snake_case)
├── Actions `on:` triggers (most webhooks + 4 synthetic ones)
└── Events API types (~17, PascalCase + Event suffix,
    actions are a SUBSET of the webhook actions)
```

| World | Shape | Notes |
| --- | --- | --- |
| Webhooks | 40+ types, full action set (`created`/`edited`/`deleted`/`synchronize`/…), full payloads (`commits[]`, check runs) | The richest stream; needs a public endpoint + admin rights per repo |
| Actions `on:` | The webhook list plus 4 synthetic triggers (`schedule`, `workflow_dispatch`, `workflow_call`, `repository_dispatch`) | The synthetic four exist in NO event stream |
| Events API | ~17 types, subset actions | What `ghclaw` polls; no config needed on the watched repos |

Types that exist **only** in the Events API: `SponsorshipEvent`,
`MemberEvent` (webhooks have them, but Actions cannot trigger on
them). Types that exist **only** in webhook-land — and are
therefore invisible to `ghclaw`: `check_run`, `check_suite`,
`status`, `workflow_run`, `deployment`, `label`, `milestone`,
`merge_group`, `branch_protection_rule`, `dependabot_alert`,
and ~25 more.

## Business-event judgment matrix

How to tell that a specific business event happened, given only
the Events API (this is what `ghclaw`'s classifier encodes):

| Business event | Primary signal | Fallback / note |
| --- | --- | --- |
| Issue opened | `IssuesEvent` `opened` | issues snapshot shows a new number |
| Issue closed | `IssuesEvent` `closed` | `state_reason` says completed vs not_planned |
| Issue reopened | `IssuesEvent` `reopened` | state snapshot open←closed |
| Issue labeled / assigned | `IssuesEvent` `labeled`/`assigned` + payload | batch labeling repeats the last label per event |
| Issue comment created | `IssueCommentEvent` `created` | issue payload's `pull_request` field says issue vs PR |
| Comment **edited** | **not visible** | closest — `updated_at` > `created_at` on the comments endpoint |
| Comment **deleted** | **not visible anywhere** | only shows up as a count mismatch |
| PR opened / merged / closed | `PullRequestEvent` `opened`/`merged`/`closed` | pulls snapshot (payload itself is shallow) |
| PR review submitted | `PullRequestReviewEvent` — verdict in `review.state`: approved / changes_requested / commented | actions are created/updated/dismissed (not webhook's submitted) |
| PR diff comment | `PullRequestReviewCommentEvent` `created` | `path:` in the detail column |
| PR ready for review | **not in events** | pulls snapshot `draft` true→false |
| Push | `PushEvent` (no action; ref/before/after) | commit list needs a compare call |
| Release published | `ReleaseEvent` `published` | `prerelease` flag in payload |
| Star / fork | `WatchEvent` `started` / `ForkEvent` `forked` | — |
| Branch / tag created-deleted | `CreateEvent` / `DeleteEvent` | `ref:` + `type:branch\|tag` |
| Wiki page changed | `GollumEvent` | pages deleted are NOT reported |
| Collaborator added | `MemberEvent` `added` | fires when the invitation is ACCEPTED |
| Repo went public | `PublicEvent` | security-sensitive |
| New discussion | `DiscussionEvent` `created` | discussion COMMENTS have no event at all |
| **Someone @-mentioned me** | **not as a signal** — but in a watched repo the comment event IS visible; keyword-match `@you` in the body (last MQ column) | the mention *notification* exists only in the Notifications API (`reason=mention`) |
| **Review requested from me** | **not in events** | Notifications API `reason=review_requested` |
| **CI result** | **not in events** | Notifications API `reason=ci_activity` |

## Delay and loss characteristics

Measured behavior, worth internalizing before you pick
triggers:

| Aspect | Reality |
| --- | --- |
| API / web-UI actions | seconds to ~1 min — good enough for interactive auto-reply |
| Git pushes (`PushEvent`) | **10–40 min delay** through GitHub's feed pipeline; occasional loss |
| Event loss | the stream is explicitly not guaranteed under load; ~300 events / 30 days window per timeline |
| Per-poll page | only the first page (100 events) is read — a bigger burst loses its tail, logged as a cursor-gap warning |
| Audit-grade? | **No.** Treat the stream as "eventually informative", not as a ledger |

Snapshot endpoints (`/issues`, `/pulls`, `/issues/{n}/comments`,
`/pulls/{n}/comments`, `/releases`) are the designed fallback:
ETag to detect change, then diff against your last snapshot —
state transitions, comment-count deltas and timestamp
heuristics (`updated_at` > `created_at`) reconstruct much of
what events miss. They cost one request per endpoint, which is
why `ghclaw`'s main line stays events-only and leaves
reconciliation to custom consumers and the roadmap.

## Endpoint scopes — repo, org, user

Three timelines, three semantics (`ghclaw` exposes them as
`--repo`, `--org`, `--user`):

| Endpoint | Answers | Caveats |
| --- | --- | --- |
| `/repos/{o}/{r}/events` | "What happened in THIS repo" | Public or private, as your token can read |
| `/orgs/{org}/events` | "What happened across the org's repos" | **Public events only** — the name literally says list-public-organization-events; private repos stay per-repo. One shared 300-event window for all repos: a loud repo can evict a quiet repo's events (shorten the interval to 10–30s on busy orgs) |
| `/users/{name}/events` | "What did this PERSON do" | Actor-centric: only events the user triggered, anywhere on GitHub. Everyone else's activity in the user's repos is invisible — it cannot stand in for per-repo watching. **Watching yourself does not surface mentions of you** (the commenter is the actor, not you) — watch the repo and match the body instead |

Org-level 1-request vs N per-repo listeners:

| Dimension | Org 1-request | N repo listeners |
| --- | --- | --- |
| Requests per cycle | 1 | N |
| Timeline | one, cross-repo order | N independent |
| 300-window | shared (loud evicts quiet) | per-repo |
| Gap blast radius | one gap loses all repos' window | one repo only |
| New public repo | covered automatically | add a listener |
| Private repos | not visible | visible per repo |
| Interval | one-size-fits-all | per repo |

There is no bulk endpoint for a **personal account's** repos —
watching "everything under user X" means one listener per repo.

## Polling vs webhook — the honest tradeoff

| Dimension | Polling (what ghclaw does) | Webhook |
| --- | --- | --- |
| Setup | none — read access suffices | public HTTPS endpoint + per-repo admin (or org webhook / GitHub App) |
| Works on repos you don't own | yes | no |
| Latency | poll interval (seconds–minutes) | usually <1s–seconds, no SLA |
| Delivery | lossy but *eventually resumable* (ETag/cursor; gaps are detected and warned) | at-most-once, no ordering, 10s ack window — endpoint down = events gone |
| Offline behavior | listener stopped = blind from "now" (fresh starts don't backfill) | endpoint down = permanently lost |
| Event types | ~17, subset actions | 40+, full actions |
| Rate limit cost | 1 conditional request per cycle (free when 304) | none for delivery |
| Attack surface | none (outbound only) | signature verification, replay protection, secret rotation |
| Audit trail | self-logged MQ + JSON files | GitHub's Recent Deliveries panel |

The complementary design — and `ghclaw`'s documented roadmap —
is a **hybrid**: webhooks (behind a tiny receiver such as a
Cloudflare Worker that verifies `X-Hub-Signature-256`, persists
to disk and ACKs within the window) feed the SAME MQ format for
repos you manage, while polling covers everything else and acts
as the reconciliation layer for webhook loss. Webhook "real-time
but fragile" and polling "slow but sturdy" patch each other's
weaknesses.

## Security-flavored events worth watching

Two Events-API types are pure security signals and map cleanly
to alert-style consumers (keyword: `PublicEvent`, `MemberEvent`
— both invisible to Actions `on:`):

- `PublicEvent` — a private repo just became public. Alert
  immediately; this is rarely intentional.
- `MemberEvent` — a collaborator was added (fired on
  invitation acceptance). Audit "who joined which repo".
- `GollumEvent` — wiki edits on sensitive pages (deployment
  docs etc.); note it cannot see page deletions.

## What ghclaw does with all this

- **Main line**: poll the events timeline only — one conditional
  request per cycle, cursor-deduped, events appended to the MQ.
- **Classified and dispatched**: `IssuesEvent/opened` → AI
  triage + `@x` reply; `IssueCommentEvent/created` → `@x` reply.
- **Everything else**: logged and left in the MQ for your own
  consumers — which is where the [MQ protocol](1-ghclaw-mq-custom-consumer.en.md)
  comes in.

## Source & Resources

- **Source & design notes:** <https://github.com/x-bash/ghclaw>
  (`.x-cmd/story/` holds the API survey and polling/webhook
  tradeoff studies this article distills)
- **GitHub Events API:** <https://docs.github.com/en/rest/activity/events>
- **Webhook events & payloads:** <https://docs.github.com/en/webhooks/webhook-events-and-payloads>
- **Notifications API:** <https://docs.github.com/en/rest/activity/notifications>
