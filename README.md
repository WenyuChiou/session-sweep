# Session Sweep — Claude Code Plugin

Automatically scan your system, find all session artifacts left behind by Claude Code (stale git worktrees, dangling refs, orphaned directories), safely remove them, and report how much disk space you reclaimed.

## Why You Need This

Every Claude Code task creates a **git worktree** — a full working copy of your repository under `~/.claude/worktrees/`. These do not get cleaned up automatically. After a few days of heavy use:

- 80 tasks × 500 MB repo = **40 GB wasted**
- Disk fills up silently
- System slows down

This plugin gives you a single command to find and purge all stale session artifacts.

## Installation

```bash
claude plugin add github:WenyuChiou/session-sweep
```

## Usage

Invoke directly:

```
/session-sweep
```

Or just describe what you want — Claude handles the rest:

```
My computer has been sluggish since I started using Claude Code heavily. Can you clean up?
Clean up worktrees in my data-pipeline repo
Set up a daily cleanup at 3 AM so I never have to think about this again
```

## Examples

### Basic: system is slow, clean everything up

You notice your MacBook fan is running constantly and Finder shows 12 GB less free space than last week.

> **You:** My computer is slow and I'm running out of disk space. Can you clean up after Claude Code?

Session Sweep scans `~/.claude/worktrees/`, finds 83 stale directories from the past week of coding sessions, checks that none are currently locked by an active session, then bulk-removes them and prunes the dangling git refs.

```
=== Session Sweep Report ===

Scanned: 4 repositories
Removed: 83 stale worktrees
Pruned:   6 dangling git refs

  data-pipeline/    41 removed   (1.3 GB freed)
  my-app/           28 removed   (870 MB freed)
  scripts/           9 removed   (95 MB freed)
  dotfiles/          5 removed   (18 MB freed)

Total space freed: 2.3 GB
Active worktrees kept: 1 (your current session)
```

---

### Targeted: clean a specific repo

You're about to push a large branch and want to free space in one specific project without touching anything else.

> **You:** Clean up stale worktrees in my data-pipeline repo only.

Session Sweep scopes the sweep to that repo's worktree container, leaving other repos untouched.

```
=== Session Sweep Report (targeted) ===

Repository: ~/projects/data-pipeline
Removed: 41 stale worktrees
Pruned:   2 dangling git refs

Total space freed: 1.3 GB
Active worktrees kept: 0 (no active sessions in this repo)
```

---

### Scheduled: set it and forget it

After your first cleanup you realize this is going to keep accumulating.

> **You:** Set up a daily cleanup at 3 AM so I never have to think about this again.

Session Sweep creates a scheduled task that runs nightly, keeping your disk clean automatically. You get a summary the next morning if anything was removed.

```
Scheduled task created: session-sweep
  Runs daily at 03:00 local time
  Next run: tomorrow at 3:00 AM

To cancel: "remove the daily session sweep"
```

## What Gets Cleaned

**Before** — after a week of heavy Claude Code use across 5 repos:

```
~/.claude/worktrees/
├── aged-maxwell/          ← stale (session ended 6 days ago)
├── bold-turing/           ← stale
├── calm-hopper/           ← stale
├── deft-lovelace/         ← ACTIVE (current session — kept)
├── eager-dijkstra/        ← stale
│   ... 79 more stale dirs
```

- **83 stale worktrees** across 4 repos
- **~1.3 GB** wasted in data-pipeline alone; **~2.3 GB** total
- Git still tracking 6 refs to directories that no longer exist

**After** — session sweep completes in ~8 seconds:

```
~/.claude/worktrees/
└── deft-lovelace/         ← your active session, untouched
```

- Only the live session remains
- All dangling git refs pruned
- 2.3 GB reclaimed

## What It Does

1. **Discovers** all `~/.claude/worktrees/` directories on your system
2. **Measures** disk usage before cleanup
3. **Checks safety** — skips active sessions (locked worktrees), never touches the main working tree, validates paths before deleting
4. **Removes** stale worktrees (bulk delete + `git worktree prune`)
5. **Prunes** dangling git refs left behind by orphaned sessions
6. **Reports** total space freed, per-repo breakdown

## Platform Support

| Platform | Status |
|----------|--------|
| macOS    | ✅ Full support |
| Linux    | ✅ Full support |
| Windows  | ✅ Full support (PowerShell + cmd fallback) |

## License

MIT — see [LICENSE](./LICENSE)
