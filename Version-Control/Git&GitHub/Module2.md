📍 **Module 2 of 8: History, Inspecting Changes & Time Travel**

---

## 📜 Viewing History — `git log`

Every commit you make is permanently recorded with a hash, author, timestamp, and message. `git log` is how you read that record.

### Basic `git log`

```
$ git log
commit 4bc91f0a3e1d2c7b8f5a9e0d1c2b3a4f5e6d7c8b
Author: Alex Rivera <alex@devteam.com>
Date:   Mon Jun 2 10:14:32 2025 +0530

    chore: pin Flask version in requirements

commit e7a3d12b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f
Author: Alex Rivera <alex@devteam.com>
Date:   Mon Jun 2 09:58:11 2025 +0530

    feat: initial project scaffold with app and requirements
```

This is readable but verbose. On a real project with hundreds of commits it becomes overwhelming. That's where flags come in.

---

### `git log --oneline`

Compresses each commit to a single line — the short hash and the subject:

```
$ git log --oneline
4bc91f0 chore: pin Flask version in requirements
e7a3d12 feat: initial project scaffold with app and requirements
```

Clean, scannable, and this is the format you'll use most in daily work.

---

### `git log --graph --decorate --all --oneline`

This is the **power combo** — essential once you start working with branches:

```
$ git log --graph --decorate --all --oneline
* 9f1a2b3 (HEAD -> main) fix: handle missing config key in db_config
* 4bc91f0 chore: pin Flask version in requirements
* e7a3d12 feat: initial project scaffold with app and requirements
```

Flag by flag:
- `--graph` — draws ASCII branch/merge lines to the left of commits
- `--decorate` — shows branch names and tags next to the relevant commits (like `HEAD -> main`)
- `--all` — shows commits from *all* branches, not just the current one
- `--oneline` — keeps it compact

> 💡 **Pro tip:** Most developers alias this to something short. You can run:
> ```
> $ git config --global alias.lg "log --graph --decorate --all --oneline"
> ```
> Then just type `git lg` forever after.

**GOTCHA:** By default, `git log` only shows the history of your *current branch*. If you've been working on another branch and switch back to `main`, you won't see those commits unless you add `--all`. Beginners often think commits have "disappeared" — they haven't, they're just on another branch.

---

### Filtering `git log`

You can narrow down history to exactly what you need:

**By author:**
```
$ git log --oneline --author="Alex Rivera"
4bc91f0 chore: pin Flask version in requirements
e7a3d12 feat: initial project scaffold with app and requirements
```

**By date:**
```
$ git log --oneline --after="2025-06-01" --before="2025-06-10"
9f1a2b3 fix: handle missing config key in db_config
4bc91f0 chore: pin Flask version in requirements
```

**By file — see only commits that touched a specific file:**
```
$ git log --oneline -- requirements.txt
4bc91f0 chore: pin Flask version in requirements
e7a3d12 feat: initial project scaffold with app and requirements
```

The `--` separator tells Git that what follows is a file path, not a branch name.

---

Now that you can read history, let's go one level deeper — not just *what* changed, but *exactly how* it changed.

---

## 🔍 Analyzing Differences — `git diff`

`git diff` shows you the line-by-line changes between two states. The key is understanding *which two states* you're comparing — this is where most beginners get confused.

### `git diff` — Working Directory vs Staging Area

No flags = compares your **unsaved edits** (Working Directory) against what's **currently staged**:

```
$ echo "DEBUG = True" >> config.py
$ git diff
diff --git a/config.py b/config.py
index e69de29..4b5f3c1 100644
--- a/config.py
+++ b/config.py
@@ -0,0 +1 @@
+DEBUG = True
```

Reading the output:
- `---` is the **before** version (what was staged / last committed)
- `+++` is the **after** version (your current working directory)
- Lines starting with `+` were **added**
- Lines starting with `-` were **removed**
- `@@` shows the line numbers of the changed section

---

### `git diff --staged` — Staging Area vs Last Commit

Once you've staged a file, plain `git diff` shows nothing (the working directory matches staging). Use `--staged` to see what's *about to be committed*:

```
$ git add config.py
$ git diff
(no output — working directory now matches staging)

$ git diff --staged
diff --git a/config.py b/config.py
index e69de29..4b5f3c1 100644
--- a/config.py
+++ b/config.py
@@ -0,0 +1 @@
+DEBUG = True
```

> 💡 **Habit to build:** Always run `git diff --staged` right before committing. It's your final sanity check — "is this exactly what I mean to record?"

---

### `git diff <commit>..<commit>` — Comparing Two Commits

```
$ git diff e7a3d12..4bc91f0
diff --git a/requirements.txt b/requirements.txt
index e69de29..9daeafb 100644
--- a/requirements.txt
+++ b/requirements.txt
@@ -0,0 +1 @@
+Flask==3.0.0
```

This shows everything that changed between those two commits. You can use short hashes (7 characters is usually enough).

**GOTCHA:** `git diff` compares the *working directory* to staging by default — not the last commit. This trips up beginners constantly. If you've staged everything, `git diff` appears blank even though you have uncommitted changes. Always remember: `git diff` = working vs staging. `git diff --staged` = staging vs last commit.

---

Now you can read history and analyze changes. What happens when you realize something went wrong and need to undo it? Git gives you a layered toolkit for this — from gentle to drastic.

---

## ↩️ Undoing Changes Safely

### `git restore` — Discarding Working Directory Changes

You edited `app.py`, realized it was a mistake, and want to throw away those edits and go back to the last committed version:

```
$ echo "import os; os.system('rm -rf /')" >> app.py   # oops
$ git restore app.py
```

No output — Git silently restores `app.py` to its last committed state. The bad edit is gone.

**GOTCHA:** `git restore` is **destructive and permanent** for uncommitted work. There is no undo. The edit was never recorded anywhere, so Git cannot recover it. Use this only when you're sure you want to throw the change away.

---

### `git restore --staged` — Unstaging Without Losing Changes

You staged a file by accident and want to move it back to the Working Directory without losing your edits:

```
$ git add config.py
$ git status
Changes to be committed:
	modified:   config.py

$ git restore --staged config.py
$ git status
Changes not staged for commit:
	modified:   config.py
```

The edits in `config.py` are still there — they've just been moved back out of the staging area. Safe operation, nothing lost.

---

### `git commit --amend` — Fixing the Most Recent Commit

You just committed and immediately notice a typo in the commit message, or you forgot to include a file. `--amend` rewrites the most recent commit:

**Fixing the message only:**
```
$ git commit --amend -m "feat: add Flask and environment configuration"
[main 7d3e1a2] feat: add Flask and environment configuration
 Date: Mon Jun 2 10:14:32 2025 +0530
 1 file changed, 1 insertion(+)
```

**Adding a forgotten file to the last commit:**
```
$ git add .env
$ git commit --amend --no-edit
```

`--no-edit` keeps the existing message — you're just adding the staged file to the commit without changing anything else.

**GOTCHA:** `--amend` rewrites history. It creates a *brand new commit* with a different SHA hash and discards the old one. This is perfectly fine for local commits that haven't been pushed to a remote yet. **Never amend a commit that's already been pushed and shared with others** — it rewrites the commit their work is based on and creates chaos. We'll cover why in detail in Module 6.

---

Now that you can undo individual file changes and fix the latest commit, let's look at the most powerful time-travel tools — for when you need to go further back.

---

## ⏳ Time Travel & Resetting — `checkout`, `reset`, `revert`

### `git checkout <hash>` — Visiting a Past Commit

```
$ git log --oneline
9f1a2b3 (HEAD -> main) fix: handle missing config key
4bc91f0 chore: pin Flask version in requirements
e7a3d12 feat: initial project scaffold

$ git checkout e7a3d12
Note: switching to 'e7a3d12'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, but if you switch back without creating a branch,
any commits you make will be lost.

HEAD is now at e7a3d12 feat: initial project scaffold
```

Your files now look exactly as they did at that commit. You can look around, run the app, inspect anything. This is **read-only time travel**.

**Detached HEAD** means HEAD (the pointer to "where you are") is pointing directly at a commit instead of at a branch. Think of it as standing at a historical snapshot — you're not on any branch's timeline.

Go back to the present:
```
$ git checkout main
Previous HEAD position was e7a3d12 feat: initial project scaffold
Switched to branch 'main'
```

---

### `git reset` — Moving the Branch Pointer Back

`git reset` moves the current branch's tip back to an earlier commit. It has three modes that differ in what happens to your changes:

Let's say your history looks like this:
```
A --- B --- C   (main, HEAD)
```
You want to undo commit C. Here's what each mode does:

---

**`git reset --soft <hash>`** — Moves HEAD back, keeps changes staged

```
$ git reset --soft 4bc91f0
```
```
A --- B   (main, HEAD)
         C's changes are in the staging area
```

Commit C is gone from history, but everything it changed is sitting staged and ready to re-commit. Use this when you want to "uncommit" but keep the work — maybe to rewrite the commit message or split it into two commits.

---

**`git reset --mixed <hash>`** — Moves HEAD back, keeps changes in working directory (DEFAULT)

```
$ git reset --mixed 4bc91f0
```
(or just `git reset 4bc91f0` — mixed is the default)

```
A --- B   (main, HEAD)
         C's changes are in the working directory, unstaged
```

Commit C is gone from history, changes are preserved but unstaged. Use this when you want to redo what you committed with more careful staging.

---

**`git reset --hard <hash>`** — Moves HEAD back, DELETES all changes

```
$ git reset --hard 4bc91f0
HEAD is now at 4bc91f0 chore: pin Flask version in requirements
```

```
A --- B   (main, HEAD)
         C's changes are GONE
```

Commit C is erased from history and every file change it made is permanently deleted. This is the nuclear option.

**GOTCHA:** `--hard` is the most dangerous command in Git. There is no recycle bin. If you haven't committed the work, it's unrecoverable. Use it deliberately and only when you are completely sure. A safer alternative for shared work is `git revert`.

---

### `git revert <hash>` — The Safe Undo for Shared History

`git revert` doesn't erase history. Instead, it creates a **new commit that is the inverse** of the target commit — it applies the opposite of every change that commit made:

```
$ git log --oneline
9f1a2b3 (HEAD -> main) fix: handle missing config key
4bc91f0 chore: pin Flask version in requirements
e7a3d12 feat: initial project scaffold

$ git revert 4bc91f0
[main d4e5f6a] Revert "chore: pin Flask version in requirements"
 1 file changed, 1 deletion(-)
```

```
$ git log --oneline
d4e5f6a (HEAD -> main) Revert "chore: pin Flask version in requirements"
9f1a2b3 fix: handle missing config key
4bc91f0 chore: pin Flask version in requirements
e7a3d12 feat: initial project scaffold
```

The original commit `4bc91f0` is still there in history — nothing was rewritten. A new commit `d4e5f6a` was added that undoes its effect.

**When to use `revert` vs `reset`:**

| Scenario | Use |
|---|---|
| Local commit, not pushed yet | `git reset` is fine |
| Commit already pushed to shared remote | Always use `git revert` |
| You want history to show the mistake and the fix | `git revert` |
| You want the mistake erased as if it never happened | `git reset` (local only) |

**GOTCHA:** `git revert` opens your editor to write the revert commit message. Add `-m` or `--no-edit` to skip that if the default message is fine.

---

✅ **Quick Check:** You have three commits: A → B → C (C is latest). You run `git reset --mixed B`. What happened to commit C? Where do its changes live now? What would you need to do to get them back into a commit?
