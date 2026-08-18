# Day 25 – Git Reset vs Revert & Branching Strategies

Two big ideas today: **undoing mistakes safely**, and **how teams organize their branches**.

## Git Reset — moving the pointer backward

Think of `reset` as rewinding time on your branch — it moves your branch pointer back to an earlier commit. The three flavors differ in what happens to your changes when you rewind.

```bash
git reset --soft HEAD~1
```
Rewinds the commit but keeps everything staged. Your changes sit in the staging area, ready to re-commit differently. Nothing is lost — the gentlest option.

```bash
git reset --mixed HEAD~1
```
Rewinds the commit **and** un-stages the changes — but they still exist in your working directory (your files). This is actually the default if you just type `git reset HEAD~1`.

```bash
git reset --hard HEAD~1
```
Rewinds the commit and throws away the changes completely. Your files go back to exactly how they looked at that earlier commit. This one is destructive — if you didn't commit or stash it somewhere first, it's gone.

**Simple way to remember it:**
- `--soft` = "undo the commit, keep everything else"
- `--mixed` = "undo the commit and un-stage, but let me still see the changes"
- `--hard` = "nuke it, clean slate"

⚠️ Never use `reset` on commits you've already pushed and someone else might have pulled. You're rewriting history — their copy of the branch will now disagree with yours. Reset is for your own local, unpushed mistakes.

## Git Revert — undoing without rewriting history

```bash
git revert <commit-hash>
```
Instead of erasing a commit, `revert` creates a **new commit that undoes the changes** from the one you're reverting. The original commit stays in history — you just add a new one on top that cancels it out.

If the branch is shared (pushed, others working off it), `revert` is the safe choice. Nobody's history gets rewritten or thrown out of sync — you're just adding a normal new commit, like any other.

## Reset vs Revert, side by side

| | `git reset` | `git revert` |
|---|---|---|
| What it does | Moves your branch pointer backward | Adds a new commit that undoes an old one |
| Removes commit from history? | Yes (with `--hard`, gone entirely) | No — original commit stays visible |
| Safe on shared/pushed branches? | ❌ No | ✅ Yes |
| When to use | Undoing your own local, not-yet-pushed mistakes | Undoing something already pushed/shared |

**Rule of thumb:** nobody's seen the commit yet → `reset` is fine. Already out in the world → use `revert`.

## Branching strategies — how teams organize work

### GitFlow
Multiple long-lived branches with specific jobs: `develop` (ongoing work), `feature/*` (new features), `release/*` (prepping a version), `hotfix/*` (urgent production fixes).

```
main ─────────────●───────────●──   (production releases)
                    \         /
develop ──●──●──●──●─●──●──●──●──   (integration branch)
           \        /
        feature/x
```

**Used by:** larger teams with scheduled, versioned releases — enterprise software, mobile apps with app-store review cycles.
**Pros:** very structured, clear separation between "in progress" and "stable."
**Cons:** heavy — a lot of branches and merge overhead for a small team.

### GitHub Flow
Much simpler: just `main` plus short-lived feature branches. Branch off, commit, open a PR, get it reviewed, merge straight back into `main`. `main` is always deployable.

```
main ──●──────●───────●──
        \    / \      /
      feature-a  feature-b
```

**Used by:** most modern web/SaaS teams doing continuous deployment.
**Pros:** simple, fast, easy to understand.
**Cons:** doesn't handle scheduled releases well — everything ships as it merges.

### Trunk-Based Development
Everyone commits directly to `main` (the "trunk") very frequently, using short-lived branches (hours, not weeks) or feature flags to hide unfinished work.

```
main ──●●●●●●●●●●●●●●●●──   (constant small commits, feature flags hide WIP)
```

**Used by:** large tech companies with strong CI/CD and automated testing.
**Pros:** avoids painful merge conflicts from long-lived branches, forces small incremental changes.
**Cons:** needs excellent test automation and discipline — risky without it.

### Which one to use

- **Startup shipping fast** → GitHub Flow. Minimal ceremony, ship as you merge.
- **Large team, scheduled releases** → GitFlow. The structure pays off when coordinating many people and release dates.
- **Your favorite open-source repo** → worth checking yourself — look at its `CONTRIBUTING.md` or branch list. Most modern open-source JS/Python projects lean GitHub Flow style (a `main` branch + PRs); some older or enterprise-backed projects still use GitFlow-style `develop` branches.

## Commands worth remembering

```bash
git reset --soft HEAD~1     # undo commit, keep changes staged
git reset --mixed HEAD~1    # undo commit, keep changes unstaged
git reset --hard HEAD~1     # undo commit, discard changes entirely
git revert <commit-hash>    # safely undo a commit with a new commit
git log --oneline           # see commit hashes to reset/revert to
```
