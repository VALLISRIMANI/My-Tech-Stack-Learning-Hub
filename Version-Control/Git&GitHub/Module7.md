📍 **Module 7 of 8: Collaboration & GitHub Workflows**

---

## 🍴 Forking Workflow — Contributing Without Direct Access

### Why Forking Exists

When you join a company, you're usually added as a collaborator to the repository and can push branches directly. But in **open source**, you can't push to a repository you don't own. Thousands of strangers can't all have write access to, say, the React or Django codebase.

**Forking** solves this. A fork is a **complete copy of a repository under your own GitHub account**. You have full write access to your fork. You do your work there, then propose your changes back to the original via a pull request.

```
Original Repo                    Your Fork
(django/django)    --fork-->    (alexrivera/django)
     ↑                                 ↓
     └─────── Pull Request ───── your changes
```

This is also useful within companies for contractors, external contributors, or security-sensitive repos where even internal developers work on forks.

---

### Forking a Repository

On GitHub, navigate to any repository and click the **Fork** button in the top right. GitHub creates `yourusername/reponame` as a full copy under your account — entire history, all branches, all tags.

Clone your fork locally:
```
$ git clone git@github.com:alexrivera/some-project.git
Cloning into 'some-project'...
```

After cloning, `origin` points to your fork. But you also need a connection to the **original repository** (called the **upstream**) so you can sync new changes from it:

```
$ git remote add upstream git@github.com:originalowner/some-project.git
```

Verify both remotes:
```
$ git remote -v
origin      git@github.com:alexrivera/some-project.git (fetch)
origin      git@github.com:alexrivera/some-project.git (push)
upstream    git@github.com:originalowner/some-project.git (fetch)
upstream    git@github.com:originalowner/some-project.git (push)
```

The convention is always:
- `origin` = your fork
- `upstream` = the original repository you forked from

---

### Syncing With Upstream

The original project keeps moving — maintainers merge other contributors' work constantly. Your fork gets stale. Here's how to sync:

```
$ git fetch upstream
remote: Enumerating objects: 18, done.
remote: Counting objects: 100% (18/18), done.
From git@github.com:originalowner/some-project
   7a1b2c3..f9e8d7c  main -> upstream/main

$ git switch main
$ git merge upstream/main
Updating 7a1b2c3..f9e8d7c
Fast-forward
 README.md        | 4 ++++
 requirements.txt | 2 ++
 2 files changed, 6 insertions(+)

$ git push origin main
```

This three-step pattern — fetch upstream, merge into local main, push to your fork — keeps your fork in sync with the original. Do this before starting any new feature branch.

---

### The Full Fork Contribution Cycle

```
# 1. Sync your fork with upstream
$ git fetch upstream
$ git switch main
$ git merge upstream/main

# 2. Create a feature branch
$ git switch -c fix/correct-tax-calculation

# 3. Do your work and commit
$ git add routes/orders.py
$ git commit -m "fix: correct tax calculation for EU customers"

# 4. Push to YOUR fork (origin)
$ git push -u origin fix/correct-tax-calculation

# 5. Open a Pull Request on GitHub:
#    from: alexrivera/some-project  fix/correct-tax-calculation
#    into: originalowner/some-project  main
```

**GOTCHA:** Always work on a **feature branch**, never directly on `main` in your fork. If you commit to `main`, syncing with upstream becomes complicated because your `main` and upstream's `main` have diverged. Keep your fork's `main` as a clean mirror of upstream.

**GOTCHA:** If the upstream project moves while your PR is under review, you may be asked to update your branch. The cleanest way is:
```
$ git fetch upstream
$ git rebase upstream/main
$ git push --force-with-lease origin fix/correct-tax-calculation
```
`--force-with-lease` is a safer version of `--force` — it refuses to push if someone else has pushed to the remote branch since you last fetched, preventing accidental overwrites. This is the one legitimate use of force-pushing: your own fork's feature branch, never a shared branch.

---

Now that your changes are pushed, you open a Pull Request. This is the central collaboration tool in modern development.

---

## 🔁 Pull Requests — The Collaboration Interface

### What a Pull Request Actually Is

A Pull Request (PR) is not a Git concept — it's a **GitHub feature**. It's a formal request to merge one branch into another, wrapped in a conversation interface. It gives your team a place to:

- Review exactly what code is changing (line-by-line diff)
- Leave comments on specific lines
- Request changes before approving
- Run automated checks (CI/CD — covered in Module 8)
- Document *why* the change was made for future developers

A PR is as much a **communication artifact** as it is a merge mechanism. The description, comments, and review thread become permanent documentation of the decision-making behind a change.

---

### Creating a Pull Request

After pushing a branch, GitHub shows a yellow banner: "Compare & pull request." Click it, or go to the Pull Requests tab → New pull request.

A good PR has:

**Title** — specific, follows Conventional Commits style:
```
fix: correct tax calculation for EU customers
```

**Description** — answers three questions:
```
## What does this PR do?
Fixes an incorrect VAT rate being applied to EU customer orders.
The rate was hardcoded at 20% but should vary by country.

## Why is this change needed?
Reported in issue #47 — EU customers were being overcharged.

## How to test it?
1. Run `pytest tests/test_orders.py::test_eu_tax_rates`
2. Manually create an order with a German address and verify 19% VAT

Closes #47
```

`Closes #47` is a magic keyword — when the PR merges, GitHub automatically closes Issue #47. More on this shortly.

---

### Code Review — Giving and Receiving Feedback

Reviewers can comment on any line of the diff by clicking the `+` icon next to it.

**Types of review outcomes:**

- **Comment** — general feedback, no explicit approval/rejection
- **Approve** — LGTM (looks good to me), ready to merge
- **Request Changes** — must be addressed before merge is allowed

**As a reviewer, good comments are specific and constructive:**

Instead of:
```
This is wrong.
```

Write:
```
This will fail for orders with zero items because `sum()` on an empty
list returns 0 but the downstream function expects at least one item.
Consider adding a guard clause: `if not items: return Decimal('0.00')`
```

**As an author, responding to review:**

- Address each comment — either make the change or explain why you disagree
- Push new commits to the same branch — the PR updates automatically
- Resolve conversations once addressed
- Don't force-push during active review — it makes the diff history unreadable (unless rebasing, then coordinate with reviewers)

**GOTCHA:** When you push new commits to a branch with an open PR, GitHub adds them to the PR automatically. You never need to close and reopen a PR. This is why you always work on a dedicated branch — not `main` — for every PR.

---

### Draft Pull Requests

If you want feedback early but the PR isn't ready to merge:
```
Open PR → Mark as Draft
```

Draft PRs show a grey badge instead of green. They can't be merged until marked "Ready for Review." This signals to teammates: "I want eyes on this direction, but don't approve it yet."

---

Now the PR is approved. How does it land on `main`? GitHub offers three strategies and the choice matters for your history.

---

## 🔀 Merging PRs on GitHub — Three Strategies

GitHub offers three buttons when merging a PR. Each produces a different history shape. Understanding the tradeoffs is essential.

---

### Strategy 1: Create a Merge Commit

```
Before:
main:    A --- B --- C
                      \
feature:               D --- E --- F

After:
main:    A --- B --- C ----------- M
                      \           /
feature:               D --- E --- F
```

A merge commit `M` is created with two parents. The full branch history is preserved exactly as it happened.

**When to use:** When you want to preserve the complete history of how a feature was built. Teams that value historical accuracy over linear history prefer this.

**Downside:** With many PRs, `git log --graph` becomes a complex web of merge commits.

---

### Strategy 2: Squash and Merge

```
Before:
main:    A --- B --- C
feature:              D --- E --- F   (3 commits)

After:
main:    A --- B --- C --- S
```

All commits from the feature branch (`D`, `E`, `F`) are squashed into a single new commit `S` on `main`. The branch is not preserved in history.

**When to use:** When feature branches have messy intermediate commits (WIP saves, typo fixes) and you want `main`'s history to be one clean commit per feature. Very popular for fast-moving teams.

**Downside:** Individual commits from the branch are lost. If the squashed commit introduces a bug, `git bisect` lands on the squash commit — you can't bisect within the original branch commits.

---

### Strategy 3: Rebase and Merge

```
Before:
main:    A --- B --- C
feature:              D --- E --- F

After:
main:    A --- B --- C --- D' --- E' --- F'
```

Each feature branch commit is replayed onto `main` individually — like a `git rebase` followed by a fast-forward merge. No merge commit. Linear history. All individual commits preserved.

**When to use:** When your feature branch has well-crafted, meaningful individual commits that tell a story, and you want linear history without losing them.

**Downside:** Rewrites commit hashes (`D'`, `E'` instead of `D`, `E`). The branch and main diverge permanently after merge.

---

### Comparison Table

| Strategy | History Shape | Individual Commits | Merge Commit | Best For |
|---|---|---|---|---|
| Merge Commit | Non-linear | ✅ Preserved | ✅ Created | Full audit trail |
| Squash & Merge | Linear | ❌ Collapsed | ❌ | Messy branches, clean main |
| Rebase & Merge | Linear | ✅ Preserved | ❌ | Clean branches, linear history |

**GOTCHA:** Most teams pick **one strategy and enforce it consistently**. Mixing strategies on the same repository produces inconsistent history that's hard to navigate. This is usually configured at the repo level in GitHub Settings → General → Pull Requests.

---

After merging, clean up:
```
# Delete the remote branch (GitHub offers a button after merge)
# Delete the local branch
$ git branch -d feature/product-search
$ git fetch origin --prune
```

`--prune` removes remote-tracking references to branches that no longer exist on the remote.

---

Now let's look at how work gets tracked and organized in the first place — before a single line of code is written.

---

## 📋 GitHub Issues & Project Management

### Issues — The Unit of Work

A GitHub Issue is a tracked item — a bug report, a feature request, a question, a task. Every meaningful piece of work on a professional project starts as an issue before it becomes a branch or PR.

**Creating a useful bug report issue:**

```
Title: Tax calculation returns incorrect amount for EU orders

## Description
When a customer with a German billing address places an order,
the tax calculation applies 20% VAT instead of the correct 19%.

## Steps to Reproduce
1. Create a customer with country = "DE"
2. Add any product to cart
3. Proceed to checkout
4. Observe tax line in order summary

## Expected Behavior
19% VAT applied (German VAT rate)

## Actual Behavior
20% VAT applied

## Environment
- Python 3.11
- Flask 3.0.0
- Reproduced on: main branch, commit 9f1a2b3
```

---

### Closing Issues via Keywords

In a commit message or PR description, these keywords automatically close the linked issue when merged to the default branch:

```
Closes #47
Fixes #47
Resolves #47
```

All three work. Use them in PR descriptions — when the PR merges, GitHub closes the issue and creates a cross-reference link between them. This creates a permanent audit trail: Issue #47 → PR #23 → commit `a9b8c7d`.

You can reference without closing:
```
Related to #47
See #47
```

**Cross-repository references:**
```
Fixes alexrivera/some-project#47
```

---

### Labels, Assignees, and Milestones

**Labels** categorize issues visually:
- `bug` — something is broken
- `enhancement` — new feature or improvement
- `good first issue` — suitable for new contributors (open source standard)
- `priority: high` — needs attention soon
- `blocked` — waiting on something else

**Assignees** designate who is responsible for the issue.

**Milestones** group issues into a release goal — "v1.1.0" milestone contains all issues that need to be resolved before that release ships. GitHub shows milestone progress as a percentage of closed issues.

---

### GitHub Projects — Kanban Boards

GitHub Projects is a built-in project management board that connects directly to your issues and PRs.

A standard software development board has columns:

```
┌──────────┐  ┌───────────┐  ┌─────────────┐  ┌──────────┐  ┌──────────┐
│ Backlog  │  │  To Do    │  │ In Progress │  │  Review  │  │   Done   │
├──────────┤  ├───────────┤  ├─────────────┤  ├──────────┤  ├──────────┤
│ Issue #52│  │ Issue #47 │  │ Issue #41   │  │  PR #23  │  │ Issue #38│
│ Issue #51│  │ Issue #45 │  │ Issue #39   │  │  PR #21  │  │ Issue #35│
│ Issue #50│  │ Issue #44 │  │             │  │          │  │ Issue #31│
└──────────┘  └───────────┘  └─────────────┘  └──────────┘  └──────────┘
```

**Automation:** GitHub Projects can auto-move cards:
- Issue opened → moves to **To Do**
- PR opened linking an issue → issue moves to **In Progress**
- PR merged → issue moves to **Done** (via `Closes #` keyword)

This connects your GitHub workflow end-to-end: **Issue created → Branch created → PR opened → PR reviewed → PR merged → Issue closed → Card moves to Done**.

---

### The Complete Professional Workflow

Putting everything together — here is how a feature goes from idea to production on a professional team:

```
1. Issue #52 created: "feat: add product search"
   └── Assigned to Alex, labeled "enhancement", added to v1.2.0 milestone

2. Alex syncs fork and creates branch:
   $ git fetch upstream
   $ git switch main && git merge upstream/main
   $ git switch -c feature/product-search

3. Alex does work, commits, pushes:
   $ git push -u origin feature/product-search

4. Alex opens PR:
   Title: "feat: add product search endpoint"
   Description: "Closes #52"
   → Issue #52 card moves to "In Review" automatically

5. Reviewer leaves comments on specific lines
   Alex pushes fixes, responds to comments

6. Reviewer approves

7. PR merged (Squash and Merge chosen by team policy)
   → Issue #52 automatically closed
   → Issue #52 card moves to "Done"
   → v1.2.0 milestone progress updates

8. Branch deleted on remote
   $ git branch -d feature/product-search
   $ git fetch origin --prune
```

This is the full loop. Every professional software team running on GitHub operates some version of this cycle.

---


✅ **Quick Check:** What is the difference between `origin` and `upstream` in a forking workflow? If the original project merges 10 new PRs while you're working on your feature, walk through the exact commands you'd run to get those changes into your local branch before opening your PR.
