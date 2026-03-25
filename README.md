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

Or naturally:

```
Clean up my stale worktrees
I'm running low on disk space, clean up after Claude Code
How many worktrees are taking up space?
Sweep up my session artifacts
```

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

## Example Output

```
=== Session Sweep Report ===

Scanned: 5 repositories
Removed: 87 stale worktrees
Pruned:   3 dangling git refs

  my-app/           42 removed   (1.8 GB freed)
  data-pipeline/    28 removed   (920 MB freed)
  frontend/         12 removed   (340 MB freed)

Total space freed: 3.1 GB
Active worktrees kept: 1 (current session)
```

## License

MIT — see [LICENSE](./LICENSE)
