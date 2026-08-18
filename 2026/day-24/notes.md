# Day 24 – Advanced Git: Merge, Rebase, Stash & Cherry-Pick

Five commands, five mental models — this is the day those click into place.

| Command | Simple meaning |
|---|---|
| `merge` | Combine two branches |
| `rebase` | Move your commits on top of another branch |
| `squash` | Combine multiple commits into one |
| `stash` | Temporarily save uncommitted work |
| `cherry-pick` | Copy one specific commit |

## Merge

The simplest way to bring a branch's work back into `main`:
```bash
git switch main
git merge feature-login
```

**If `main` hasn't moved since you branched off**, Git can just slide the `main` pointer forward to your latest commit — no extra commit needed. This is called a **fast-forward merge**, and it's the easiest case.

**If both branches have new commits**, Git can't just fast-forward — it creates a **merge commit** that ties both histories together. Nothing to worry about, that's just Git recording "these two branches came together here."

## Merge conflicts

A conflict happens when the same line was changed differently on both branches, and Git genuinely can't guess which version you want. You'll see something like this inside the file:

```
<<<<<<< HEAD
PORT=9090
=======
PORT=9000
>>>>>>> feature-config
```

Everything above `=======` is your current branch's version; everything below is the incoming branch's version. Pick the one you want (or blend them), delete the `<<<<<<<` / `=======` / `>>>>>>>` markers, then:
```bash
git add config.txt
git commit -m "Resolve config merge conflict"
```

If it's a mess and you want to back out entirely:
```bash
git merge --abort
```
That's a genuinely useful escape hatch — don't be afraid to use it.

## Rebase

Instead of merging two histories together, rebase **replays** your branch's commits on top of the latest `main`:
```bash
git switch feature-dashboard
git rebase main
```

The result is a straight line instead of a branching mess. The catch: rebase creates *new* commits (same changes, different IDs) because a commit's ID depends on its parent, and rebase changes the parent. So:

- **Rebase your own local, unpushed work** — totally safe, makes history cleaner.
- **Don't rebase a branch other people have already pulled** — their history will no longer match yours, and it gets messy fast.

**Merge vs rebase**, the short version: merge preserves what actually happened (including the mess); rebase pretends you built your feature cleanly on top of the latest `main`. Use merge when history matters, rebase when you just want a tidy line.

## Squash

Sometimes a feature branch has a pile of small, messy commits (`fix typo`, `fix typo again`, `oops`) that aren't useful individually. Squash merges them into one clean commit on `main`:
```bash
git switch main
git merge --squash feature-profile
git commit -m "Add profile feature"
```
All the changes land, but as a single logical commit instead of five scrappy ones.

## Stash

For when you're mid-change and suddenly need to drop everything:
```bash
git stash push -m "Profile work in progress"
```
Your working directory goes clean, so you can switch branches, fix whatever's urgent, then come back and restore your work:
```bash
git switch feature-profile
git stash pop
```

`git stash pop` restores your changes **and** deletes the stash. `git stash apply` restores them but **keeps** the stash around — useful if you want to apply the same stash to more than one branch.

Other handy stash commands:
```bash
git stash list           # see everything you've stashed
git stash show -p        # see what's actually in the latest stash
git stash drop stash@{0} # delete a specific one
```

## Cherry-pick

Sometimes you don't want an entire branch, just *one* commit from it — say, a critical fix buried in a feature branch with nine other unrelated commits:
```bash
git switch main
git cherry-pick abc222
```
Git applies just that commit's changes to your current branch. If it conflicts, same drill as a merge conflict — fix the file, `git add`, then `git cherry-pick --continue` (or `--abort` to cancel).

## Putting it together — a realistic scenario

You're deep in `feature-monitoring` with uncommitted changes when production monitoring breaks:

```bash
git stash push -m "Monitoring work in progress"
git switch main
git switch -c hotfix-monitoring
# fix the actual problem
git add monitoring.conf
git commit -m "Fix monitoring configuration"
git push -u origin hotfix-monitoring
# open a PR, get it merged
git switch feature-monitoring
git stash pop
```

That's the whole point of these tools — they let you context-switch without losing or committing half-finished work.

## Command reference

```bash
# Merge
git merge <branch>
git merge --squash <branch>
git merge --abort

# Rebase
git rebase main
git rebase --continue
git rebase --abort

# Stash
git stash push -m "message"
git stash list
git stash show -p
git stash apply
git stash pop
git stash drop stash@{0}

# Cherry-pick
git cherry-pick <commit>
git cherry-pick --continue
git cherry-pick --abort

# History
git log --oneline --graph --decorate --all
```

## The five things to actually remember

- **Merge** — bring two histories together.
- **Rebase** — put my commits on top of the latest base.
- **Squash** — turn many small commits into one logical one.
- **Stash** — save my unfinished work temporarily.
- **Cherry-pick** — bring this one specific commit here.
