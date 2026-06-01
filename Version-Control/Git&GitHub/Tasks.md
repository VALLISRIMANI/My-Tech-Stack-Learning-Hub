## 🏆 Module 1: Mini-Project

You're starting version control on a new Python web application. Complete these tasks:

1. Create a new directory called `portfolio-app` and initialize a Git repository inside it.
2. Configure your `user.name` and `user.email` globally (use any realistic name/email).
3. Create three files: `app.py`, `config.py`, and `README.md`.
4. Stage only `app.py` and `config.py` — leave `README.md` unstaged. Run `git status` and observe the difference between staged and untracked files.
5. Commit the two staged files with a meaningful commit message following the Conventional Commits style (`feat:`, `chore:`, etc.).
6. Now stage and commit `README.md` separately with its own commit message.
7. Add a line of content to `app.py`, then stage and commit that change.

**Reflection question:** You have a file with two separate bug fixes in it. How would Git's staging area let you commit them as two separate commits even though they're in the same file? (Hint: look up `git add -p` — we'll cover it later, but think about why this would be useful.)

---


## 🏆 Module 2: Mini-Project

You're working on a Python web app and need to investigate and clean up some messy commit history. Complete these tasks:

1. In your `portfolio-app` repo from Module 1, add a new file `db_config.py` with some content, stage it, and commit it with the message `"feat: add database configuration module"`.
2. Now add a line to `app.py`, but **don't commit yet**. Run `git diff` to see the unstaged change.
3. Stage `app.py` and run `git diff --staged` to confirm what's about to be committed.
4. Commit it with a deliberately **typo'd message** like `"feat: updat app entry pointt"`.
5. Use `git commit --amend` to fix the message to `"feat: update app entry point"`.
6. Run `git log --graph --decorate --all --oneline` to see your full clean history.
7. Now use `git reset --soft HEAD~1` (this means "one commit before HEAD") to uncommit the last commit while keeping the changes staged. Verify with `git status`.
8. Re-commit it cleanly.
9. Finally, use `git revert` on your very first commit and observe what happens to the log.

**Reflection question:** You've pushed a commit to a shared repository and a colleague has already pulled it and based new work on it. You realize that commit has a critical bug. Why would `git reset --hard` be dangerous here, and why is `git revert` the professional choice?

---


## 🏆 Module 3: Mini-Project

You're developing two features in parallel on your web app. Complete these tasks:

1. Starting from `main`, create and switch to a branch called `feature/product-search`. Add a new file `routes/search.py`, make two commits on it with realistic messages.
2. Switch back to `main` and make one new commit directly on it (edit `README.md` with a line about the project).
3. Merge `feature/product-search` into `main`. Observe whether it's a fast-forward or three-way merge and explain why based on the history shape.
4. Now create a second branch `feature/payment-gateway` from `main`. Edit `app.py` on this branch — change the first line to something specific.
5. Switch back to `main` and edit that **same first line** of `app.py` to something different. Commit it.
6. Merge `feature/payment-gateway` into `main` — you should get a conflict. Resolve it manually, stage the resolution, and complete the merge.
7. Run `git log --graph --decorate --all --oneline` and observe the full branching and merging history as a graph.

**Reflection question:** Your team is debating whether to always use `git merge --no-ff` to force merge commits even when a fast-forward is possible. What are the arguments for and against this policy? What information do you preserve or lose with each approach?

---