📍 **Module 1 of 8: Version Control Basics & Local Git Setup**

---

## 🧠 What is Version Control, and Why Does Git Exist?

Imagine you're writing a complex Python application. Over weeks, you make hundreds of changes. One day you introduce a bug — but you're not sure when. Without version control, you're staring at your current broken code with no way to go back.

Version control solves this. It's a system that **records changes to files over time**, so you can recall specific versions later, understand what changed and why, and collaborate without stepping on each other's work.

---

### CVCS vs DVCS — Two Generations of Version Control

**Centralized Version Control Systems (CVCS)** — like SVN or CVS — work like a library. There's one central server holding the "real" history. Everyone checks files out, edits them, and checks them back in. The problem? If the server goes down, nobody can commit, branch, or even see history. The whole team is blocked.

**Distributed Version Control Systems (DVCS)** — like Git — work differently. Every developer has a **full copy of the entire repository**, including all history, locally on their machine. There's no single point of failure. You can commit, branch, diff, and time-travel entirely offline. The remote server (like GitHub) is just a convenient shared meeting point — not the source of truth.

Git was created by **Linus Torvalds in 2005** to manage the Linux kernel source code after the team lost access to their previous VCS tool. He needed something fast, distributed, and able to handle thousands of contributors. Git was built in about two weeks and has dominated the industry ever since.

---

## 🏗️ Core Git Architecture — The Three Areas

Before you run a single command, you need to understand the mental model that underpins *everything* in Git. There are **three distinct areas** where your code lives at any moment:

```
┌─────────────────────┐     git add      ┌──────────────────┐     git commit     ┌──────────────────────┐
│   Working Directory │ ───────────────► │  Staging Area    │ ─────────────────► │  Local Repository    │
│  (your actual files)│                  │  (the index)     │                    │  (.git folder)       │
└─────────────────────┘                  └──────────────────┘                    └──────────────────────┘
```

**Working Directory** — This is your project folder as you see it in your file explorer or editor. When you edit `app.py`, that change lives here. Git knows about it, but hasn't done anything with it yet.

**Staging Area (Index)** — Think of this as a **loading dock**. Before you commit, you deliberately choose which changes from your working directory to "stage." This lets you commit only part of your changes — for example, stage the fix to `routes/login.py` without staging the half-finished feature in `routes/payments.py`. This is one of Git's most powerful and misunderstood features.

**Local Repository** — The `.git` folder inside your project. This is where Git stores the full history of every commit ever made. When you run `git commit`, your staged snapshot gets permanently recorded here with a unique ID (a SHA-1 hash like `a3f2c91`).

> 💡 **Analogy:** Writing a photograph. Working Directory = what you're looking at. Staging Area = what you've framed in the viewfinder. Repository = the printed photo in the album, permanent.

**GOTCHA:** Beginners often think `git add` *saves* or *uploads* their work. It doesn't. It just moves changes to the staging area. Nothing is permanently recorded until `git commit`.

---

## ⚙️ Installation & Configuration

### Installation

- **Linux (Debian/Ubuntu):**
```
$ sudo apt install git
```
- **macOS:**
```
$ brew install git
```
- **Windows:** Download from git-scm.com — Git Bash is included.

Verify it worked:
```
$ git --version
git version 2.44.0
```

---

### Configuration — `git config`

Before your first commit, Git needs to know who you are. Every commit you make is stamped with your name and email — this is how teams know who changed what.

Git has three config levels:

| Level | Flag | Scope | Stored In |
|---|---|---|---|
| System | `--system` | All users on the machine | `/etc/gitconfig` |
| Global | `--global` | Your user account | `~/.gitconfig` |
| Local | `--local` | This repo only | `.git/config` |

Local overrides Global overrides System.

**Setting your identity (do this once globally):**

```
$ git config --global user.name "Alex Rivera"
$ git config --global user.email "alex@devteam.com"
```

**Set your preferred editor** (used when writing commit messages):

```
$ git config --global core.editor "code --wait"
```
This sets VS Code. Use `"vim"` or `"nano"` if you prefer terminal editors.

**Verify your config:**

```
$ git config --list
user.name=Alex Rivera
user.email=alex@devteam.com
core.editor=code --wait
```

**GOTCHA:** If you skip the `--global` flag, the setting only applies to the current repository. New developers often wonder why their name appears correctly in one project and not another — usually it's a missing `--global`.

---

Now that your identity is configured and you understand the three-area architecture, let's put it all together and actually create a repository.

---

## 🚀 Initializing a Repository — `git init`, `git add`, `git commit`

### `git init` — Creating a New Repository

```
$ mkdir web-app && cd web-app
$ git init
Initialized empty Git repository in /home/alex/web-app/.git/
```

This creates a hidden `.git/` folder inside `web-app/`. That folder *is* the repository — it holds all the history, configuration, and internal Git objects. Your actual project files live alongside it, not inside it.

**GOTCHA:** Never manually edit or delete anything inside `.git/`. You'll corrupt the repository.

---

### Your First Files

Let's create some realistic project files:

```
$ touch app.py requirements.txt README.md
```

Now check what Git sees:

```
$ git status
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	app.py
	README.md
	requirements.txt

nothing added to commit but untracked files present
```

Git sees the files but they're **untracked** — they exist in your Working Directory but Git isn't recording changes to them yet.

---

### `git add` — Moving to the Staging Area

Stage all three files:

```
$ git add app.py requirements.txt README.md
```

Or use `.` to stage everything in the current directory:

```
$ git add .
```

Check status again:

```
$ git status
On branch main

No commits yet

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   app.py
	new file:   README.md
	new file:   requirements.txt
```

They've moved from "Untracked" to "Changes to be committed" — they're on the loading dock, ready to be committed.

---

### `git commit` — Recording to the Repository

```
$ git commit -m "feat: initial project scaffold with app and requirements"
[main (root-commit) e7a3d12] feat: initial project scaffold with app and requirements
 3 files changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 app.py
 create mode 100644 README.md
 create mode 100644 requirements.txt
```

Let's unpack that output:
- `main` — the branch you committed to
- `root-commit` — this is the very first commit (no parent)
- `e7a3d12` — the short SHA-1 hash uniquely identifying this commit forever
- `3 files changed` — the diff summary

**The `-m` flag** lets you write the message inline. Without it, Git opens your configured editor so you can write a longer message.

---

### Seeing the full flow in action

Now let's modify a file and go through the whole cycle again:

```
$ echo "Flask==3.0.0" >> requirements.txt
$ git status
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
	modified:   requirements.txt
```

Stage it, then commit:

```
$ git add requirements.txt
$ git commit -m "chore: pin Flask version in requirements"
[main 4bc91f0] chore: pin Flask version in requirements
 1 file changed, 1 insertion(+)
```

You've now completed your first full **edit → stage → commit** cycle.

**GOTCHA:** A very common beginner mistake is editing a file, running `git commit`, and wondering why the change wasn't saved. You must `git add` *after* every edit — the staging area snapshots the file at the moment you ran `git add`, not at commit time. If you edit a file, stage it, edit it again, and then commit — only the *first* edit is committed.

---

✅ **Quick Check:** What is the difference between `git add` and `git commit`? Describe what happens to a file's location (which of the three areas it moves through) when you run each command.
