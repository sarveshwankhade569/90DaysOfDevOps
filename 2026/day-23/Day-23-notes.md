# Day 23 – Git Branching & GitHub

## What a branch actually is

Picture your repo as a line of commits:
```
main:  A --- B --- C
```
When you create a branch, Git just adds a new pointer starting from wherever you are — your new commits go there instead of on `main`:
```
main:      A --- B --- C
                         \
feature-1:                D --- E
```
Nothing on `feature-1` touches `main` until you deliberately merge it. That's the whole point of branching — you can experiment, break things, and commit half-finished work without risking the stable code.

**HEAD** is just Git's way of tracking "which branch am I currently on." Run `git branch --show-current` and it'll tell you.

## Branch commands you'll actually use

| Command | What it does |
|---|---|
| `git branch` | List local branches |
| `git branch -a` | List local + remote branches |
| `git switch feature-1` | Switch to an existing branch |
| `git switch -c feature-1` | Create a branch and switch to it in one step |
| `git branch -d feature-1` | Delete a branch (only if it's merged) |
| `git branch -D feature-1` | Force-delete (careful — can lose unmerged work) |
| `git log --oneline --graph --all` | Visualize all branches at once |

You'll see older tutorials use `git checkout` to switch branches — that still works, but `checkout` also does file restoration and a few other things, which makes it easy to misuse. Modern Git split that up: use `git switch` for branches and `git restore` for files. Simpler to reason about.

## Seeing branches actually work

```bash
git switch main
git switch -c feature-1          # create + switch
echo "This is feature 1" > feature-1.txt
git add feature-1.txt
git commit -m "Add feature 1"
```

Now switch back to `main`:
```bash
git switch main
ls
```
`feature-1.txt` won't be there, and `git log --oneline` won't show your commit either. That's the proof — the file and the commit only exist on `feature-1`. Switch back and they reappear. This isolation is the entire reason branches are useful.

## Connecting to GitHub

Check if a remote is already set:
```bash
git remote -v
```
If not, create an empty repo on GitHub (don't let it auto-generate a README or `.gitignore` — your local repo already has content), then link it:
```bash
git remote add origin https://github.com/YOUR_USERNAME/devops-git-practice.git
```

`origin` is just a nickname for that URL, so you don't have to type it every time.

Push your branches:
```bash
git switch main
git push -u origin main

git switch feature-1
git push -u origin feature-1
```
The `-u` flag links your local branch to the remote one, so future pushes on that branch can just be `git push`.

## fetch vs pull

- **`git fetch`** downloads info about what's changed on the remote, but doesn't touch your working files. It's a safe way to check "did anything change?" before deciding what to do.
- **`git pull`** is `fetch` + `merge` combined — it pulls the changes down *and* merges them into your current branch immediately.

If you're not sure what a pull is about to do, `fetch` first and look around, then `pull` when you're confident.

## origin vs upstream (when working with forks)

- **`origin`** — your own GitHub repo (or your fork of someone else's).
- **`upstream`** — the original repo you forked from.

```bash
git remote add upstream https://github.com/ORIGINAL_OWNER/PROJECT.git
```

To keep your fork's `main` up to date with the original project:
```bash
git fetch upstream
git switch main
git merge upstream/main
git push origin main
```

## Clone vs Fork

- **Clone** copies a repo you already have access to, straight to your machine: `git clone <url>`.
- **Fork** makes *your own* GitHub copy of someone else's repo — used when you don't have write access and want to contribute anyway.

The typical open-source contribution flow:
```
fork the repo → clone your fork → create a branch → commit → push → open a Pull Request → maintainer reviews & merges
```

## The full feature workflow, start to finish

```bash
git switch -c feature-login
echo "Login feature is under development" > login.txt
git add login.txt
git commit -m "Add login feature"
git push -u origin feature-login
```

Then on GitHub, open a **Pull Request** — this is literally just a request: *"I have changes on feature-login, please review and merge them into main."*

Once it's approved and merged on GitHub, bring it back down locally:
```bash
git switch main
git pull origin main
```

## Cleaning up a finished branch

```bash
git branch -d feature-login              # delete locally
git push origin --delete feature-login   # delete on GitHub
```

## The mental model worth keeping

```
main ── feature branch ── commits ── push ── Pull Request ── review ── merge back into main
```

And with a fork in the picture:
```
upstream (original repo) → your fork (origin) → clone to your machine → branch → commit → push → PR → back to upstream
```

