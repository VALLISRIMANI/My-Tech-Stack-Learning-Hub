📍 **Module 6 of 8: Advanced Git & Rewriting History**

---

## 🔀 Rebasing — `git rebase`

### Why Rebasing Exists

Recall from Module 5 that when you merge a feature branch into `main`, a three-way merge creates a merge commit. On an active team, those accumulate fast and history looks like tangled spaghetti:

```
$ git log --graph --oneline
*   a1b2c3d (main) Merge branch 'feature/search'
|\
| * 7f8e9d0 feat: add search pagination
| * 3c4d5e6 feat: scaffold search route
* | f1e2d3c fix: correct tax calculation
* | 9b8a7f6 chore: update dependencies
|/
*   2e3f4a5 (previous merge) Merge branch 'feature/auth'
|\
...
```

Rebase gives you an alternative: instead of merging, **replay your branch's commits on top of the target branch**, producing a perfectly linear history as if you had written your feature after all the latest `main` changes.

---

### How Rebase Works Internally

Before rebase — you branched off `main` at commit `C`, and `main` has since moved forward to `F`:

```
          D --- E          (feature/product-search)
         /
A --- B --- C --- F        (main)
```

After `git rebase main` from the feature branch:

```
A --- B --- C --- F --- D' --- E'    (feature/product-search)
                  ▲
                main
```

Git took commits `D` and `E`, temporarily set them aside, moved the branch base to `F`, then **replayed** each commit one by one on top of `F`. The result is `D'` and `E'` — new commits with the same changes but different SHA hashes (because their parent changed).

The feature branch now looks like it was written *after* all of `main`'s latest work. When you merge it, it'll be a clean fast-forward with no merge commit needed.

---

### Running a Rebase

```
$ git switch feature/product-search
$ git rebase main
Successfully rebased and updated refs/heads/feature/product-search.
```

If there are no conflicts, that's all it takes. Now merge cleanly:

```
$ git switch main
$ git merge feature/product-search
Updating f1e2d3c..9a8b7c6
Fast-forward
 routes/search.py | 12 ++++++++++++
 1 file changed, 12 insertions(+)
```

Perfect linear history — no merge commit, no spaghetti.

---

### Rebase Conflicts

Just like merging, rebasing can encounter conflicts when the same lines were changed on both branches. The difference is that rebase applies commits **one at a time** — so you may encounter and resolve conflicts multiple times, once per replayed commit.

```
$ git rebase main
Auto-merging app.py
CONFLICT (content): Merge conflict in app.py
error: could not apply 3c4d5e6... feat: scaffold search route
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
```

Resolution flow:
```
# 1. Fix the conflict markers in app.py manually
# 2. Stage the resolution
$ git add app.py

# 3. Continue replaying the remaining commits
$ git rebase --continue
```

If you want to abandon the entire rebase and go back:
```
$ git rebase --abort
```

Everything returns to exactly how it was before you ran `git rebase`.

**GOTCHA:** During a rebase, **do not** use `git commit` after resolving conflicts — use `git rebase --continue`. Using `git commit` creates an extra unwanted commit in the middle of the replay. This is one of the most common rebase mistakes.

---

### The Golden Rule of Rebasing

This is the single most important rule in this entire module:

> **Never rebase commits that have already been pushed to a shared remote branch.**

Here is exactly why. Rebase creates *new* commits (`D'`, `E'`) with different SHA hashes than the originals (`D`, `E`). If your colleague already pulled `D` and `E` and based new work on them, and you then rebase and force-push `D'` and `E'` — their local history now has commits (`D`, `E`) that no longer exist on the remote. Git considers their branch and the remote to have completely diverged. Merging becomes a nightmare of duplicate commits and broken history.

**Safe to rebase:** Local commits that have never been pushed. Your own private feature branch before opening a pull request.

**Never rebase:** `main`, `develop`, or any branch others are actively working on.

---

Now that you understand basic rebase, let's look at its most powerful form — interactive rebase, which lets you rewrite history with complete control.

---

## ✏️ Interactive Rebasing — `git rebase -i`

### Why It Exists

Sometimes your commit history on a feature branch is messy — you have "WIP" commits, typo fixes on top of typo fixes, debug commits you never meant to keep. Before merging into `main` or opening a pull request, you want to clean it up into a coherent, professional story.

Interactive rebase lets you **reorder, rename, combine, split, or delete** any commits in a range.

---

### Starting an Interactive Rebase

`HEAD~N` means "N commits back from HEAD." To edit the last 4 commits:

```
$ git rebase -i HEAD~4
```

Git opens your configured editor with a list like this:

```
pick 3c4d5e6 feat: scaffold search route
pick 7f8e9d0 feat: add search result model
pick b2c3d4e WIP: pagination not working yet
pick 9e0f1a2 fix: pagination working now

# Rebase f1e2d3c..9e0f1a2 onto f1e2d3c (4 commands)
#
# Commands:
# p, pick   = use commit as-is
# r, reword = use commit, but edit the commit message
# e, edit   = use commit, but stop for amending
# s, squash = meld into previous commit, keep both messages
# f, fixup  = meld into previous commit, discard this message
# d, drop   = remove commit entirely
```

The list is in **oldest-to-newest order** (top to bottom). You edit this file, save, and close — Git executes your instructions.

---

### Squash — Combining Commits

You want to combine the "WIP" and "fix" commits into the feature commit above them — they're really one logical unit:

```
pick 3c4d5e6 feat: scaffold search route
pick 7f8e9d0 feat: add search result model
squash b2c3d4e WIP: pagination not working yet
squash 9e0f1a2 fix: pagination working now
```

`squash` melds a commit into the one **above** it. Git opens the editor again to let you write the combined commit message:

```
feat: add search result model with pagination

# This is a combination of 3 commits.
# The first commit's message is:
feat: add search result model

# This is the 2nd commit message:
WIP: pagination not working yet

# This is the 3rd commit message:
fix: pagination working now
```

Edit to keep only the meaningful message, save. Result — two clean commits instead of four messy ones:

```
$ git log --oneline
a9b8c7d feat: add search result model with pagination
3c4d5e6 feat: scaffold search route
```

---

### Fixup — Squash Without Keeping the Message

`fixup` works like `squash` but automatically discards the squashed commit's message — no editor prompt:

```
pick 3c4d5e6 feat: scaffold search route
pick 7f8e9d0 feat: add search result model
fixup b2c3d4e WIP: pagination not working yet
fixup 9e0f1a2 fix: pagination working now
```

---

### Reword — Fixing a Commit Message

```
pick 3c4d5e6 feat: scaffold search route
reword 7f8e9d0 feat: add search result model
pick b2c3d4e WIP: pagination not working yet
```

When Git reaches the `reword` line, it pauses and opens the editor just for that commit message. You fix it, save, and the rebase continues.

---

### Drop — Deleting a Commit Entirely

```
pick 3c4d5e6 feat: scaffold search route
drop b2c3d4e WIP: pagination not working yet
pick 9e0f1a2 fix: pagination working now
```

The `WIP` commit and all its changes are permanently erased from history. Use this for debug commits, accidental commits, or commits that added then removed something with no net effect.

---

### Reorder — Just Move the Lines

Reordering commits is as simple as cutting and pasting lines in the editor. Git replays them in the new order. Be careful — if commits depend on each other, reordering can cause conflicts.

**GOTCHA:** Interactive rebase rewrites all commit hashes from the first edited commit onward. Everything after your first non-`pick` instruction gets a new SHA. This is why the Golden Rule applies doubly here — never interactive-rebase commits that have been shared.

---

Now let's look at a more surgical tool: applying a single specific commit from anywhere in history to your current branch.

---

## 🍒 Cherry-Picking — `git cherry-pick`

### Why It Exists

Sometimes you don't want an entire branch merge — you just want **one specific commit** from another branch applied to your current branch.

Classic scenario: a critical bug was fixed on `feature/payment-gateway` but that feature isn't ready to merge yet. You need that fix on `main` right now, without taking the half-finished feature with it.

```
$ git log --oneline feature/payment-gateway
d4e5f6a fix: prevent double-charge on payment retry   ← you want this
8b9c0d1 feat: add 3D Secure authentication            ← not ready yet
3e4f5a6 feat: scaffold payment gateway integration    ← not ready yet
```

---

### Running Cherry-Pick

Switch to the branch you want to apply the commit to, then cherry-pick by hash:

```
$ git switch main
$ git cherry-pick d4e5f6a
[main e7f8a9b] fix: prevent double-charge on payment retry
 Date: Wed Jun 4 09:15:22 2025 +0530
 1 file changed, 4 insertions(+), 1 deletion(-)
```

Git applied exactly the changes from `d4e5f6a` as a **new commit** on `main` with a new hash (`e7f8a9b`). The original commit still exists on `feature/payment-gateway` unchanged.

---

### Cherry-Picking Multiple Commits

A range of commits:
```
$ git cherry-pick d4e5f6a..8b9c0d1
```

Specific individual commits (not a range):
```
$ git cherry-pick d4e5f6a 3e4f5a6
```

---

### Cherry-Pick Without Committing

If you want to apply the changes but review them before committing:

```
$ git cherry-pick d4e5f6a --no-commit
```

The changes are applied to your working directory and staged, but no commit is created. You can inspect, modify, then commit manually.

---

### Cherry-Pick Conflicts

If the cherry-picked commit conflicts with your current branch:

```
$ git cherry-pick d4e5f6a
CONFLICT (content): Merge conflict in routes/payments.py
error: could not apply d4e5f6a... fix: prevent double-charge on payment retry
hint: After resolving the conflicts, mark them with
hint: "git add <paths>", then run "git cherry-pick --continue"
```

Resolve the conflict, stage it, then:
```
$ git cherry-pick --continue
```

Or abort entirely:
```
$ git cherry-pick --abort
```

**GOTCHA:** Cherry-picking duplicates commits — the same logical change now exists in two places in history with different SHAs. When `feature/payment-gateway` eventually merges into `main`, Git usually handles the duplicate gracefully (it sees the diff is already applied), but it can occasionally cause confusing conflicts. Use cherry-pick for genuine emergency backports, not as a substitute for proper branching strategy.

**GOTCHA:** Cherry-pick copies a commit's *diff* — the change it introduced — not the commit's *state*. If that commit relied on code introduced by a previous commit in the same branch, cherry-picking it in isolation may fail or produce broken code because its dependencies aren't there.

---

Now for the final tool in this module — one of the most elegant algorithms in Git, and one of the most underused.

---

## 🐛 Debugging with Git — `git bisect`

### Why It Exists

You have a repository with 500 commits. Your app has a bug. It definitely wasn't there 3 weeks ago but it's definitely there now. Which of those ~80 commits introduced it?

You could check out commits one by one and test each — but that's up to 80 tests in the worst case. `git bisect` uses **binary search** to find the culprit in at most `log₂(80) ≈ 7` steps. It cuts the search space in half every step.

---

### How Binary Search Works Here

You tell Git two things:
- A **bad commit** — where the bug is definitely present (usually `HEAD`)
- A **good commit** — where the bug was definitely absent (a known-good earlier commit)

Git checks out the commit exactly halfway between them. You test it. You tell Git "good" or "bad." Git eliminates half the remaining commits and checks out the new midpoint. Repeat until Git has pinpointed the exact commit.

---

### Running `git bisect`

```
$ git bisect start
$ git bisect bad                    # current HEAD has the bug
$ git bisect good v1.0.0            # v1.0.0 tag was definitely clean
Bisecting: 39 revisions left to test after this (roughly 6 steps)
[a3b4c5d6e7f8] chore: update dependencies
```

Git checked out commit `a3b4c5d6e7f8` — the midpoint. Your working directory now reflects that commit's state. Run your test:

```
$ python tests/test_orders.py
....F
FAILED: test_order_total_calculation
```

Bug is present here. Tell Git:
```
$ git bisect bad
Bisecting: 19 revisions left to test after this (roughly 5 steps)
[1f2e3d4c5b6a] feat: add discount code system
```

New midpoint. Test again:
```
$ python tests/test_orders.py
........
OK
```

No bug here. Tell Git:
```
$ git bisect good
Bisecting: 9 revisions left to test after this (roughly 4 steps)
[7c8b9a0d1e2f] refactor: extract order calculation to service layer
```

Continue until:
```
$ git bisect bad
7c8b9a0d1e2f is the first bad commit
commit 7c8b9a0d1e2f
Author: Alex Rivera <alex@devteam.com>
Date:   Fri May 30 14:22:11 2025 +0530

    refactor: extract order calculation to service layer

:040000 040000 3b4c5d.. 9e0f1a.. M    services/order_service.py
```

Git has pinpointed the exact commit. In 6 steps instead of potentially 40+.

---

### Ending a Bisect Session

Always end with:
```
$ git bisect reset
Previous HEAD position was 7c8b9a0 refactor: extract order calculation to service layer
Switched to branch 'main'
```

This returns you to the branch you were on before starting. Without this, you'll be in a detached HEAD state.

---

### Automated Bisect

If you have a test script that exits with code 0 (pass) or non-zero (fail), you can fully automate bisect:

```
$ git bisect start
$ git bisect bad HEAD
$ git bisect good v1.0.0
$ git bisect run python tests/test_orders.py
```

Git runs the test script at each midpoint automatically, marks good/bad itself, and reports the culprit commit — no manual intervention needed.

**GOTCHA:** Always run `git bisect reset` when finished. If you forget, you're stuck in bisect mode in a detached HEAD state. Check with `git status` — it will say `HEAD detached at <hash>` if you forgot.

**GOTCHA:** `git bisect` checks out historical commits — your working directory changes completely at each step. Make sure you have no uncommitted changes before starting, or they may interfere with testing. Commit or stash first.

---


✅ **Quick Check:** What is the fundamental difference between `git merge` and `git rebase` in terms of what they produce in history? And state the Golden Rule of Rebasing in your own words — not just what it says, but *why* violating it causes problems for your teammates.
