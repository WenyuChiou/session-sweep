---
name: worktree-cleanup
description: "Automatically discover and clean up stale git worktrees across your entire system to reclaim disk space. Use this skill whenever the user mentions slow performance, low disk space, too many worktrees, cleanup, maintenance, freeing memory, or wants to reduce clutter from Claude Code sessions. Also triggers for 'my computer is slow', 'running out of space', 'clean up after Claude', or any mention of worktree accumulation. This skill is especially relevant for heavy Claude Code users who spawn many code tasks — each task creates a full git worktree copy that persists after the session ends."
---

# Worktree Cleanup

## Why This Exists

Every time Claude Code starts a code task, it creates a **git worktree** — a full working copy of your repository. This is great for isolation (each task gets its own sandbox), but these copies pile up fast. After a busy day, you might have 50-100+ worktrees sitting on disk, each one a complete copy of your repo.

**Real-world impact:** A 500MB repo × 80 worktrees = **40GB** of wasted space. Users regularly report their machines slowing down after a few days of heavy Claude Code usage.

This skill scans your entire system, finds all stale worktrees, safely removes them, and tells you exactly how much space you got back.

## Step-by-Step Process

### 1. Find All Worktrees

First, figure out what OS you're on and scan accordingly.

**macOS / Linux:**
```bash
# Claude Code stores worktrees here
find "$HOME" -maxdepth 6 -type d -name "worktrees" -path "*/.claude/worktrees" 2>/dev/null

# Standard git worktrees (less common, but check anyway)
find "$HOME" -maxdepth 5 -type d -name "worktrees" -path "*/.git/worktrees" 2>/dev/null
```

**Windows (PowerShell):**
```powershell
Get-ChildItem -Path $env:USERPROFILE -Recurse -Depth 6 -Directory -Filter "worktrees" -ErrorAction SilentlyContinue |
  Where-Object { $_.FullName -match '\\\.claude\\worktrees$' }
```

The depth limit keeps the scan fast (usually under 5 seconds). Claude Code always puts worktrees in `<repo>/.claude/worktrees/<random-name>/`.

### 2. Measure Current Disk Usage

Before deleting anything, record how much space each worktrees directory is using. This is what makes the before/after report possible — users love seeing the concrete impact.

```bash
# macOS/Linux
du -sh /path/to/.claude/worktrees/

# Windows
(Get-ChildItem -Path "C:\path\.claude\worktrees" -Recurse | Measure-Object -Property Length -Sum).Sum / 1MB
```

### 3. Check What's Safe to Remove

Not every worktree should be deleted. Here's how to tell them apart:

| Status | How to detect | Action |
|--------|--------------|--------|
| **Stale** (no active session) | No lock file in `.git/worktrees/<name>/locked` | ✅ Safe to remove |
| **Active** (session running) | Lock file exists | ❌ Skip it |
| **Main working tree** | Listed first in `git worktree list` | ❌ Never remove |
| **Broken** (orphaned reference) | Directory missing but git reference exists | ✅ `git worktree prune` handles this |

To check:
```bash
cd /path/to/parent-repo
git worktree list
# Then verify lock status for each entry
```

### 4. Remove Stale Worktrees

**For a few worktrees** (under 10):
```bash
git worktree remove /path/to/worktree --force
```

**For many worktrees** (10+, the common case): Bulk delete is much faster.

macOS/Linux:
```bash
rm -rf /path/to/.claude/worktrees/stale-name-1 /path/to/.claude/worktrees/stale-name-2
cd /path/to/parent-repo
git worktree prune
```

Windows:
```powershell
Remove-Item -Recurse -Force "C:\path\.claude\worktrees\stale-name-1"
# If Remove-Item fails (sharing violation), try:
cmd /c rmdir /s /q "C:\path\.claude\worktrees\stale-name-1"
# Then:
cd C:\path\to\parent-repo
git worktree prune
```

### 5. Report Results

After cleanup, measure disk usage again and present a clear summary:

```
=== Worktree Cleanup Report ===

Scanned: 5 repositories
Removed: 87 stale worktrees

  my-app/           42 removed   (1.8 GB freed)
  data-pipeline/    28 removed   (920 MB freed)
  frontend/         12 removed   (340 MB freed)
  scripts/           3 removed   (45 MB freed)
  dotfiles/          2 removed   (12 MB freed)

Total space freed: 3.1 GB
Active worktrees kept: 1 (current session)
```

The before/after comparison is the most valuable part of this skill. Always report both the count and the disk space.

## Troubleshooting

**Windows sharing violations:** Claude Code's host process sometimes holds file handles on worktree parent directories. The worktree contents get deleted (space is freed), but the empty folder can't be removed until the app restarts. This is normal — `git worktree prune` still cleans up the git references, and the empty shell disappears on next restart.

**HEAD.lock errors:** If `git worktree prune` fails with a lock error, another git process is active in that repo. Wait a few seconds and retry.

**Permission denied:** Some worktrees may be locked by running processes. Skip them and note it in the report — don't let one error stop the whole cleanup.

**Cloud sync folders:** Skip any worktrees found inside Google Drive, OneDrive, Dropbox, or iCloud folders. Deleting files in sync folders can cause slow cascading syncs or conflicts.

## Prevent Future Accumulation

After cleanup, suggest setting up automatic daily maintenance:

> Tip: You can create a scheduled task in Claude to run worktree cleanup daily at a quiet time (e.g., 3 AM). This prevents buildup and keeps your disk free without any manual effort. Just say "schedule a daily worktree cleanup."
