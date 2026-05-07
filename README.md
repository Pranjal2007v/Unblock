<div align="center">

# 🚧 unblock

**Automatic issue assignment expiry for GitHub — because claiming isn't contributing.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub Actions](https://img.shields.io/badge/powered%20by-GitHub%20Actions-2088FF?logo=github-actions&logoColor=white)](https://github.com/features/actions)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

</div>

---

## The Problem

You've seen it before. Someone gets assigned to an issue, and then… nothing. Days pass. Weeks pass. Other contributors see the assignment and move on, assuming it's being handled. The issue just sits there — claimed but untouched.

This is **cookie licking** — grabbing something just so no one else can have it, without actually doing anything with it. It quietly kills momentum in open source projects and puts an unnecessary manual burden on maintainers to chase people down.

---

## The Solution

**unblock** adds a lightweight automation layer that treats issue assignments as *temporary and conditional* — not permanent locks.

The system is built around two GitHub Actions workflows — one that responds to slash commands in issue comments, and one that runs on a daily schedule to enforce deadlines automatically. No external server, no database, no dependencies beyond the default `GITHUB_TOKEN`.

---

## How It Works

```
Anyone comments /assign-check @user 7d
│
▼
Bot checks if user has forked the upstream repo
│
├── No fork? ──▶ Rejected with a message, stops here
│
▼
Bot posts contributor bio card
(repo commits, merged PRs, issues opened, account age, followers)
│
▼
Maintainer reviews and comments /approve-assign @user
│
▼
Bot assigns issue + adds label assigned:7d + stores deadline
│
▼
Daily cron runs at 09:00 UTC
│
├── 3 days left, no update? ──▶ Warning comment posted to assignee
│
└── Deadline passed, no update? ──▶ Unassigned + label removed + issue reopened
```

---

## Features

- 🔍 **Fork verification** — contributors must have forked the upstream repo before they can be considered
- 👤 **Contributor bio card** — maintainers see repo commits, merged PRs, issues opened, account age, and public profile before approving
- 🔐 **Maintainer-gated approval** — anyone can trigger a check, but only listed maintainers can approve the actual assignment
- ⏰ **Configurable deadlines** — set any deadline from 1 to 90 days per assignment (`/assign-check @user 14d`)
- ⚠️ **3-day warning** — bot pings the assignee when they're running out of time
- 🔁 **Auto-unassign** — if the deadline passes with no activity, the issue is freed for others automatically
- 💬 **All state in comments** — deadline is stored in a hidden HTML comment, no external storage needed
- 🛠️ **Zero setup beyond a token** — uses the default `GITHUB_TOKEN`, no secrets to configure

---

## Workflows

unblock ships as two workflow files. Both go in `.github/workflows/`.

---

### 1. `assign-bot.yml` — Slash command handler + deadline enforcer

This is the main workflow. It handles two jobs:

**Job 1: `handle-command`** — fires on every new issue comment and routes slash commands.

**Job 2: `check-deadlines`** — fires on a daily cron (`0 9 * * *`, 09:00 UTC) and scans all open issues with an `assigned:Xd` label.

```yaml
name: Assign Check Bot

on:
  issue_comment:
    types: [created]
  schedule:
    - cron: '0 9 * * *'
```

#### Configuration

Open the workflow file and update the `MAINTAINERS` array and `UPSTREAM` constant at the top of the script:

```js
const MAINTAINERS = ['your-username', 'another-maintainer'];
const UPSTREAM    = 'org/repo';  // the canonical upstream repo to check forks against
```

No other configuration is needed. The `GITHUB_TOKEN` is provided automatically by GitHub Actions.

#### Commands

| Command | Who can use | Description |
|---|---|---|
| `/assign-check @user Nd` | Anyone | Checks if user has forked the upstream, then posts their contributor bio card. `N` is the requested deadline in days (1–90). |
| `/approve-assign @user` | Maintainers only | Assigns the issue, creates an `assigned:Nd` label, and starts the deadline clock. Reads the deadline from the most recent `/assign-check` for that user. |

#### Fork verification

The bot checks forks in two passes:

1. **Fast path** — tries `GET /repos/{user}/{upstream-repo-name}` directly and checks if `parent.full_name` matches the upstream. One API call.
2. **Fallback** — if the user renamed their fork, lists all their forked repos and checks each parent. Handles any fork name.

#### Bio card

When the fork check passes, the bot posts a table like this:

```
## 🤖 Contributor Bio — @username

| Field                        | Value     |
|------------------------------|-----------|
| 🍴 Forked repo               | ✅ Yes    |
| 📅 Account created           | 11mo ago  |
| 👥 Followers                 | 27        |
| 📦 Public repos              | 57        |
| 🔨 Commits in this repo      | 4         |
| ✅ PRs merged in this repo   | 1         |
| 📝 Issues opened in this repo| 7         |

Requested deadline: 7 days

A maintainer can approve with:
/approve-assign @username
```

#### Deadline tracking

Once approved, the bot posts a comment containing a hidden machine-readable tag:

```html
<!-- ASSIGN_BOT: assignee=username deadline=2025-06-01 days=7 -->
```

The daily cron reads this tag to know who is assigned and when their deadline is. No external storage or database is used.

Activity is defined as: **any comment posted by the assignee after the assignment comment**. If the assignee posts at least one comment, the deadline is considered met and the cron skips that issue.

---

### 2. `deepwiki-bot.yml` — AI issue answering bot

This is a companion workflow that automatically answers new issues and quote-reply follow-ups using DeepWiki (repo documentation context) + Groq (LLM).

```yaml
name: DeepWiki Issue Bot

on:
  issues:
    types: [opened]
  issue_comment:
    types: [created]
```

#### How it works

1. When a new issue is opened, the bot fetches relevant documentation from [DeepWiki](https://deepwiki.com) for the configured repo, then generates a markdown answer using Groq's `llama-3.3-70b-versatile` model.
2. For follow-up questions, users quote-reply the bot's comment (comments starting with `>`). The bot strips the quoted lines, fetches fresh DeepWiki context, and replies with the full conversation history for continuity.
3. Bot-to-bot loops are prevented — the workflow ignores comments made by `github-actions[bot]`.
4. Issue template boilerplate (environment fields, checkboxes, N/A lines) is stripped before sending to the LLM so it gets a clean question.

#### Configuration

| Variable | Where | Description |
|---|---|---|
| `OPENAI_API_KEY` | Repository secret | Your [Groq API key](https://console.groq.com) (uses OpenAI-compatible endpoint) |
| `REPO_NAME` | Workflow env | The repo DeepWiki should search, e.g. `apache/incubator-hugegraph` |

```yaml
env:
  OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
  REPO_NAME: apache/incubator-hugegraph
```

#### Secrets required

| Secret | Required by |
|---|---|
| `GITHUB_TOKEN` | Both workflows (auto-provided by GitHub, no setup needed) |
| `OPENAI_API_KEY` | DeepWiki bot only — add in repo Settings → Secrets → Actions |

---

## Getting Started

1. Copy both workflow files into `.github/workflows/` in your repo:
   - `assign-bot.yml`
   - `deepwiki-bot.yml` *(optional — only if you want AI issue answering)*

2. In `assign-bot.yml`, update the config block:
```js
const MAINTAINERS = ['your-github-username'];
const UPSTREAM    = 'org/repo-name';
```

3. If using the DeepWiki bot, add your Groq API key as a repository secret named `OPENAI_API_KEY`.

4. Push to your default branch. Both workflows activate immediately — no further setup.

---

## Example Comments

**Fork check failed:**
> 🤖 **@username hasn't forked this repo.**
>
> Not gonna let you work on it buddy — fork the repo first, then someone can run `/assign-check` for you again.

**Deadline warning (3 days left):**
> ⚠️ **Deadline reminder — @username**
>
> You have **3 days left** (deadline: 2025-06-01) to post an update on this issue.
> If you need more time or can't continue, please let the maintainers know so someone else can pick this up.

**Auto-unassign:**
> ⏰ **Time's up — @username has been unassigned.**
>
> The 7-day deadline passed with no update on this issue. This issue is now open for someone else to pick up.
> @username — if you'd still like to work on this, ask a maintainer to re-assign you.

---

## Philosophy

unblock isn't about punishing contributors. Life happens — people get busy, scope changes, PRs stall. The goal is simply to **keep work visible and moving** so that open source projects don't quietly grind to a halt because of stale assignments.

The maintainer-gated approval step exists for a reason: a bio card gives maintainers just enough signal to make a quick, informed decision — without turning issue assignment into a gatekeeping exercise.

Fair for contributors. Easier for maintainers. Better for everyone.

---

## Design Doc

Read the design doc here: [Design Doc](./DESIGN.md)

---

## Contributing

Contributions are welcome! Please check [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## License

MIT © [Pranjal2007v](https://github.com/Pranjal2007v)
[bitflicker64](https://github.com/bitflicker64)
[ishitasharma2490](https://github.com/ishitasharma2490)
