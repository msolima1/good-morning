# good-morning

A Claude Code skill that generates a comprehensive daily work briefing by pulling together every tool you actually work in.

## Problem

Starting the day means checking six different tools -- calendar, email, issue tracker, chat, Drive, and a code-review tool -- before you know what actually needs attention. Each one shows its own slice; none of them tell you what to do first.

## What it does

Pulls from all your work sources in parallel and surfaces what needs attention today, organized by priority rather than by source.

Sources scanned:
- **Calendar** -- meetings today, how soon they start, what's already passed
- **Email** -- unread items that need action
- **Issue tracker** -- tickets assigned, blocked, or due, ranked by a priority score
- **Chat** -- @mentions and action items assigned to you, pulled from a notification feed plus a scan of your priority spaces
- **Drive** -- docs shared with you, comments pending
- **Code review tool** -- open merge/pull requests, pipeline status, unresolved discussions, ranked by a priority score

## How it works (agent flow)

```
Calendar / Email / Issue tracker / Chat / Drive / Code review
   -> fetched in parallel via CLI wrappers
   -> each ticket/MR scored (priority field, age, staleness, blocked/review/overdue bonuses)
   -> chat notifications filtered to actionable patterns (assigned, mentioned, replied-to)
     and resolved to direct document/thread links
   -> everything merged into one briefing, grouped by status and sorted by score
   -> output: a scannable Markdown briefing + optional Google Sheet log with checkboxes
```

### Priority scoring

Both tickets and merge requests get a numeric score so the "what should I do first" ranking is reproducible rather than vibes-based -- e.g. a blocked ticket gets a flat bonus, age and staleness add points per day up to a cap, and an overdue item or failing pipeline dominates the score. See `SKILL.md` for the exact point table; it's meant to be tuned per team.

## Guardrails

- **Read-only against every source.** Nothing is marked read, resolved, or merged automatically -- the skill only reads and summarizes.
- **Parallel fetches, single pass.** All sources are queried once per run; there's no polling loop or background process to misfire.
- **Chat scan is scoped.** @mention scanning is limited to a configured list of priority spaces (not every space you've ever joined) to keep the run fast and the noise down.
- **Sheet export avoids duplicate tabs.** Before creating a new day's sheet tab, it searches for an existing one first rather than blindly appending.

## Configure

- CLI binary paths for your issue tracker, code-review, and Google Workspace wrappers are resolved by OS/architecture at the top of `SKILL.md` -- point them at whatever CLIs you actually have.
- The list of "priority spaces" to scan for chat @mentions is meant to be edited to your own team's spaces.
- The optional Google Sheet export writes to a sheet you name (default example: "Daily Brief") and reads your OAuth token from a local credentials file -- replace the token-loading snippet with however your Workspace CLI stores credentials.

## Usage

```
/good-morning
```
