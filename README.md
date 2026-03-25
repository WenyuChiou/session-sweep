# Worktree Cleanup — Claude Code Plugin

Automatically scan your system, find stale git worktrees left behind by Claude Code sessions, safely remove them, and report how much disk space you reclaimed.

## Why You Need This

Every Claude Code task creates a **git worktree** — a full working copy of your repository. These don't get cleaned up automatically. After a few days of heavy use:

- 80 tasks × 500 MB repo = **40 GB wasted**
- Disk fills up silently
- System slows down

This plugin gives you a single command to find and purge all the stale copies.

## Installation

```bash
claude plugin add github:wenyuchiou1234/worktree-cleanup
```

## Usage

Just describe what you want and Claude handles it:

```
/worktree-cleanup
```

Or naturally:

```
Clean up my stale worktrees
I'm running low on disk space, clean up after Claude Code
How many worktrees are taking up space?
```

## What It Does

1. **Discovers** all `.claude/worktrees/` directories on your system
2. **Measures** disk usage before cleanup
3. **Checks safety** — skips active sessions, keeps main working tree
4. **Removes** stale worktrees (bulk delete + `git worktree prune`)
5. **Reports** total space freed, per-repo breakdown

## Platform Support

| Platform | Status |
|----------|--------|
| macOS    | ✅ Full support |
| Linux    | ✅ Full support |
| Windows  | ✅ Full support (PowerShell + cmd fallback) |

## Example Output

```
=== Worktree Cleanup Report ===

Scanned: 5 repositories
Removed: 87 stale worktrees

  my-app/           42 removed   (1.8 GB freed)
  data-pipeline/    28 removed   (920 MB freed)
  frontend/         12 removed   (340 MB freed)

Total space freed: 3.1 GB
Active worktrees kept: 1 (current session)
```

## License

MIT — see [LICENSE](./LICENSE)
