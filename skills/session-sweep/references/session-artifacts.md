# Claude Code Session Artifacts

Where Claude Code leaves files on your system, what they are, and how to clean them up.

---

## Worktree Storage Locations

Claude Code creates worktrees in one location:

```
~/.claude/worktrees/<random-name>/
```

This is a **global** directory under your home folder, shared across all repositories. Each worktree subdirectory is a full git checkout of a specific repo at a specific commit, created when Claude Code began a task.

Example on macOS/Linux:
```
~/.claude/worktrees/
├── bold-turing/        ← worktree for task on repo A (stale)
├── calm-hopper/        ← worktree for task on repo B (stale)
├── deft-lovelace/      ← worktree for current active session (keep)
└── eager-dijkstra/     ← worktree for task on repo A (stale)
```

Example on Windows:
```
C:\Users\you\.claude\worktrees\
├── bold-turing\
├── calm-hopper\
└── deft-lovelace\
```

> **Note:** Some older Claude Code documentation and community posts refer to a per-repo location (`<repo>/.claude/worktrees/`). In current versions, the global `~/.claude/worktrees/` path is what Claude Code actually uses. If you find a `.claude/worktrees/` directory inside a specific repo, it may be from an older version or a custom configuration.

---

## Worktree Naming Convention

Claude Code names worktrees using random **adjective + scientist/mathematician** pairs:

```
bold-turing
calm-hopper
deft-lovelace
eager-dijkstra
fierce-curie
graceful-knuth
happy-tesla
idle-hawking
jolly-noether
keen-lalande
```

The names are generated randomly at session start and have no relationship to the task content, repo name, or branch. They are purely for uniqueness and human readability (easier to type than a UUID).

This naming scheme means you cannot tell which repo a worktree belongs to from its directory name alone. To find the repo, check the worktree's `.git` file:

```bash
cat ~/.claude/worktrees/bold-turing/.git
# gitdir: /home/user/projects/my-app/.git/worktrees/bold-turing
```

That path leads back to the main repo's tracking entry.

---

## Lock File Mechanism

An **active** Claude Code session locks its worktree to prevent accidental deletion.

**Lock file location:**
```
<main-repo>/.git/worktrees/<name>/locked
```

Note: this file lives inside the **main repository's** `.git` directory, not inside the worktree directory itself.

**How it works:**
1. When a Claude Code session starts, it runs `git worktree lock <path>` on its worktree.
2. Git creates a `locked` file at `<main-repo>/.git/worktrees/<name>/locked`.
3. The presence of this file (regardless of content) tells `git worktree prune` to skip this entry.
4. When the session ends cleanly, Claude Code runs `git worktree unlock <path>`, removing the file.

**What this means for cleanup:**
- If a session crashed or was force-killed, the lock file may remain even though no session is active. A stale lock file makes `git worktree prune` skip that entry indefinitely.
- To check: `git worktree list --porcelain` — entries with a `locked` line are locked.
- To manually clear a stale lock: `git worktree unlock <path>` or delete the `locked` file directly.

**How session-sweep handles this:** It uses `git worktree list --porcelain` to read lock status before removing anything. Locked worktrees are skipped and listed in the report as "kept (locked)".

---

## What `git worktree prune` Cleans vs. What Needs Manual Deletion

These are two different problems:

| Problem | What it is | How to fix |
|---------|-----------|------------|
| **Stale directory** | The `~/.claude/worktrees/<name>/` folder still exists on disk | Manual: `rm -rf` (macOS/Linux) or `Remove-Item -Recurse -Force` (Windows) |
| **Dangling git ref** | `<main-repo>/.git/worktrees/<name>/` entry exists but the worktree directory is gone | `git worktree prune` |

`git worktree prune` only cleans up git's **internal tracking data** — it does nothing to directories that still exist on disk. Conversely, deleting the directory without pruning leaves orphaned entries in `.git/worktrees/` that git complains about.

**Correct order for bulk cleanup:**
```bash
# 1. Delete the directories first
rm -rf ~/.claude/worktrees/bold-turing
rm -rf ~/.claude/worktrees/calm-hopper

# 2. Then prune git's stale references
cd /path/to/main-repo
git worktree prune
```

Reversing this order (prune then delete) also works, but git will emit warnings about missing directories during the prune step.

---

## Other Session Artifacts Under `~/.claude/`

Beyond worktrees, Claude Code stores additional session data:

### Conversation and session data

```
~/.claude/
├── worktrees/              ← git worktrees (covered above)
├── projects/               ← per-project persistent state
│   └── <encoded-path>/
│       ├── memory/         ← persistent memory files (user, feedback, project, reference)
│       └── *.jsonl         ← conversation logs
├── settings.json           ← global Claude Code configuration
├── settings.local.json     ← local overrides (not synced)
└── CLAUDE.md               ← global instructions loaded into every session
```

### What is safe to delete manually

| Path | Safe to delete? | Notes |
|------|----------------|-------|
| `~/.claude/worktrees/<stale-name>/` | ✅ Yes | Run `git worktree prune` after |
| `~/.claude/projects/<path>/*.jsonl` | ✅ Yes | Conversation history only; settings and memory unaffected |
| `~/.claude/projects/<path>/memory/` | ⚠️ With care | Deletes Claude's persistent memory for that project |
| `~/.claude/settings.json` | ❌ No | Recreates from scratch — loses all configuration |
| `~/.claude/CLAUDE.md` | ❌ No | Loses all global instructions |

### Scheduled task artifacts

If you have set up a scheduled session sweep, the schedule is managed through Claude Code's built-in scheduled tasks feature. Use Claude Code's task management commands to list or cancel scheduled tasks — there is no separate file path to manage manually.

---

## How Much Space Do Worktrees Use?

A worktree is a **full working copy** of the repository at a specific commit — it contains all tracked files but shares git objects with the main repo (objects are not duplicated).

Rough size estimates:

| Repo size | Per worktree | 50 worktrees |
|-----------|-------------|-------------|
| 10 MB     | ~10 MB      | ~500 MB     |
| 100 MB    | ~100 MB     | ~5 GB       |
| 500 MB    | ~500 MB     | ~25 GB      |
| 2 GB      | ~2 GB       | ~100 GB     |

Git objects (`.git/objects/`) are stored only in the main repo and shared — worktrees do not duplicate them. The space comes from the checked-out working files.
