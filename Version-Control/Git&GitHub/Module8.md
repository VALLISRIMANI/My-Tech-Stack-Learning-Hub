📍 **Module 8 of 8: Production-Grade Workflows & GitHub Actions (CI/CD)**

---

## 🌊 Branching Models — GitFlow vs GitHub Flow

### Why Branching Models Exist

Individual Git commands don't tell you *how to organize* your work across a team and a release cycle. A branching model is a **agreed-upon set of conventions** — which branches exist, what they mean, who merges into what, and when. Without one, teams create chaos: people commit directly to `main`, features land half-finished, hotfixes get lost.

Two models dominate the industry. They suit very different types of projects.

---

### GitFlow — The Release-Oriented Model

GitFlow was defined by Vincent Driessen in 2010. It's designed for projects with **scheduled, versioned releases** — software that ships in discrete versions like `v1.2.0`, `v2.0.0`. Think: mobile apps, packaged software, APIs with versioned endpoints.

**The permanent branches:**

```
main      — production-ready code only. Every commit here is a release.
develop   — integration branch. Features merge here first, not main.
```

**The temporary branch types:**

```
feature/*    — branches off develop, merges back into develop
release/*    — branches off develop when ready to ship, merges into main AND develop
hotfix/*     — branches off main for urgent production fixes, merges into main AND develop
```

**The full GitFlow lifecycle:**

```
                    ┌─────────────────────────────────────────────┐
main                │ v1.0 ──────────────────────── v1.0.1 ── v1.1│
                    │  ↑                              ↑        ↑   │
                    │  │    ┌──── hotfix/payment ─────┘        │   │
develop             │  └────┴──────────────────────────────────┘   │
                    │       ↑              ↑                        │
feature/user-auth ──┘       │              │                        │
feature/search ─────────────┘              │                        │
release/v1.1 ──────────────────────────────┘                        │
                    └─────────────────────────────────────────────┘
```

**The workflow in practice:**

```
# Starting a new feature
$ git switch -c feature/user-auth develop    # branch from develop

# Finishing a feature
$ git switch develop
$ git merge --no-ff feature/user-auth
$ git branch -d feature/user-auth

# Starting a release (feature freeze — no new features after this)
$ git switch -c release/v1.1.0 develop      # branch from develop
# only bug fixes committed here
$ git switch main
$ git merge --no-ff release/v1.1.0
$ git tag -a v1.1.0 -m "Release v1.1.0"
$ git switch develop
$ git merge --no-ff release/v1.1.0          # sync fixes back to develop
$ git branch -d release/v1.1.0

# Emergency hotfix on production
$ git switch -c hotfix/payment-crash main   # branch from main, NOT develop
$ git add routes/payments.py
$ git commit -m "fix: prevent crash on empty payment method"
$ git switch main
$ git merge --no-ff hotfix/payment-crash
$ git tag -a v1.0.1 -m "Hotfix v1.0.1"
$ git switch develop
$ git merge --no-ff hotfix/payment-crash    # critical: sync fix to develop too
$ git branch -d hotfix/payment-crash
```

**GOTCHA:** The most dangerous GitFlow mistake is forgetting to merge a hotfix back into `develop`. You fix a production crash, tag `v1.0.1`, ship it — and three months later when `v1.2.0` is released from `develop`, the crash is back because `develop` never got the fix.

---

### GitHub Flow — The Continuous Delivery Model

GitHub Flow was defined by GitHub in 2011 as a deliberate simplification of GitFlow. It's designed for teams that **deploy continuously** — web applications, SaaS products, services where `main` is always live in production and you ship multiple times per day.

**The entire model in one diagram:**

```
main ──────────────────────────────────────────── (always deployable)
        ↑          ↑           ↑
feature/search   fix/tax   feature/auth
```

**The rules:**

1. `main` is always deployable — every commit on `main` is production-ready
2. All work happens on descriptive feature branches off `main`
3. Push your branch and open a PR early (Draft PR if not ready)
4. After review and CI passes, merge into `main`
5. Deploy immediately after merging

**The workflow:**

```
$ git switch -c feature/product-search
# work, commit, push
$ git push -u origin feature/product-search
# open PR, get review, CI passes
# merge PR → deploy to production
$ git branch -d feature/product-search
```

That's the entire model. No `develop`, no `release` branches, no hotfix branches. A bug in production is just another feature branch.

---

### GitFlow vs GitHub Flow — When to Use Each

| Factor | GitFlow | GitHub Flow |
|---|---|---|
| Release cadence | Scheduled versions (weekly, monthly) | Continuous (multiple times daily) |
| Team size | Larger, multiple parallel releases | Any size, single release track |
| Product type | Packaged software, versioned APIs, mobile apps | Web apps, SaaS, internal tools |
| Complexity | Higher — more branch types to manage | Lower — just feature branches |
| Hotfix handling | Dedicated hotfix branch type | Just another branch from main |
| Parallel versions | ✅ Supports maintaining v1.x and v2.x simultaneously | ❌ Assumes one production version |

**GOTCHA:** Many teams cargo-cult GitFlow for web applications that ship continuously — they end up with a `develop` branch that serves no real purpose other than adding an extra merge step. If you deploy continuously, use GitHub Flow. GitFlow's complexity is only justified when you genuinely need to maintain multiple release versions simultaneously.

---

Now that you have a branching model, you need to enforce it. Branch protection rules are how you make the model mandatory rather than optional.

---

## 🔒 Branch Protection Rules

### Why They Exist

Left unprotected, any collaborator with push access can:
- Force-push directly to `main` and rewrite history
- Push broken code that fails all tests
- Merge their own PR without any review
- Delete `main` entirely

Branch protection rules let repository admins **enforce quality gates** at the GitHub level — not just team culture.

---

### Configuring Branch Protection

In GitHub: Repository → Settings → Branches → Add branch protection rule.

Set the branch name pattern — `main` protects exactly `main`. `release/*` protects all release branches.

---

### Key Protection Options

**Require a pull request before merging**

Nobody can push directly to `main`. All changes must arrive via PR. This is the foundational rule.

```
☑ Require a pull request before merging
  ☑ Require approvals: 2         ← number of required approving reviews
  ☑ Dismiss stale pull request approvals when new commits are pushed
  ☑ Require review from Code Owners
```

`Dismiss stale approvals` is important — without it, someone could get 2 approvals on clean code, then push malicious commits after approval. The approvals auto-dismiss when new commits are pushed.

**CODEOWNERS file** — a `.github/CODEOWNERS` file maps file paths to required reviewers:

```
# .github/CODEOWNERS

# Global owners — review required on any PR
*                   @alexrivera

# Payment routes require the payments team
routes/payments.py  @payments-team

# Infrastructure files require the DevOps lead
*.yml               @devops-lead
Dockerfile          @devops-lead
```

If a PR touches `routes/payments.py`, GitHub automatically adds `@payments-team` as a required reviewer.

---

**Require status checks to pass before merging**

This connects CI/CD (GitHub Actions) to branch protection. You specify which automated checks must pass:

```
☑ Require status checks to pass before merging
  ☑ Require branches to be up to date before merging
  Required checks:
    ✓ test (pytest)
    ✓ lint (flake8)
    ✓ security-scan
```

If any of these checks fail, GitHub blocks the Merge button — even if all humans approved the PR. This is how you guarantee that broken code physically cannot reach `main`.

---

**Block force pushes**

```
☑ Do not allow force pushes
```

Prevents `git push --force` on the protected branch. Protects against accidental or intentional history rewriting.

---

**Require linear history**

```
☑ Require linear history
```

Forces all PRs to use Squash or Rebase merge strategy — no merge commits allowed on `main`. Enforces the linear history preference at the repo level.

---

**GOTCHA:** Repository admins can bypass protection rules by default. For truly critical branches, enable **"Include administrators"** to enforce the rules on everyone including yourself. This prevents the "I'll just force-push this one time" exception that gradually erodes team discipline.

**GOTCHA:** Branch protection rules require a GitHub Pro, Team, or Enterprise account for private repositories. They work on all public repositories for free.

---

With protection rules in place, the enforcement mechanism is automated checks — which brings us to GitHub Actions.

---

## ⚙️ Introduction to GitHub Actions — CI/CD Automation

### What CI/CD Means

**Continuous Integration (CI):** Every time code is pushed or a PR is opened, an automated pipeline runs tests, linters, and security scans. You get immediate feedback — before a human even looks at the PR.

**Continuous Delivery/Deployment (CD):** When code merges to `main`, an automated pipeline deploys it to staging or production.

Before CI/CD existed, teams had a dedicated person whose job was running tests and managing deployments. Now it's automated, instant, and consistent.

---

### How GitHub Actions Works

GitHub Actions is GitHub's built-in CI/CD platform. You define **workflows** in YAML files stored in `.github/workflows/` in your repository. GitHub executes them automatically on events you specify.

**Core concepts:**

```
Workflow      — a YAML file defining automation (.github/workflows/ci.yml)
Event         — what triggers the workflow (push, pull_request, schedule)
Job           — a group of steps that run on one virtual machine
Step          — a single task within a job (run a command or use an action)
Action        — a reusable step from the GitHub Marketplace
Runner        — the virtual machine that executes the job (ubuntu-latest, etc.)
```

---

### Your First Workflow — CI for a Python App

Create the file:
```
$ mkdir -p .github/workflows
$ touch .github/workflows/ci.yml
```

```yaml
# .github/workflows/ci.yml

name: CI Pipeline

# Trigger: run on every push and every PR targeting main
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest       # GitHub-hosted runner — clean Ubuntu VM

    steps:
      # Step 1: Check out the repository code onto the runner
      - name: Checkout code
        uses: actions/checkout@v4

      # Step 2: Set up the Python version
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      # Step 3: Install dependencies
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      # Step 4: Run linter
      - name: Lint with flake8
        run: |
          pip install flake8
          flake8 . --max-line-length=88 --exclude=venv/

      # Step 5: Run tests
      - name: Run pytest
        run: |
          pip install pytest
          pytest tests/ -v
```

Every field explained:

- `name:` — human-readable label shown in GitHub UI
- `on:` — the event triggers. `push` to `main`/`develop` and any PR targeting `main`
- `jobs:` — one or more jobs. Each runs on its own fresh VM
- `runs-on: ubuntu-latest` — use GitHub's free Ubuntu runner
- `uses: actions/checkout@v4` — official action that clones your repo onto the runner. Without this, the runner has no code
- `uses: actions/setup-python@v5` — official action to install Python
- `with:` — parameters passed to an action
- `run:` — execute shell commands directly. `|` allows multi-line

---

### Adding a Separate Lint Job

Jobs can run in **parallel** by default — they don't wait for each other. This speeds up feedback:

```yaml
jobs:
  lint:
    name: Lint Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Run flake8
        run: |
          pip install flake8
          flake8 . --max-line-length=88

  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Install and test
        run: |
          pip install -r requirements.txt pytest
          pytest tests/ -v
```

Both jobs start simultaneously. If lint fails, you know immediately — you don't wait for tests to finish first.

---

### Job Dependencies — `needs`

Sometimes jobs must run in sequence. A deployment should only happen after tests pass:

```yaml
jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: |
          pip install -r requirements.txt pytest
          pytest tests/ -v

  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: test                  # only runs if 'test' job passes
    if: github.ref == 'refs/heads/main'   # only on pushes to main, not PRs
    steps:
      - uses: actions/checkout@v4
      - name: Deploy application
        run: |
          echo "Deploying to production server..."
          # your actual deploy commands here
          # e.g. SSH to server, pull latest, restart service
```

`needs: test` — `deploy` waits for `test` to succeed. If tests fail, deploy never runs.

`if: github.ref == 'refs/heads/main'` — this condition restricts deploy to only fire on pushes to `main`, not on PRs. You don't want to deploy every time someone opens a PR.

---

### Using Secrets in Workflows

Deployment often requires credentials — SSH keys, API tokens, cloud provider keys. **Never hardcode these in your YAML file** — it's a public commit.

GitHub Secrets: Repository → Settings → Secrets and variables → Actions → New repository secret.

Add a secret named `DEPLOY_SSH_KEY`. Use it in your workflow:

```yaml
      - name: Deploy via SSH
        env:
          SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
          DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
        run: |
          echo "$SSH_KEY" > deploy_key
          chmod 600 deploy_key
          ssh -i deploy_key user@$DEPLOY_HOST "cd /app && git pull && restart_app"
```

`${{ secrets.SECRET_NAME }}` — the secrets syntax. GitHub masks these values in log output — they never appear in plain text even in the Actions run logs.

---

### Matrix Strategy — Testing Across Multiple Versions

Test your app against multiple Python versions simultaneously:

```yaml
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: |
          pip install -r requirements.txt pytest
          pytest tests/ -v
```

This creates **three parallel jobs** — one per Python version. If your app breaks on 3.12 but works on 3.11, you catch it immediately.

---

### Connecting Actions to Branch Protection

Going back to branch protection — now you can connect your CI jobs as required status checks:

```
Settings → Branches → main → Edit protection rule
☑ Require status checks to pass before merging
  Required checks:
    Run Tests (test)
    Lint Check (lint)
```

The status check names match your job `name:` fields in the YAML. Now the merge button is physically blocked until both jobs show green. No human can accidentally merge a PR that breaks tests.

---

### The Complete Production Pipeline

Here is the full workflow YAML combining everything:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  lint:
    name: Lint Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install flake8 && flake8 . --max-line-length=88

  test:
    name: Run Tests
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: |
          pip install -r requirements.txt pytest
          pytest tests/ -v

  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [lint, test]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
        run: echo "Deploying to production..."
```

The complete flow this creates:

```
Developer opens PR
       ↓
GitHub Actions triggers: lint + test run in parallel
       ↓
Both pass → PR shows green checkmarks
       ↓
Required reviewers approve
       ↓
Merge button unlocks
       ↓
Developer merges PR
       ↓
Push to main triggers pipeline again: lint + test + deploy
       ↓
Production updated automatically
```

**GOTCHA:** GitHub Actions minutes are free for public repositories and have a monthly free tier for private repositories (2,000 minutes/month on free plan). Each job consumes minutes separately — a matrix of 3 Python versions counts as 3 × job duration. Be mindful of this on private repos with heavy CI.

**GOTCHA:** The `actions/checkout@v4` step is required in almost every job. Without it, the runner's filesystem is completely empty — it has no idea what your code looks like. Forgetting this step is a very common first-timer mistake.

---


✅ **Quick Check:** A developer on your team pushes directly to `main`, bypassing the PR process. Three questions: (1) Which branch protection rule prevents this? (2) Even if that rule wasn't set, which GitHub Actions condition would prevent a broken deployment? (3) If a bug somehow reaches production anyway, which Git command from Module 6 would you use to find the exact commit that introduced it — and how does it work?

---

## 🎓 Course Complete — Full Curriculum Summary

You've covered the complete Git and GitHub curriculum:

| Module | Topics Mastered |
|---|---|
| 1 | Version control theory, three-area architecture, init, add, commit |
| 2 | git log, git diff, restore, amend, reset (soft/mixed/hard), revert |
| 3 | Branch pointers, switch, merge (fast-forward vs three-way), conflicts |
| 4 | .gitignore, stash, annotated tags, git grep, pickaxe (-S) |
| 5 | SSH keys, remote add, push, clone, fetch vs pull, tracking branches |
| 6 | Rebase, Golden Rule, interactive rebase (-i), cherry-pick, bisect |
| 7 | Forking, upstream sync, pull requests, merge strategies, Issues, Projects |
| 8 | GitFlow vs GitHub Flow, branch protection, GitHub Actions CI/CD |

---
