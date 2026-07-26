[Atlassian Course](https://www.atlassian.com/git/tutorials/saving-changes/git-stash)

# States
* Untracked
* Staged
* Committed
* Modified
# Merge
- Non-Destructive Action
- Merge Commit
- Fast forward merge
- 3-way merge
- Merge conflict

# Rebase
* Used with caution
* Re-write project history
* No extra merge commits -> (Brand new commits)
* Linear history
* Golden rule of rebasing
* Interactive rebase 

# Stash
- Local shelves
- Re-applying your stashed changes -> `git stash pop` or `git stash apply`
- WIP -> Working in progress
- Good practice to annotate your stashes with a description, using `git stash save "message"`
- Partial stashes -> iterate over all changes **Hunk** and decide whether to stash or not
![[Pasted image 20260617003359.png|449]]
# Cherry-Pick
- Cherry picking is the act of picking a commit from a branch and applying it to another
* Not recommended but useful in some cases (team collaboration like backend and frontend, bug hotfixes, undoing changes)

```
a - b - c - d   Main
         \
           e - f - g Feature
           
>> git cherry-pick commitSha(f)

a - b - c - d - f   Main
         \
           e - f - g Feature
```

# Commands
* `git clean` **worked with untracked files** and `git reset` **worked with tracked files**
* `git fetch` and `git pull`

| Command        | Scope        | Common use cases                                                                                                      |
| -------------- | ------------ | --------------------------------------------------------------------------------------------------------------------- |
| `git reset`    | Commit-level | Discard commits in a **private branch** or throw away uncommitted changes, introduce orphan commits (eligible for GC) |
| `git reset`    | File-level   | Unstage a file                                                                                                        |
| `git checkout` | Commit-level | Switch between branches or inspect old snapshots                                                                      |
| `git checkout` | File-level   | Discard changes in the working directory                                                                              |
| `git revert`   | Commit-level | Undo commits in a **public branch**, produce **new commit** that has inverse content of the previous commit           |
| `git revert`   | File-level   | (N/A)                                                                                                                 |

# Reset vs Rvert
- Revert = Undo with a new commit (history stays).
- Reset = Go back in time (history changes).

| Git Revert                           | Git Reset                                   |
| ------------------------------------ | ------------------------------------------- |
| Creates a new commit to undo changes | Removes or rewrites commit history          |
| Safe for shared repositories         | Can be risky for shared repositories        |
| Keeps history intact                 | Changes history                             |
| Can undo any individual commit       | Only moves backward from the current commit |

**Reset Types**
```
Working Directory  -->  Staging Area  -->  Repository
      (files)            (git add)         (git commit)

--soft   : Repository only
--mixed  : Repository + Staging
--hard   : Repository + Staging + Working Directory
```

# GitHub Actions
* Pipeline
* CI -> Continuous Integration
* CD -> Continuous Delivery

# SVN vs GIT
- SVN (Subversion) is a Centralized Version Control System (CVCS). There is one central repository on a server.
* Git is a Distributed Version Control System (DVCS). Every developer has a complete copy of the repository, including its history.

| Feature            | SVN               | Git                   |
| ------------------ | ----------------- | --------------------- |
| Architecture       | Centralized       | Distributed           |
| Local commits      | ❌ No              | ✅ Yes                 |
| Branching          | Slower, heavier   | Fast and lightweight  |
| Offline work       | Limited           | Fully supported       |
| Repository history | Stored on server  | Stored on every clone |
| Performance        | Depends on server | Mostly local and fast |
