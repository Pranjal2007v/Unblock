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

Once someone is assigned to an issue, the system starts a quiet countdown. If no meaningful activity is detected within a configurable window, the assignment is automatically removed and the issue is freed up for other contributors.

### What counts as "meaningful activity"?
- Opening a pull request that references the issue
- Pushing commits linked to the issue
- Any other activity signal you configure

No activity? The issue is unassigned automatically, a comment is posted explaining why, and it's back in the pool — no maintainer intervention needed.

---

## How It Works
Issue gets assigned
│
▼
Timer starts ⏱️
│
▼
Activity detected? ──── YES ──▶ All good, timer resets
│
NO
│
▼
Grace period check
│
▼
Still nothing? ──▶ Auto-unassign + comment posted
│
▼
Issue reopened for others 🎉

---

## Features

- ⏰ **Configurable deadlines** — set your own timeout window per repo or issue label
- 🔍 **Smart activity detection** — tracks PRs, commits, and other signals, not just comments
- 💬 **Friendly notifications** — posts a clear comment before and after unassigning
- 🔁 **No punishment, just flow** — contributors can re-assign themselves if they come back
- 🛠️ **Zero maintainer overhead** — runs entirely via GitHub Actions, no server needed

---

## Getting Started

### 1. Add the workflow file

Create `.github/workflows/unblock.yml` in your repository:

```yaml
name: unblock

on:
  schedule:
    - cron: '0 * * * *' # runs every hour
  issues:
    types: [assigned]

jobs:
  unblock:
    runs-on: ubuntu-latest
    steps:
      - uses: your-username/unblock@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          days-until-unassign: 7
          warning-before-days: 2
```

### 2. Configure (optional)

| Option | Default | Description |
|---|---|---|
| `days-until-unassign` | `7` | Days of inactivity before unassigning |
| `warning-before-days` | `2` | Days before expiry to post a warning comment |
| `exempt-labels` | `''` | Comma-separated labels that skip the timer |
| `assigned-label` | `''` | Label to add when an issue is assigned |

---

## Example Comment

When unblock removes an assignment, it posts something like this:

> 👋 Hey @contributor! This issue has been assigned for 7 days without any linked activity (PR or commit). To keep things moving, the assignment has been released so others can pick it up.
>
> No worries — if you're still working on it, feel free to re-assign yourself or leave a comment!

---

## Philosophy

unblock isn't about punishing contributors. Life happens — people get busy, scope changes, PRs stall. The goal is simply to **keep work visible and moving** so that open source projects don't quietly grind to a halt because of stale assignments.

Fair for contributors. Easier for maintainers. Better for everyone.

---

## Contributing

Contributions are welcome! Please check [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## License

MIT © [Pranjal2007v](https://github.com/Pranjal2007v)
[bitflicker64(https://github.com/bitflicker64)]
[ishitasharma2490(https://github.com/ishitasharma2490)]
