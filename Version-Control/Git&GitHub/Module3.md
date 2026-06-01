📍 **Module 3 of 8: Branching & Merging Mechanics**

---

## 🌿 Branching Strategy — How Git Branches Actually Work

### The Pointer Model

Most people imagine branches as copies of code. They're not. A branch in Git is simply a **lightweight movable pointer to a commit**.

When you make a commit, that commit has a SHA hash. The branch name (`main`, `develop`, `feature/login`) is just a label that points to one specific commit — the tip of that branch. When you make a new commit on a branch, the pointer automatically moves forward to the new commit.

There's also a special pointer called **HEAD**. HEAD points to the branch you're currently on. When you switch branches, HEAD moves. When you commit, the branch HEAD points to moves forward.

Visualized:

```
e7a3d12 --- 4bc91f0 --- 9f1a2b3
                              ▲
                            main
                              ▲
                            HEAD
```

Creating a branch doesn't copy any files. It just creates a new pointer at the current commit. That's why branching in Git is nearly instantaneous — no matter how large the repo.

---

### `git branch` — Creating and Listing Branches

List all local branches:
```
$ git branch
* main
```

The `*` marks the branch HEAD is currently on.

Create a new branch:
```
$ git branch feature/user-auth
```

This creates a new pointer at the current commit. It does **not** switch you to it.

```
$ git branch
  feature/user-auth
* main
```

Visualized:
```
e7a3d12 --- 4bc91f0 --- 9f1a2b3
                              ▲
                            main  ← HEAD
                              ▲
                     feature/user-auth
```

Both pointers sit at the same commit for now. The moment you make a new commit on either branch, they diverge.

Delete a branch (after it's been merged):
```
$ git branch -d feature/user-auth
Deleted branch feature/user-auth (was 9f1a2b3).
```

Force delete an unmerged branch:
```
$ git branch -D feature/user-auth
```

**GOTCHA:** `-d` refuses to delete a branch with unmerged commits — Git is protecting you from losing work. `-D` overrides that protection. Use `-D` only when you deliberately want to throw away that branch's work.

---

### `git switch` — Changing Branches

```
$ git switch feature/user-auth
Switched to branch 'feature/user-auth'
```

HEAD now points to `feature/user-auth`. Your working directory updates instantly to reflect that branch's state.

Create and switch in one step:
```
$ git switch -c feature/payment-gateway
Switched to a new branch 'feature/payment-gateway'
```

`-c` stands for "create." This is the command you'll type dozens of times a day.

> 💡 **Historical note:** `git checkout -b` does the same thing and you'll see it everywhere in older tutorials and Stack Overflow answers. `git switch` was introduced in Git 2.23 (2019) as a cleaner, more focused command. Both work — prefer `git switch` for clarity.

**GOTCHA:** Git will stop you from switching branches if you have uncommitted changes that conflict with the target branch. Either commit your changes first, or use `git stash` (covered in Module 4) to temporarily set them aside.

---

### Making Commits on a Branch

Let's do this for real. You're adding a login route to the app:

```
$ git switch -c feature/user-auth
Switched to a new branch 'feature/user-auth'

$ touch routes/auth.py
$ git add routes/auth.py
$ git commit -m "feat: scaffold auth routes module"
[feature/user-auth 3c7d8e9] feat: scaffold auth routes module
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 routes/auth.py

$ echo "def login(): pass" >> routes/auth.py
$ git add routes/auth.py
$ git commit -m "feat: add login endpoint stub"
[feature/user-auth 7a1b2c3] feat: add login endpoint stub
 1 file changed, 1 insertion(+)
```

Now history looks like this:

```
                           3c7d8e9 --- 7a1b2c3
                          /                  ▲
e7a3d12 --- 4bc91f0 --- 9f1a2b3       feature/user-auth ← HEAD
                              ▲
                            main
```

Switch back to `main` and notice `routes/auth.py` doesn't exist there — it only exists on `feature/user-auth`:

```
$ git switch main
Switched to branch 'main'

$ ls routes/
(empty — auth.py doesn't exist on main yet)
```

That's branch isolation working exactly as intended.

---

Now your feature is done and you want to bring it back into `main`. That's merging.

---

## 🔀 Merging Workflows — Fast-Forward vs Three-Way

### Fast-Forward Merge

A fast-forward merge happens when the branch you're merging in is **directly ahead** of the branch you're merging into — there's a straight line of commits between them with no divergence.

Situation: `main` hasn't moved since `feature/user-auth` branched off:

```
                           3c7d8e9 --- 7a1b2c3
                          /                  ▲
e7a3d12 --- 4bc91f0 --- 9f1a2b3       feature/user-auth
                              ▲
                            main ← HEAD
```

Merge:
```
$ git switch main
$ git merge feature/user-auth
Updating 9f1a2b3..7a1b2c3
Fast-forward
 routes/auth.py | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 routes/auth.py
```

Git simply slides the `main` pointer forward to `7a1b2c3`. No new commit is created. History remains a straight line:

```
e7a3d12 --- 4bc91f0 --- 9f1a2b3 --- 3c7d8e9 --- 7a1b2c3
                                                       ▲
                                               main ← HEAD
                                                       ▲
                                              feature/user-auth
```

**GOTCHA:** Fast-forward merges produce clean linear history, which looks tidy, but the fact that a feature branch existed is lost from the log. If you want to preserve the branch structure visually in history, use `git merge --no-ff` to force a merge commit even when fast-forward is possible. Teams often have opinions about this — some prefer linear history, others prefer explicit merge commits.

---

### Three-Way Merge

A three-way merge happens when **both branches have diverged** — each has commits the other doesn't. Git can't just slide a pointer; it has to combine two different timelines.

Situation: while `feature/user-auth` was being developed, someone also committed directly to `main`:

```
                           3c7d8e9 --- 7a1b2c3
                          /                  ▲
e7a3d12 --- 4bc91f0 --- 9f1a2b3 --- f5e4d3c  feature/user-auth
                                          ▲
                                        main ← HEAD
```

Git uses **three commits** to perform the merge:
1. The tip of `main` (`f5e4d3c`)
2. The tip of `feature/user-auth` (`7a1b2c3`)
3. Their **common ancestor** (`9f1a2b3`) — the last commit both branches share

```
$ git merge feature/user-auth
Merge made by the 'ort' strategy.
 routes/auth.py | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 routes/auth.py
```

Git automatically creates a **merge commit** — a special commit with two parents:

```
                           3c7d8e9 --- 7a1b2c3
                          /                   \
e7a3d12 --- 4bc91f0 --- 9f1a2b3 --- f5e4d3c --- M
                                                ▲
                                          main ← HEAD
```

`M` is the merge commit. It has two parents and represents the moment the two timelines were rejoined.

---

When Git can combine both branches' changes automatically, it does so silently. But sometimes the same lines were changed in both branches — that's a **conflict**.

---

## ⚔️ Merge Conflicts — When Git Needs Your Help

A conflict occurs when **the same lines of the same file were changed differently on both branches**. Git doesn't know which version is correct — that's a human judgment — so it stops and asks you to decide.

### Triggering a Conflict

On `main`, someone edited `app.py` line 1:
```
$ git switch main
$ echo "# Main App v2" > app.py
$ git add app.py && git commit -m "docs: update app header comment"
```

On `feature/user-auth`, the same line was edited differently:
```
$ git switch feature/user-auth
$ echo "# Main App - Auth Branch" > app.py
$ git add app.py && git commit -m "docs: label app for auth work"
```

Now merge:
```
$ git switch main
$ git merge feature/user-auth
Auto-merging app.py
CONFLICT (content): Merge conflict in app.py
Automatic merge failed; fix conflicts and then commit the result.
```

---

### Reading Conflict Markers

Open `app.py` and you'll see this:

```
<<<<<<< HEAD
# Main App v2
=======
# Main App - Auth Branch
>>>>>>> feature/user-auth
```

Breaking it down:
- `<<<<<<< HEAD` — start of the conflict block. Everything below this line until `=======` is **your current branch's version** (`main`)
- `=======` — the dividing line
- `>>>>>>> feature/user-auth` — everything between `=======` and this line is **the incoming branch's version**

---

### Resolving the Conflict

You must manually edit the file to produce the correct final result. Delete all three marker lines and write what the file should actually contain:

```
# Main App v2 - with Auth
```

Save the file. Then tell Git you've resolved it by staging it:

```
$ git add app.py
$ git status
All conflicts fixed but you are still merging.
  (use "git commit" to conclude merge)
```

Finalize the merge with a commit:

```
$ git commit -m "merge: integrate feature/user-auth into main"
[main a9b8c7d] merge: integrate feature/user-auth into main
```

---

### Aborting a Merge

If you get into a conflict and realize it's not the right time to merge, you can cancel entirely and go back to the pre-merge state:

```
$ git merge --abort
```

Everything goes back to exactly how it was before you ran `git merge`.

**GOTCHA:** The most common beginner mistake during a conflict is forgetting to remove the marker lines (`<<<<<<<`, `=======`, `>>>>>>>`). If you commit the file with those markers still in it, your application code will be broken and contain literal angle-bracket syntax. Always double-check the file after resolving.

**GOTCHA:** `git status` is your best friend during a merge. Run it constantly. It tells you exactly which files are conflicted, which are resolved, and what to do next. Never guess — just run `git status`.

---


✅ **Quick Check:** What are the three commits Git uses to perform a three-way merge, and why does it need all three? What does Git produce as output of a three-way merge that a fast-forward merge does not?
