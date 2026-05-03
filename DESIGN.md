# Unblock — design decisions

> Why we built it this way, and what we deliberately chose not to build.

---

## The problem

In open-source repos, contributors claim issues by self-assigning and then go quiet. The issue sits in a "taken" state, discouraging others from picking it up. Maintainers are left manually chasing inactive assignees — awkward, time-consuming, and entirely automatable.

This is known as **cookie licking**: marking something as yours without actually working on it. Unblock removes the manual burden by treating assignments as time-limited, conditional locks rather than permanent ownership.

---

## Why pure YAML — no backend

The biggest architectural decision was to implement Unblock entirely as a GitHub Actions workflow with no external server, no database, and no hosted service.

**GitHub is already the data layer.** Everything Unblock needs to know is stored in GitHub: who is assigned, when the issue was last active, whether a linked PR exists, what labels are applied. On every run, the action queries this live from the GitHub API. Nothing needs to persist anywhere else.

**Zero infrastructure to maintain.** A server-based approach requires hosting, uptime monitoring, authentication, and deployment pipelines — all before the action does its first useful thing. A YAML action installs in under two minutes: drop the file into `.github/workflows/` and it runs on the next cron tick.

**The tradeoffs we accepted:**
- No persistent history of assignment events across runs
- No cross-repository aggregation or dashboards
- Polling instead of real-time webhook-driven processing
- No external notifications (Slack, email) without additional steps

These are real limitations. They are acceptable because Unblock's core job — keeping issues unblocked — requires none of them. If the project grows to need them, a backend can be added as a separate layer on top without changing the YAML core.

---

## Key decisions

| Decision | Rationale |
|---|---|
| Cron as the primary trigger | Predictable daily runs are the safety net. Event-driven triggers fire immediately on assignment changes but cron catches everything. |
| Default 14 days until unassign | Two weeks gives contributors enough time for meaningful progress without leaving issues blocked indefinitely. Configurable per repo. |
| Default 3-day warning before unassign | A heads-up comment gives the assignee a chance to respond or apply an exempt label before losing the assignment. Prevents the action feeling punitive. |
| Exempt labels escape hatch | Some issues are intentionally paused — blocked on external dependencies, reserved for a specific person. Labels let maintainers protect these without disabling the whole action. |
| Friendly comment language | Contributors should feel invited to re-engage, not shamed. The goal is momentum, not punishment. |
| Allow re-assignment after unassign | If someone returns and wants to pick the issue back up, they just self-assign again. Timer resets. Fair and non-adversarial. |
| `GITHUB_TOKEN` by default | The built-in token works for most public repos with no extra setup. Maintainers only need a PAT for cross-repo use cases. |
| No backend in v1 | The infrastructure cost is high and the marginal benefit for a single-repo action is low. Build it when contributors start asking "I wish I could see..." often enough. |

---

## Decisions in the workflow file

**Both `schedule` and `issues` triggers.** The cron handles the steady-state case — scan everything once a day. The `issues` trigger fires immediately when an assignment changes, so a freshly assigned issue gets its timer reset without waiting for the next cron window. Both together give accuracy without sacrificing reliability.

**`workflow_dispatch` included.** Manual dispatch lets maintainers run the action on demand for testing and debugging, without having to temporarily edit the cron schedule.

**Narrow permissions.** The action only requests `issues: write` and `contents: read`. No pull-request permissions, no admin access. Scoping tightly reduces blast radius if the action is ever misconfigured.

**Pin to a SHA in production.** The workflow references `@v1` for readability, but mutable tags can be force-pushed. In production, replace `@v1` with the full commit SHA of the release you want to lock to. Dependabot can keep this updated automatically.

---
<img width="1292" height="1558" alt="image" src="https://github.com/user-attachments/assets/a0b43e0c-dfd8-40b3-83fc-04b8c0596a56" />

## What a backend would add (and when to build it)

The following would become possible with a backend:

- An analytics dashboard showing assignment history, median time-to-unassign, contributor trends
- Cross-repository tracking across an organisation
- Contributor reputation scoring based on historical follow-through
- Webhook-driven processing that eliminates cron polling latency
- External notifications via Slack, email, or any other channel

**When to build it:** when maintainers or contributors start asking "I wish I could see..." with enough frequency that the friction outweighs the infrastructure cost. Until then, YAML-only is strictly better — lower cost, lower maintenance, easier adoption.

---

## Design philosophy

**Claiming is not contributing.** An assignment is a promise of intent, not a claim of ownership. Unblock enforces this by making assignments time-limited and conditional on visible progress. The issue pool stays open to everyone.

**Zero maintainer overhead.** Every hour spent chasing inactive contributors is an hour not spent reviewing PRs or improving the project. Unblock eliminates this category of work entirely. It should run invisibly in the background.

**Friendly by default.** Warning comments invite re-engagement. Unassignment messages explain what happened and how to re-claim the issue. The system is a helpful reminder, not a penalty.
<img width="1292" height="1064" alt="image" src="https://github.com/user-attachments/assets/286f7547-7057-406f-9984-6869f8671d5f" />
