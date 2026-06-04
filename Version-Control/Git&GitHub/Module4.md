📍 **Module 4 of 8: Git Utilities & Advanced Local Operations**

---

## 🙈 Ignoring Files — `.gitignore`

### Why It Exists

Not every file in your project folder should be version controlled. Some files are:
- **Generated automatically** — `__pycache__/`, `*.pyc`, `node_modules/`
- **Environment-specific** — `.env` files containing secrets, API keys, database passwords
- **Editor noise** — `.vscode/`, `.idea/`, `.DS_Store`
- **Build artifacts** — `dist/`, `build/`, `*.egg-info/`

Committing these causes real problems: secrets leak into history, binary files bloat the repo, and every developer's machine generates slightly different noise files that create constant meaningless diffs.

`.gitignore` is a plain text file you place in your repository that tells Git which files and patterns to permanently ignore.

---

### Syntax and Rules

Create a `.gitignore` in the root of your project:

```
$ touch .gitignore
```

Inside it, each line is a pattern:

```
# Comments start with a hash

# Ignore a specific file
.env

# Ignore all files with a specific extension
*.pyc
*.log

# Ignore an entire directory
__pycache__/
node_modules/
venv/

# Ignore a directory anywhere in the tree
**/__pycache__/

# Ignore all .txt files EXCEPT this one specific file
*.txt
!important_notes.txt

# Ignore files only in the root, not subdirectories
/config.local.py
```

A real Python web app `.gitignore`:

```
# Python
__pycache__/
*.py[cod]
*.egg-info/
dist/
build/
venv/
.env

# Editor
.vscode/
.idea/
.DS_Store

# Logs
*.log
logs/

# Testing
.coverage
htmlcov/
```

After saving this file:
```
$ git status
Untracked files:
	.gitignore

(nothing else — all ignored files are now invisible to git status)
```

Stage and commit the `.gitignore` itself — it's a project file that should be shared:
```
$ git add .gitignore
$ git commit -m "chore: add .gitignore for Python and editor files"
[main b3c4d5e] chore: add .gitignore for Python and editor files
 1 file changed, 18 insertions(+)
 create mode 100644 .gitignore
```

---

### Global `.gitignore`

Some patterns should be ignored across **all your projects** — like `.DS_Store` on macOS or `.idea/` from JetBrains IDEs. You can set a global ignore file:

```
$ git config --global core.excludesfile ~/.gitignore_global
$ echo ".DS_Store" >> ~/.gitignore_global
$ echo ".idea/" >> ~/.gitignore_global
```

This file applies to every repository on your machine without touching individual project `.gitignore` files.

---

### Untracking a File That's Already Committed

`.gitignore` only affects **untracked files**. If you accidentally committed `.env` in a previous commit, adding it to `.gitignore` won't stop Git from tracking further changes to it.

To stop tracking it without deleting it from your filesystem:

```
$ git rm --cached .env
rm '.env'
$ git commit -m "fix: stop tracking .env file with secrets"
[main c4d5e6f] fix: stop tracking .env file with secrets
 1 file changed, 0 insertions(+), 0 deletions(-)
 delete mode 100644 .env
```

`--cached` means "remove from Git's index (staging area) only — leave the actual file on disk." After this commit, `.env` is gone from the repository but still exists locally.

**GOTCHA:** Removing a file from tracking doesn't erase it from history. If `.env` contained real secrets, those secrets are still readable in older commits. For a true secret leak, you need `git filter-branch` or the BFG Repo Cleaner to rewrite history — and you need to rotate the leaked credentials immediately regardless.

**GOTCHA:** A pattern in `.gitignore` won't work if Git is already tracking that file. Always `git rm --cached` first, then commit, then the `.gitignore` pattern takes effect.

---

Now that your repo is clean of noise files, let's talk about what happens when you're in the middle of work and need to switch context immediately.

---

## 📦 Stashing — `git stash`

### Why It Exists

You're halfway through building `feature/payment-gateway` — `routes/payments.py` is modified but not ready to commit. Suddenly your team lead says there's a critical bug on `main` that needs fixing right now.

You can't switch branches with uncommitted changes that conflict with the other branch. You're not ready to commit either — the work is incomplete. This is exactly what `git stash` is for.

**Stash is a temporary clipboard for work-in-progress.** It takes all your uncommitted changes (staged and unstaged), saves them in a separate stack, and restores your working directory to a clean state.

---

### `git stash push` — Saving Work in Progress

```
$ git status
On branch feature/payment-gateway
Changes not staged for commit:
	modified:   routes/payments.py
	modified:   app.py

$ git stash push -m "WIP: payment gateway integration, halfway through validation"
Saved working directory and index state On feature/payment-gateway: WIP: payment gateway integration, halfway through validation
```

Now your working directory is clean:
```
$ git status
On branch feature/payment-gateway
nothing to commit, working tree clean
```

You can now freely switch to `main`, fix the bug, commit it, and come back.

The `-m` flag adds a description. Without it, Git auto-generates one from the branch name and last commit — but that's hard to identify later if you have multiple stashes.

---

### `git stash list` — Viewing the Stash Stack

Stashes accumulate in a stack (last in, first out):

```
$ git stash list
stash@{0}: On feature/payment-gateway: WIP: payment gateway integration, halfway through validation
stash@{1}: On feature/user-auth: WIP: session handling incomplete
```

`stash@{0}` is always the most recent. The number in `{}` is the index.

---

### `git stash pop` — Restoring and Removing

After fixing the bug on `main` and switching back:

```
$ git switch feature/payment-gateway
$ git stash pop
On branch feature/payment-gateway
Changes not staged for commit:
	modified:   routes/payments.py
	modified:   app.py

Dropped stash@{0}
```

`pop` restores the most recent stash and **removes it from the stash list**. Your working directory is back exactly as you left it.

---

### `git stash apply` — Restoring Without Removing

If you want to apply a stash but keep it in the list (useful for applying the same stash to multiple branches):

```
$ git stash apply stash@{1}
```

You have to manually `git stash drop stash@{1}` when you're done with it.

---

### `git stash drop` and `git stash clear`

Drop a specific stash:
```
$ git stash drop stash@{0}
Dropped stash@{0} (3f2a1b9c...)
```

Wipe the entire stash list:
```
$ git stash clear
```

**GOTCHA:** `git stash pop` can cause conflicts just like merging — if the branch has changed since you stashed, the restored changes might conflict with newer commits. Git will mark the conflicts exactly like a merge conflict, and you resolve them the same way.

**GOTCHA:** By default, `git stash` does **not** stash untracked files (new files you've never staged). Use `git stash push -u` (`-u` for untracked) to include them. Beginners often stash, switch branches, and find a new file they forgot to stash is still sitting in their working directory.

---

Now your working directory is clean and your history is organized. Let's talk about permanently marking important moments in that history.

---

## 🏷️ Tagging — `git tag`

### Why It Exists

Branches move — every new commit advances the branch pointer. But sometimes you want a **permanent, immovable marker** on a specific commit. Releases are the classic use case: `v1.0.0`, `v1.2.3`, `v2.0.0-beta`. Tags are how you tell the world (and your deployment scripts) "this exact commit is what we shipped."

---

### Semantic Versioning (SemVer) — The Standard

Before creating tags, understand the naming convention. SemVer uses the format `MAJOR.MINOR.PATCH`:

| Part | Increment When | Example |
|---|---|---|
| MAJOR | Breaking, incompatible changes | `1.0.0` → `2.0.0` |
| MINOR | New features, backward compatible | `1.0.0` → `1.1.0` |
| PATCH | Bug fixes, backward compatible | `1.0.0` → `1.0.1` |

Pre-release suffixes: `v1.0.0-alpha`, `v1.0.0-beta`, `v2.0.0-rc.1`

---

### Lightweight vs Annotated Tags

**Lightweight tag** — just a pointer to a commit, nothing more:
```
$ git tag v0.1.0
```

**Annotated tag** — a full Git object with tagger name, email, date, and a message. This is what you should always use for releases:
```
$ git tag -a v1.0.0 -m "Release v1.0.0 - initial public launch"
```

Flag breakdown:
- `-a` — create an annotated tag
- `v1.0.0` — the tag name
- `-m` — the tag message (like a commit message)

---

### Viewing Tags

```
$ git tag
v0.1.0
v1.0.0

$ git show v1.0.0
tag v1.0.0
Tagger: Alex Rivera <alex@devteam.com>
Date:   Tue Jun 3 11:30:00 2025 +0530

Release v1.0.0 - initial public launch

commit 7a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b
Author: Alex Rivera <alex@devteam.com>
Date:   Tue Jun 3 11:20:00 2025 +0530

    feat: finalize user auth and payment routes
```

`git show` on a tag displays the tag metadata first, then the commit it points to.

---

### Tagging a Past Commit

You can tag any commit retroactively by providing its hash:

```
$ git log --oneline
7a1b2c3 feat: finalize user auth and payment routes
4bc91f0 chore: pin Flask version in requirements
e7a3d12 feat: initial project scaffold

$ git tag -a v0.1.0 4bc91f0 -m "Early development checkpoint"
```

---

### Deleting a Tag

```
$ git tag -d v0.1.0
Deleted tag 'v0.1.0' (was 4bc91f0)
```

**GOTCHA:** Tags are **not pushed to remotes automatically** when you run `git push`. You have to push them explicitly — we'll cover the exact commands in Module 5. Many developers tag a release locally and then wonder why it doesn't appear on GitHub.

**GOTCHA:** Unlike branches, tags don't move when you make new commits. They are permanently fixed to the commit they were created on. If you need to "move" a tag, you delete it and recreate it — but this is dangerous if others have already pulled the tag.

---

You can see your history and tag important points in it. Now let's learn to *search* through that history and your codebase with surgical precision.

---

## 🔎 Searching History — `git grep` and `git log -S`

### `git grep` — Searching File Contents

`git grep` searches through the **current state of your tracked files** for a pattern — like `grep` but Git-aware (respects `.gitignore`, only searches tracked files):

```
$ git grep "def login"
routes/auth.py:def login():
```

Search across a specific commit or branch instead of current working tree:
```
$ git grep "def login" feature/user-auth
routes/auth.py:def login():
```

Show line numbers:
```
$ git grep -n "DEBUG"
config.py:1:DEBUG = True
```

Case-insensitive search:
```
$ git grep -i "flask"
requirements.txt:1:Flask==3.0.0
app.py:3:from flask import Flask
```

Show context lines around the match:
```
$ git grep -n -A 2 "def login"
routes/auth.py:1:def login():
routes/auth.py:2-    # TODO: implement JWT
routes/auth.py:3-    pass
```

`-A 2` shows 2 lines **after** each match. `-B 2` shows lines **before**.

**When to use `git grep` over regular `grep`:** It's faster on large repositories because it only searches tracked files. It also lets you search any point in history, not just the current files.

---

### `git log -S` — The Pickaxe

This is one of the most powerful and underused Git features. The **pickaxe** (`-S`) searches through commit history to find the exact commit where a specific string was **added or removed**.

Scenario: You're debugging and find a function `calculate_order_total` that has a bug. You want to know which commit introduced it:

```
$ git log -S "calculate_order_total" --oneline
3d4e5f6 feat: add order total calculation to checkout flow
```

Git scanned every commit in history and found exactly which one changed the number of occurrences of that string. That's the commit that introduced it.

Show the full diff of what changed in that commit:
```
$ git log -S "calculate_order_total" --oneline -p
3d4e5f6 feat: add order total calculation to checkout flow

diff --git a/routes/orders.py b/routes/orders.py
index e69de29..7c3f1a2 100644
--- a/routes/orders.py
+++ b/routes/orders.py
@@ -0,0 +1,4 @@
+def calculate_order_total(items):
+    return sum(item['price'] * item['qty'] for item in items)
```

**`-S` vs `-G`:** `-S` finds commits where the **count** of the string changed (i.e., where it was added or deleted). `-G` accepts a regex and finds commits where any line matching the pattern was added or removed — more flexible but slower.

```
$ git log -G "def calculate_.*total" --oneline
3d4e5f6 feat: add order total calculation to checkout flow
```

**GOTCHA:** The pickaxe searches string occurrences, not just file contents. If a string appears 3 times in a file and a commit changes it to 4 occurrences, `-S` will find that commit. If a refactor moves the string from one file to another without changing the total count, `-S` won't find it — use `-G` in that case.

---

### Combining Search Tools

A real debugging workflow often chains these together:

```
# Step 1: Find which file contains the suspicious function
$ git grep -n "process_payment"
routes/payments.py:14:def process_payment(amount, card):

# Step 2: Find which commit introduced or last changed it
$ git log -S "process_payment" --oneline
8f9a0b1 feat: implement payment processing endpoint

# Step 3: See the full diff of that commit
$ git show 8f9a0b1
```

This three-step pattern — grep to locate, pickaxe to find the commit, show to inspect it — is how experienced developers diagnose regressions quickly.

---


✅ **Quick Check:** You have uncommitted changes on `feature/payment-gateway` and need to urgently fix a bug on `main`. Walk through the exact sequence of Git commands you would run — from stashing to fixing to restoring your work — in the correct order.
