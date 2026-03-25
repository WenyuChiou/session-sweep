---
name: session-sweep
description: "Sweep up ALL session artifacts left behind by Claude Code: stale git worktrees, dangling git refs, and orphaned worktree directories. Use this skill whenever the user mentions slow performance, low disk space, too many worktrees, cleanup, maintenance, freeing space, or wants to reduce clutter from Claude Code sessions. Also triggers for 'my computer is slow', 'running out of space', 'clean up after Claude', or any mention of worktree or session accumulation. Especially relevant for heavy Claude Code users who spawn many code tasks — each task creates a full git worktree copy that persists after the session ends."
---

# Session Sweep

## Why This Exists

Every time Claude Code starts a code task, it creates a **git worktree** — a full working copy of your repository stored under `~/.claude/worktrees/<random-name>/` (your home directory, not inside any repo). This is great for isolation (each task gets its own sandbox), but these copies pile up fast. After a busy day, you might have 50–100+ worktrees sitting on disk, each one a complete copy of your repo.

**Real-world impact:** A 500 MB repo × 80 worktrees = **40 GB** of wasted space. Users regularly report their machines slowing down after a few days of heavy Claude Code usage.

This skill scans your system, finds all stale worktrees and dangling git refs, safely removes them, and tells you exactly how much space you got back.

## What Gets Swept

| Artifact | Where it lives | How removed |
|----------|---------------|-------------|
| Stale worktree directories | `~/.claude/worktrees/<name>/` | `rm -rf` + `git worktree prune` |
| Dangling git refs | `<repo>/.git/worktrees/<name>/` | `git worktree prune` |
| Orphaned gitdir pointers | Tracked by git internals | `git worktree prune` |

## Step-by-Step Process

### 1. Find All Worktree Container Directories

First, detect the OS and scan for Claude Code's worktree storage location. Claude Code always stores worktrees under `~/.claude/worktrees/` (your home directory, not inside any repo).

**macOS / Linux:**
```bash
# Find Claude Code worktree containers
find "$HOME" -maxdepth 6 -type d -path "*/.claude/worktrees" 2>/dev/null
```

**Windows (PowerShell):**
```powershell
Get-ChildItem -Path $env:USERPROFILE -Recurse -Depth 6 -Directory -Filter "worktrees" -ErrorAction SilentlyContinue |
  Where-Object { $_.FullName -match '\\\.claude\\worktrees$' }
```

The depth limit keeps the scan fast (usually under 5 seconds). Each result is a container directory holding multiple `<random-name>/` subdirs, one per past session.

### 2. Measure Current Disk Usage

Before deleting anything, record how much space each worktrees container is using. This makes the before/after report possible.

```bash
# macOS/Linux
du -sh ~/.claude/worktrees/

# Windows
(Get-ChildItem -Path "$env:USERPROFILE\.claude\worktrees" -Recurse -ErrorAction SilentlyContinue |
  Measure-Object -Property Length -Sum).Sum / 1MB
```

### 3. Identify Stale vs. Active Worktrees

Use `git worktree list --porcelain` from the main repo to get machine-readable status including lock state. This is the most reliable method — it reads git's internal state rather than requiring manual file inspection.

```bash
cd /path/to/parent-repo
git worktree list --porcelain
```

Output format:
```
worktree /path/to/main
HEAD abc123
branch refs/heads/main

worktree /path/to/stale-session
HEAD def456
branch refs/heads/claude/some-task
locked  # ← this line only appears if locked

worktree /path/to/orphaned
HEAD 789abc
prunable gitdir file points to non-existent location  # ← safe to prune
```

| Status | How to detect | Action |
|--------|--------------|--------|
| **Stale** (no active session) | No `locked` line in `--porcelain` output | ✅ Safe to remove |
| **Active** (session running) | `locked` line present | ❌ Skip — removing an active worktree corrupts the session |
| **Main working tree** | First entry in `git worktree list` | ❌ Never remove |
| **Prunable** (orphaned ref) | `prunable` line present | ✅ `git worktree prune` handles automatically |

**Note on lock files:** `git worktree lock <path>` writes a marker at `<main-repo>/.git/worktrees/<name>/locked`. Using `git worktree list --porcelain` reads this for you — there is no need to inspect the file path manually.

### 4. Remove Stale Worktrees

> **Warning:** `git worktree remove --force` silently discards any uncommitted changes in the worktree. Confirm with the user before proceeding if they may have in-progress work in a stale worktree.

**For a few worktrees** (under 10):
```bash
git worktree remove /path/to/worktree --force
```

**For many worktrees** (10+, the common case): Bulk delete via the filesystem is much faster, then let git clean up its refs.

macOS/Linux — **always validate the path contains `.claude/worktrees/` before deleting**:
```bash
WORKTREE_PATH="/path/to/.claude/worktrees/stale-name"

# Safety check: only delete paths that are confirmed worktree containers
if [[ "$WORKTREE_PATH" == *"/.claude/worktrees/"* ]]; then
  rm -rf "$WORKTREE_PATH"
else
  echo "SKIP: path does not look like a Claude worktree: $WORKTREE_PATH"
fi

# After removing all stale dirs, clean up git's internal refs
cd /path/to/parent-repo
git worktree prune
```

Windows:
```powershell
$path = "C:\Users\you\.claude\worktrees\stale-name"

# Safety check: only delete confirmed worktree paths
if ($path -match '\\\.claude\\worktrees\\') {
  Remove-Item -Recurse -Force $path
  # If Remove-Item fails (sharing violation), fall back to:
  # cmd /c rmdir /s /q $path
} else {
  Write-Host "SKIP: path does not look like a Claude worktree: $path"
}

# Clean up git refs
cd C:\path\to\parent-repo
git worktree prune
```

### 5. Report Results

After cleanup, measure disk usage again and present a clear summary:

```
=== Session Sweep Report ===

Scanned: 5 repositories
Removed: 87 stale worktrees
Pruned:   3 dangling git refs

  my-app/           42 removed   (1.8 GB freed)
  data-pipeline/    28 removed   (920 MB freed)
  frontend/         12 removed   (340 MB freed)
  scripts/           3 removed   (45 MB freed)
  dotfiles/          2 removed   (12 MB freed)

Total space freed: 3.1 GB
Active worktrees kept: 1 (current session)
```

The before/after comparison is the most valuable part. Always report both count and disk space.

## Troubleshooting

**Windows sharing violations:** Claude Code's host process sometimes holds file handles on worktree parent directories. The worktree contents get deleted (space is freed), but the empty folder cannot be removed until the app restarts. This is normal — `git worktree prune` still cleans up the git references, and the empty shell disappears on next restart.

**HEAD.lock errors:** If `git worktree prune` fails with a lock error, another git process is active in that repo. Wait a few seconds and retry.

**Permission denied:** Some worktrees may be locked by running processes. Skip them and note it in the report — do not let one error stop the whole sweep.

**Cloud sync folders:** Skip any worktrees found inside Google Drive, OneDrive, Dropbox, or iCloud folders. Deleting files in sync folders can cause slow cascading syncs or conflicts. Detect them by checking for known sync folder names in the path before deleting.

## Prevent Future Accumulation

After cleanup, suggest setting up automatic daily maintenance:

> Tip: You can create a scheduled task in Claude to run session-sweep daily at a quiet time (e.g., 3 AM). This prevents buildup and keeps your disk free without any manual effort. Just say "schedule a daily session sweep."
