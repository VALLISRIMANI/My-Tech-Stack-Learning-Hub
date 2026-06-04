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


## 🏆 Module 4: Mini-Project

You're cleaning up and organizing your web app repository professionally. Complete these tasks:

1. Create a proper `.gitignore` for your `portfolio-app` repo that ignores: Python cache files (`__pycache__/`, `*.pyc`), a virtual environment folder (`venv/`), `.env`, `.DS_Store`, and `*.log`. Commit it with an appropriate message.
2. Create a file called `.env` with content like `SECRET_KEY=abc123`. Verify that `git status` does not show it (confirming `.gitignore` works).
3. Simulate a context switch: make edits to two files without committing, then stash them with a descriptive message. Verify `git status` is clean after stashing, then pop the stash and confirm your changes are back.
4. Create an annotated tag `v0.1.0` on your earliest commit (use the hash) with the message `"Initial development checkpoint"`. Create another annotated tag `v1.0.0` on your latest commit with message `"First stable release"`. Run `git tag` to confirm both exist.
5. Add a function definition like `def connect_db():` to `db_config.py` and commit it. Then use `git grep` to find it, and use `git log -S` to find the exact commit that introduced it.

**Reflection question:** Your `.env` file was accidentally committed three months ago and has been in the repository ever since. You've now added it to `.gitignore` and run `git rm --cached .env`. Is your secret key safe now? What else would a security-conscious team need to do, and why?

---


## 🏆 Module 5: Mini-Project

You're taking your local `portfolio-app` online and setting up a real remote workflow. Complete these tasks:

1. Create a new repository on GitHub called `portfolio-app` (no README, no `.gitignore`).
2. Generate an SSH key pair if you don't have one, add the public key to GitHub, and test with `ssh -T git@github.com`.
3. Add the GitHub remote as `origin` and push your `main` branch with `git push -u origin main`. Verify the commits appear on GitHub's web interface.
4. Push your `v1.0.0` annotated tag to the remote. Confirm it appears under the **Tags** section on GitHub.
5. Create a new branch `feature/contact-page` locally, make two commits on it, and push it with the correct first-push command. Confirm the branch appears on GitHub.
6. Simulate a teammate's update: directly edit a file through GitHub's web interface (click any file → pencil icon → make a change → commit). Now on your local machine, run `git fetch origin`, then `git log main..origin/main --oneline` to see the incoming commit before merging it with `git merge origin/main`.

**Reflection question:** Your colleague says "I always just use `git pull` — it's faster and I've never had a problem." What specific scenario would you describe to them to illustrate why `git fetch` + review + `git merge` is a safer professional habit? What would go wrong in that scenario with a blind `git pull`?

---



## 🏆 Module 6: Mini-Project

You're cleaning up your feature branch history and doing some surgical Git work. Complete these tasks:

1. On a feature branch, make 5 commits — intentionally messy ones: a WIP commit, a "fix typo" commit, a debug commit, and two real feature commits. Run `git log --oneline` to see them.
2. Use `git rebase -i HEAD~5` to clean them up: squash the WIP and its fix into one clean commit, drop the debug commit, and reword one message. Run `git log --oneline` after to confirm the clean result.
3. Create a second branch from `main`. Make a bug fix commit on it. Cherry-pick that specific commit onto your feature branch. Confirm with `git log --oneline` that the fix appears on both branches.
4. Set up a bisect session: mark `HEAD` as bad and your very first commit as good. Walk through at least 3 bisect steps manually (marking good/bad), then `git bisect reset`.
5. Simulate the Golden Rule violation scenario in your mind: if you had already pushed your feature branch and a colleague had pulled it, what exactly would happen to their repository if you ran `git rebase main` on your branch and force-pushed? Write out the sequence of events in plain English.

**Reflection question:** You're about to open a pull request and your feature branch has 11 commits: 3 real feature commits, 4 "WIP" saves, 2 typo fixes, 1 debug print removal, and 1 merge commit from syncing with main. What interactive rebase strategy would you use to turn this into a clean, reviewable PR history? What would the ideal final commit structure look like?

---



## 🏆 Module 7: Mini-Project

You're running a full collaborative workflow on your project. Complete these tasks:

1. Fork any public repository on GitHub (a classmate's, or any small open source project). Clone your fork, and add the original as `upstream`. Run `git remote -v` to confirm both remotes.
2. Sync your fork with upstream using the fetch → merge → push cycle.
3. Create a feature branch on your fork, make two commits, and push it. Open a Pull Request on GitHub from your fork's branch into the original (or into your own fork's `main` if the original isn't yours). Write a proper PR description with a title, what it does, and a `Closes #1` reference (create a dummy issue first).
4. On your own `portfolio-app` repo, create three Issues: one bug report, one feature request, one task. Apply appropriate labels, assign yourself, and add them to a milestone called `v1.1.0`.
5. Create a GitHub Project board with at least four columns. Add your issues to it. Manually move one to "In Progress."
6. Open a PR on `portfolio-app` with `Closes #1` in the description. Merge it and observe that Issue #1 closes automatically.

**Reflection question:** Your team is debating between Squash and Merge vs Rebase and Merge as the standard PR merge strategy. You write detailed, meaningful individual commits on your branches. Your colleague writes messy WIP commits and cleans them up with interactive rebase before opening PRs. Which strategy works for both of you — and what discipline does each developer need to maintain for it to work well?

---



## 🏆 Module 8: Mini-Project

You're setting up a production-grade repository for your web app. Complete these tasks:

1. Choose a branching model for your `portfolio-app` and document it in `CONTRIBUTING.md` — explain which model you chose and why it fits your project's deployment style.
2. On GitHub, add branch protection to `main` with: required PR before merging, at least 1 required approval, and dismiss stale reviews on new commits.
3. Create `.github/workflows/ci.yml` with a workflow that triggers on push to `main` and on PRs to `main`, runs on `ubuntu-latest`, sets up Python 3.11, installs dependencies from `requirements.txt`, and runs `pytest tests/`.
4. Create a second job in the same workflow for linting with `flake8`. Make it run in parallel with the test job.
5. Add a `deploy` job that `needs` both lint and test to pass, runs only on push to `main` (not on PRs), and simply `echo`es "Deploying to production" as a placeholder.
6. Commit and push the workflow. Open a PR to `main` and observe the Actions tab — watch the jobs run in real time.
7. Go back to branch protection settings and add your CI job names as required status checks.

**Reflection question:** You've now completed the full Git and GitHub curriculum — from `git init` to CI/CD pipelines. Think about a real or hypothetical project you'd build. Which branching model would you use, and why? What branch protection rules would you enforce from day one? And what would your CI pipeline check on every PR — tests, linting, security scanning, something else? Describe the complete system you'd set up.

---