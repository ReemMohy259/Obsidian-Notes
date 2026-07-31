## Fundamentals

**1. What is Git, and how is it different from a centralized VCS like SVN?** Git is a **distributed version control system** — every developer has a full copy of the repository (all history) locally, not just the latest snapshot. This means you can commit, branch, and view history offline, and there's no single point of failure. Centralized systems (like SVN) rely on one central server; you need network access to commit and see history.

**2. What's the difference between Git and GitHub?** Git is the version control tool itself (runs locally, command-line based). GitHub is a **hosting platform** for Git repositories, adding features like pull requests, issue tracking, code review, and CI/CD integration. GitLab and Bitbucket are similar alternatives.

**3. What is a repository (repo)?** A folder tracked by Git, containing your project files plus a hidden `.git` directory that stores the entire history, branches, commits, and configuration.

**4. What's the difference between `git init` and `git clone`?**

- `git init` — creates a new, empty Git repository in the current folder.
- `git clone <url>` — copies an existing remote repository (all history, branches) to your local machine.

**5. What are the three main areas/states in Git?**

- **Working directory** — your actual files, where you make edits
- **Staging area (index)** — a holding zone for changes you're about to commit (`git add`)
- **Repository (.git)** — where committed snapshots are permanently stored (`git commit`)

---

## Everyday Commands

**6. What does `git status` do?** Shows the current state of the working directory and staging area — which files are modified, staged, or untracked.

**7. What's the difference between `git add`, `git commit`, and `git push`?**

- `git add <file>` — stages changes (moves them to the index, marking them ready to commit)
- `git commit -m "message"` — saves the staged changes as a new snapshot in the local repository history
- `git push` — uploads local commits to the remote repository (e.g., GitHub)

**8. What's the difference between `git pull` and `git fetch`?**

- `git fetch` — downloads new commits/branches from the remote but does **not** merge them into your current branch. Safe, lets you review changes first.
- `git pull` — does a `fetch` **and then merges** (or rebases, depending on config) the remote changes into your current branch automatically.

**9. What does `git diff` show?** The differences between your working directory and the last commit (or between any two commits/branches, depending on arguments). `git diff --staged` shows differences between staged changes and the last commit.

**10. What's the difference between `git log` and `git log --oneline`?** `git log` shows full commit details (hash, author, date, message) for history. `--oneline` condenses each commit to a single line (short hash + message) for a quicker overview.

**11. How do you undo changes in a file before committing?**

```bash
git restore <file>          # discard working directory changes (Git 2.23+)
git checkout -- <file>      # older equivalent
```

**12. How do you unstage a file (remove it from the staging area without losing changes)?**

```bash
git restore --staged <file>
# or older: git reset <file>
```

---

## Branching & Merging

**13. What is a branch in Git?** A lightweight, movable pointer to a specific commit. Branches let you work on features, fixes, or experiments in isolation without affecting the main codebase (`main`/`master`).

**14. How do you create and switch to a new branch?**

```bash
git branch feature-x        # create
git checkout feature-x      # switch
# or combined:
git checkout -b feature-x
# modern equivalent:
git switch -c feature-x
```

**15. What's the difference between `git merge` and `git rebase`?**

- **Merge**: combines two branches by creating a new "merge commit" that has two parents. Preserves full history exactly as it happened, but can create a messy, non-linear history.
- **Rebase**: replays your branch's commits on top of another branch's latest commit, creating a clean, **linear** history. Rewrites commit history (new commit hashes), so it should be avoided on shared/public branches (*Golden rule of rebase*).

**16. What is a merge conflict, and how do you resolve one?** A conflict happens when Git can't automatically combine changes — e.g., the same line was edited differently in two branches. Git marks the conflicting section in the file with `<<<<<<<`, `=======`, `>>>>>>>` markers. To resolve:

1. Open the file, manually edit to keep the correct content, remove the markers
2. `git add <file>` to mark it resolved
3. `git commit` (for a merge) or `git rebase --continue` (for a rebase)

**17. What's a fast-forward merge?** When the branch being merged in has no divergent commits from the target branch — Git simply moves the branch pointer forward, without creating a merge commit.

**18. What does `git branch -d` vs `git branch -D` do?**

- `-d` — deletes a branch, but only if it's already merged (safety check)
- `-D` — force-deletes a branch regardless of merge status

**19. How do you see which branch you're currently on?**

```bash
git branch      # lists branches, current one marked with *
git status      # also shows current branch at the top
```

---

## Undoing & History Rewriting

**20. What's the difference between `git reset` and `git revert`?**

- `git reset` — moves the branch pointer to a previous commit, optionally changing the working directory/staging area (`--soft`, `--mixed`, `--hard`). Rewrites history — risky on shared branches.
- `git revert` — creates a **new commit** that undoes the changes of a previous commit, without altering existing history. Safe for shared/public branches.

**21. What's the difference between `git reset --soft`, `--mixed`, and `--hard`?**

- `--soft` — moves HEAD to the target commit, keeps changes staged
- `--mixed` (default) — moves HEAD, unstages changes but keeps them in the working directory
- `--hard` — moves HEAD, and **discards** all changes in staging and working directory (destructive — use carefully)

**22. How do you amend the last commit?**

```bash
git commit --amend -m "new message"
```

Useful for fixing a typo in the last commit message or adding a forgotten file — but avoid doing this on commits already pushed/shared.

**23. What is `git stash` used for?** Temporarily saves uncommitted changes (working directory + staging) without committing, so you can switch branches cleanly. Restore later with `git stash pop` (applies and removes from stash list) or `git stash apply` (applies but keeps it in the list).

**24. How do you see the commit history for a specific file?**

```bash
git log -- <file>
git log -p -- <file>   # with diffs shown
```

**25. What's `git cherry-pick`?** Applies a specific commit from one branch onto another, without merging the whole branch — useful for pulling in a single bug fix without bringing in unrelated changes.

---

## Remote Repositories & Collaboration

**26. What is `origin` in Git?** The default name Git gives to the remote repository you cloned from (or added). It's just a convention/alias — you could name a remote anything.

**27. How do you add a remote repository?**

```bash
git remote add origin https://github.com/user/repo.git
```

**28. What is a Pull Request (PR) / Merge Request (MR)?** A request to merge changes from one branch (often a feature branch or fork) into another (often `main`), used for code review and discussion before merging. It's a platform feature (GitHub/GitLab), not a native Git command.

**29. What's the difference between forking and branching?**

- **Branching**: creating a new line of development within the _same_ repository — common when you have write access.
- **Forking**: creating your own **copy of the entire repository** under your account, typically used when you don't have write access to the original repo (e.g., open-source contributions).

**30. What's a `.gitignore` file for?** Specifies files/patterns Git should never track (e.g., build output, `node_modules/`, `.env` secrets, IDE config files) — keeps the repo clean and avoids committing sensitive or generated files.

**31. How do you resolve a "rejected push because remote contains work you don't have" error?** Someone else pushed changes since you last pulled. Fetch and integrate their changes first:

```bash
git pull            # or: git fetch && git rebase origin/main
git push
```

---

## Practical / Workflow Questions

**32. What Git branching strategy have you used or heard of (e.g., Git Flow, trunk-based development)?**

- **Git Flow**: long-lived `develop` and `main` branches, plus feature/release/hotfix branches — more structured, common in release-cycle-based projects.
- **Trunk-based development**: everyone commits frequently to a single main branch (or very short-lived feature branches merged quickly) — favored for CI/CD and fast-moving teams.

**33. How would you fix a mistake you already pushed to a shared branch?** Prefer `git revert` (creates a new commit undoing the change) rather than `git reset` + force-push, since force-pushing rewrites history other people may have already pulled — this can break their local repos and is generally considered unsafe on shared branches.

**34. What would you do if you accidentally committed a sensitive file (like a password or API key)?**

- Remove it from tracking (`git rm --cached <file>`) and add it to `.gitignore`
- If it was already pushed, it's not enough to just delete it in a new commit — the secret is still in history, so you'd need to rewrite history (`git filter-repo` or BFG Repo-Cleaner) **and rotate/invalidate the leaked credential immediately**, since anyone with repo access could have already seen it.

**35. What's the difference between `HEAD`, a branch, and a commit hash?**

- **Commit hash** — a unique SHA-1 identifier for a specific snapshot
- **Branch** — a movable pointer to a commit (the latest commit on that line of work)
- **HEAD** — a pointer to whatever you currently have checked out (usually points to a branch, which in turn points to a commit; can also point directly to a commit in "detached HEAD" state)

**36. What is a detached HEAD state?** When `HEAD` points directly to a commit instead of a branch (e.g., after `git checkout <commit-hash>`). Any new commits made here aren't attached to a branch, so they can be lost/orphaned unless you create a branch from that point before switching away.

**37. How do you squash multiple commits into one?** Interactive rebase:

```bash
git rebase -i HEAD~3
# then mark commits as "squash" or "s" in the editor that opens
```

Useful for cleaning up a messy feature branch's history before merging into `main`.

