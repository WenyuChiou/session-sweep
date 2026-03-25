# Git Worktree Command Reference

## Core Commands

### `git worktree list`

Lists all worktrees linked to the current repository.

```
$ git worktree list
/home/user/projects/my-app          abc1234 [main]
/home/user/.claude/worktrees/bold-turing  def5678 [claude/fix-auth]
/home/user/.claude/worktrees/calm-hopper  ghi9012 (detached HEAD)
```

Columns: absolute path, HEAD commit (short SHA), branch name (or `detached HEAD`).

The **first entry is always the main working tree** — never remove it.

---

### `git worktree list --porcelain`

Machine-readable version. One attribute per line, entries separated by blank lines.

```
worktree /home/user/projects/my-app
HEAD abc1234567890abcdef1234567890abcdef123456
branch refs/heads/main

worktree /home/user/.claude/worktrees/bold-turing
HEAD def5678901234567890abcdef1234567890abcdef
branch refs/heads/claude/fix-auth
locked reason for lock goes here

worktree /home/user/.claude/worktrees/calm-hopper
HEAD ghi9012345678901234567890abcdef1234567890
detached
prunable gitdir file points to non-existent location
```

**Key fields:**

| Field | Meaning |
|-------|---------|
| `worktree <path>` | Absolute path to the worktree directory |
| `HEAD <sha>` | Full commit SHA the worktree is checked out at |
| `branch <ref>` | Full ref name (`refs/heads/...`) |
| `detached` | Worktree is in detached HEAD state |
| `locked [reason]` | Worktree is locked; optional reason follows on same line |
| `prunable <reason>` | Worktree's gitdir is gone; safe to clean with `git worktree prune` |

Use `--porcelain` when scripting — the default format can change between git versions.

---

### `git worktree remove <path>`

Removes a linked worktree: deletes the directory and unregisters it from git's internal tracking.

```bash
git worktree remove /home/user/.claude/worktrees/bold-turing
```

**Flags:**

| Flag | Effect |
|------|--------|
| `--force` | Remove even if the worktree has untracked files, uncommitted changes, or an unresolved HEAD. **Data loss risk** — any uncommitted work is gone. |

Fails silently on nothing: if the path does not exist as a registered worktree, git reports an error rather than silently succeeding.

**When to use:** Small numbers of worktrees (under ~10). For bulk cleanup, `rm -rf` + `git worktree prune` is significantly faster because it avoids per-worktree git overhead.

---

### `git worktree prune`

Scans git's internal worktree registry (`<repo>/.git/worktrees/`) and removes entries whose corresponding directories no longer exist on disk.

```bash
git worktree prune
```

**Flags:**

| Flag | Effect |
|------|--------|
| `--dry-run` | Show what would be pruned without deleting anything |
| `--verbose` | Print each entry being pruned |
| `--expire <time>` | Only prune entries older than `<time>` (e.g., `--expire=now` prunes everything prunable, `--expire=2.weeks.ago` is the default) |

**What it does NOT do:** It does not delete worktree directories from disk. It only removes stale entries from `.git/worktrees/`. You must delete the directories first (or they were already gone), then run `prune` to clean up git's tracking data.

---

### `git worktree lock` / `git worktree unlock`

Marks a worktree as active so that `git worktree prune` skips it.

```bash
git worktree lock /path/to/worktree --reason "active Claude session"
git worktree unlock /path/to/worktree
```

Claude Code uses this mechanism to protect live sessions. A locked worktree has a `locked` file at `<repo>/.git/worktrees/<name>/locked`.

---

## Internal Structure: `.git/worktrees/<name>/`

For each registered worktree, git maintains a subdirectory inside the main repo's `.git/worktrees/`:

```
<main-repo>/.git/worktrees/<name>/
├── gitdir          ← path to the worktree's .git file (text file)
├── HEAD            ← current branch/commit in the worktree (text file)
├── commondir       ← points back to the main repo's .git/ (text file, contains "../..")
├── locked          ← exists only if the worktree is locked; may contain a reason string
└── config          ← rarely present; per-worktree config overrides
```

**`gitdir`** — contains the absolute path to `<worktree-dir>/.git`, which is itself a text file (not a directory) pointing back here. This bidirectional link is how git connects the worktree directory to its tracking entry. If the worktree directory is deleted, `gitdir` points to a non-existent location — this is the `prunable` condition.

**`HEAD`** — the worktree's current ref, e.g. `ref: refs/heads/claude/fix-auth` or a bare SHA for detached state.

**`locked`** — presence of this file (regardless of content) marks the worktree as locked. Content is an optional human-readable reason.

---

## Common Error Messages

### `fatal: 'path' is already checked out`
Another worktree (or the main tree) is already on that branch. Git prevents two worktrees from being on the same branch simultaneously to avoid conflicting index state. Checkout a new branch or use a different worktree.

### `fatal: validation failed, cannot lock`
Attempted `git worktree lock` on a worktree that is already locked, or lock file could not be created (permissions issue).

### `fatal: not a git repository`
Running a git command from inside a worktree directory that has lost its `.git` file — the file that points back to the main repo's `.git/`. This happens if the `.git` file was accidentally deleted. Check `<main-repo>/.git/worktrees/<name>/gitdir` to find where the `.git` file should be, then recreate it.

### `error: HEAD file of worktree is garbled`
The `HEAD` file inside `.git/worktrees/<name>/` is corrupted or empty. Run `git worktree prune --verbose` to inspect, then manually remove the corrupted entry from `.git/worktrees/` if needed.

### `error: failed to lock new config file`
Git cannot write to the repo config during a worktree operation, usually a permissions issue or another git process holding a lock. Check for stale `config.lock` files.

### `warning: gitdir file points to non-existent location`
Emitted by `git worktree list` when a worktree directory has been manually deleted. Safe to resolve with `git worktree prune`.
