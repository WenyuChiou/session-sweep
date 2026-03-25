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

> **Cloud sync check:** Before scanning or deleting, check whether `~/.claude/worktrees/` (or any parent directory) is inside a cloud sync folder (Google Drive, OneDrive, Dropbox, iCloud). Deleting files in sync folders can trigger slow cascading syncs or conflicts. Detect by checking for known sync folder names in the path:
> ```bash
> echo "$HOME" | grep -Ei "google.drive|onedrive|dropbox|icloud"
> ```
> If detected, warn the user and skip any worktrees inside sync folders.

**Windows (PowerShell):**
```powershell
Get-ChildItem -Path $env:USERPROFILE -Recurse -Depth 6 -Directory -Filter "worktrees" -ErrorAction SilentlyContinue |
  Where-Object { $_.FullName -match '\\\.claude\\worktrees$' }
```

The depth limit keeps the scan fast (usually under 5 seconds). Each result is a container directory holding multiple `<random-name>/` subdirs, one per past session.

### 1b. Identify Parent Repo for Each Worktree

Each worktree directory contains a `.git` file (not a directory) that points back to the main repository. Read this file to determine which repo owns each worktree, then group worktrees by parent repo.

```bash
# For each worktree dir found, read its .git pointer to identify the parent repo
cat ~/.claude/worktrees/<name>/.git
# Output: gitdir: /home/user/projects/my-app/.git/worktrees/<name>
# Parent repo = /home/user/projects/my-app
```

Group all worktrees by parent repo, then run `git worktree list --porcelain` **once per repo** (not once per worktree). This is much faster for systems with many worktrees.

**macOS/Linux — parse parent repo from each worktree’s .git file:**
```bash
declare -A REPO_WORKTREES  # map: repo_path → list of worktree names

for wt_dir in ~/.claude/worktrees/*/; do
  git_file="$wt_dir/.git"
  if [[ -f "$git_file" ]]; then
    # Extract: "gitdir: /path/to/repo/.git/worktrees/name" → "/path/to/repo"
    gitdir=
    repo_root=
    REPO_WORKTREES["$repo_root"]+= " $wt_dir"
  fi
done

# Now run git worktree list once per repo
for repo in "${!REPO_WORKTREES[@]}"; do
  echo "=== $repo ===" 
  git -C "$repo" worktree list --porcelain
done
```

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

**Stale lock files:** If a worktree shows as `locked` in `git worktree list --porcelain` but no Claude Code session is currently running (e.g., after a crash or force-kill), the lock is stale. Report these as “suspected stale lock” in the summary. After user confirmation, offer `git worktree unlock <path>` as a recovery option:

```bash
# Check if a locked worktree has a stale lock
git worktree list --porcelain | grep -A2 "locked"

# If confirmed stale (no active Claude session):
git worktree unlock /path/to/worktree
# Then re-run the sweep to pick it up for deletion
```

### 3.5. Confirm Before Deleting

**Never proceed to deletion without explicit user confirmation.**

After identifying all stale worktrees, present a preview and ask:

```
Found 47 stale worktrees (2.1 GB). Proceed with deletion? [y/N]
```

Include:
- Total count of stale worktrees to be removed
- Estimated disk space that will be freed (from Step 2 measurements)
- Per-repo breakdown if multiple repos are involved
- Any worktrees that will be skipped (active/locked) and why

Only proceed to Step 4 after receiving explicit confirmation (`y` or `yes`). If the user says no, abort and report what was found without deleting anything.

### 4. Remove Stale Worktrees

> **Scope:** Step 1 found container directories (e.g., `~/.claude/worktrees/`). In this step you iterate over the **individual subdirectories** within that container (e.g., `~/.claude/worktrees/bold-turing/`, `~/.claude/worktrees/calm-hopper/`). Never delete the container directory itself — only the individual `<name>/` subdirectories identified as stale in Step 3.

> **Warning:** `git worktree remove --force` silently discards any uncommitted changes in the worktree. Confirm with the user before proceeding if they may have in-progress work in a stale worktree.

**For a few worktrees** (under 10):
```bash
git worktree remove /path/to/worktree --force
```

**For many worktrees** (10+, the common case): Bulk delete via the filesystem is much faster, then let git clean up its refs.

macOS/Linux — **always validate the path contains `.claude/worktrees/` before deleting**:
```bash
WORKTREE_PATH="/path/to/.claude/worktrees/stale-name"

# Normalize the path to prevent traversal attacks (e.g., /.claude/worktrees/../../../etc)
SAFE_PATH=$(realpath --canonicalize-missing "$WORKTREE_PATH")

# Safety check: only delete paths that are confirmed worktree containers
if [[ "$SAFE_PATH" == *"/.claude/worktrees/"* ]]; then
  rm -rf "$SAFE_PATH"
else
  echo "SKIP: path does not look like a Claude worktree: $SAFE_PATH (resolved from $WORKTREE_PATH)"
fi

# After removing all stale dirs, clean up git's internal refs
cd /path/to/parent-repo
git worktree prune --expire=now
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
git worktree prune --expire=now
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


## Prevent Future Accumulation

After cleanup, suggest setting up automatic daily maintenance:

> Tip: You can create a scheduled task in Claude to run session-sweep daily at a quiet time (e.g., 3 AM). This prevents buildup and keeps your disk free without any manual effort. Just say "schedule a daily session sweep."

## References

For deeper context on the underlying mechanisms, see:

- `references/git-worktree-reference.md` — Git worktree command reference and internals: all `git worktree` subcommands, flags, the `.git/worktrees/<name>/` directory structure, and common error messages
- `references/session-artifacts.md` — How Claude Code creates and manages session files: worktree storage locations, the adjective-scientist naming convention, lock file mechanics, what `git worktree prune` cleans vs. what needs manual deletion, and other artifacts under `~/.claude/`
