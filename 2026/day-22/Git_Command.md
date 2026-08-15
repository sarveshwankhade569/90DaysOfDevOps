# Day 22 – Git Introduction: Your First Repository

The goal here isn't to memorize commands — it's to understand the flow data takes through Git:

```
Working Directory → git add → Staging Area → git commit → Repository (.git)
```

Once that clicks, everything else is just vocabulary.

## Install & configure Git

Check if it's already installed:
```bash
git --version
```
If not:
```bash
sudo apt update
sudo apt install git -y
```

Every commit needs a name and email attached to it — set these once, globally:
```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

Check what's set:
```bash
git config --list
```

## Create and initialize a repository

```bash
mkdir devops-git-practice
cd devops-git-practice
git init
```

`git init` creates a hidden `.git/` folder — that's the actual database Git uses to store everything: history, config, references. You'll never need to touch it directly, just know it's there. If you ever delete it, your files stay but all Git history is gone.

Run `git status` right after `git init` and you'll see "no commits yet" — makes sense, since you haven't done anything yet.

## The three areas of Git

This is the concept the rest of Git builds on:

- **Working directory** — where you actually edit files.
- **Staging area** — where you put files you're about to commit (`git add` puts them here).
- **Repository** — the permanent, committed history (`git commit` puts them here).

```
Working Directory --git add--> Staging Area --git commit--> Repository (.git)
```

## Watching it happen

Create a file:
```bash
touch git-commands.md
git status
```
It'll show up as **untracked** — Git sees it, but isn't tracking changes to it yet.

Add some content to the file, then stage it:
```bash
git add git-commands.md
git status
```
Now it shows as "changes to be committed" — it's staged, but not committed yet.

Before committing, it's worth checking exactly what you're about to commit:
```bash
git diff --cached
```

Then commit it:
```bash
git commit -m "Add initial Git commands reference"
```
This creates a permanent snapshot. You'll get a short commit ID back, like `abc1234`.

Check your history:
```bash
git log            # full details
git log --oneline  # one line per commit — you'll use this one constantly
```

## Making more commits

Edit the file again — add a new section, save it — then:
```bash
git status         # see that it shows "modified"
git diff           # see exactly what changed, line by line
git add git-commands.md
git commit -m "Add branch commands to Git reference"
```

Repeat that a couple more times with different changes, and `git log --oneline` will show a growing stack of commits, each one a snapshot you can always come back to.

## The workflow to internalize

This is the loop you'll repeat constantly, in real DevOps work and everywhere else:

```bash
git status          # what changed?
git diff            # review the actual changes
git add .           # stage them
git diff --cached   # review what's staged, one last check
git commit -m "Describe what changed"
git log --oneline   # confirm it landed
```

## Command reference

| Command | What it does |
|---|---|
| `git config --global user.name "Name"` | Set your commit author name |
| `git config --global user.email "email"` | Set your commit email |
| `git init` | Create a new repository |
| `git status` | See what's changed / staged |
| `git add file` | Stage a file |
| `git add .` | Stage everything |
| `git diff` | Show unstaged changes |
| `git diff --cached` | Show staged changes |
| `git commit -m "message"` | Save staged changes as a snapshot |
| `git log` / `git log --oneline` | View commit history |
| `git show` | Show details of a specific commit |
| `git branch` | List branches |

## A few things worth actually understanding

- **`git add` vs `git commit`** — `add` stages a change (marks it as "ready"), `commit` actually saves it permanently to history. You can stage things, check them over, and un-stage them before they ever become part of the record.
- **`git diff` vs `git diff --cached`** — `git diff` shows changes you haven't staged yet; `git diff --cached` shows what's staged and about to be committed.
- **`rm -rf .git`** — deletes all of Git's tracking and history. Your actual files stay exactly as they are; you'd just lose the ability to see past versions or use any Git commands on that folder.
- **Commit messages matter** — `"Fix nginx health check"` tells you something six months from now; `"changes"` tells you nothing. Future-you (or a teammate) will thank you for being specific.

## Why this matters

Git isn't just "save code" — it gives you a controlled, searchable history of every change. That becomes genuinely valuable the day something breaks in production and you need to know exactly what changed, when, and why.
