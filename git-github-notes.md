# Git & GitHub — Day-to-Day Reference Notes

A practical, working reference for everyday Git/GitHub usage: commands, common errors and their fixes, PR review workflows, and general best practices.

---

## 1. Setup & Configuration

```bash
# Identity (required before first commit)
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Useful defaults
git config --global init.defaultBranch main
git config --global core.editor "code --wait"      # use VS Code as commit editor
git config --global pull.rebase false               # merge by default on pull
git config --global credential.helper store          # cache credentials

# View config
git config --list
git config user.name
```

**SSH setup for GitHub (recommended over HTTPS):**
```bash
ssh-keygen -t ed25519 -C "you@example.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
# Copy ~/.ssh/id_ed25519.pub into GitHub > Settings > SSH Keys
ssh -T git@github.com   # test connection
```

---

## 2. Everyday Workflow

```bash
git clone git@github.com:org/repo.git       # clone via SSH
git status                                   # what's changed
git add file.txt                             # stage one file
git add .                                    # stage everything
git add -p                                   # stage interactively, hunk by hunk
git commit -m "message"                      # commit staged changes
git commit -am "message"                     # stage tracked files + commit (skips new files)
git push                                     # push current branch
git push -u origin branch-name               # push + set upstream (first push of a branch)
git pull                                     # fetch + merge/rebase
git fetch                                    # fetch without merging
git log --oneline --graph --all              # compact visual history
git diff                                     # unstaged changes
git diff --staged                            # staged changes
```

### Branching
```bash
git branch                                   # list local branches
git branch -a                                # list all (local + remote)
git checkout -b feature/xyz                  # create + switch
git switch -c feature/xyz                    # same, newer syntax
git switch main                              # switch branch
git branch -d feature/xyz                    # delete local branch (safe)
git branch -D feature/xyz                    # force delete local branch
git push origin --delete feature/xyz         # delete remote branch
```

### Stashing
```bash
git stash                                    # save uncommitted changes
git stash push -m "wip: notes"               # stash with a message
git stash list                               # see all stashes
git stash pop                                # apply + remove most recent stash
git stash apply stash@{1}                    # apply specific stash, keep it in list
git stash drop stash@{0}                     # delete a specific stash
git stash clear                              # delete all stashes
```

---

## 3. Rewriting History (use with care)

```bash
git commit --amend                           # edit last commit message/content
git commit --amend --no-edit                 # add staged changes to last commit, keep message

git rebase -i HEAD~3                         # interactive rebase, last 3 commits
# pick / reword / edit / squash / fixup / drop

git rebase main                              # rebase current branch onto main
git rebase --continue                        # after resolving conflicts mid-rebase
git rebase --abort                           # bail out, restore pre-rebase state

git reset --soft HEAD~1                      # undo last commit, keep changes staged
git reset --mixed HEAD~1                     # undo last commit, keep changes unstaged (default)
git reset --hard HEAD~1                      # undo last commit, DISCARD changes (destructive)

git revert <commit-hash>                     # create a new commit that undoes a commit (safe for shared branches)

git cherry-pick <commit-hash>                # apply a specific commit onto current branch
```

> **Rule of thumb:** `reset`/`rebase` rewrite history — safe on branches only you use. `revert` is safe on shared/public branches because it doesn't rewrite anything.

---

## 4. Common Errors & Fixes

### "fatal: not a git repository"
You're not inside a repo folder, or `.git` is missing/corrupted.
```bash
cd /path/to/repo      # make sure you're in the right dir
git status             # confirms it's a repo
```

### "Updates were rejected because the remote contains work that you do not have locally"
Someone else pushed since your last pull.
```bash
git pull --rebase origin main     # replay your commits on top of latest
# resolve any conflicts, then:
git push
```
Avoid `git push --force` on shared branches unless you're certain — it overwrites others' work. Use `--force-with-lease` instead if you must force-push (it fails safely if remote has new commits you don't know about):
```bash
git push --force-with-lease
```

### Merge conflicts
```bash
git status                        # shows conflicted files
# open each file, resolve the <<<<<<< ======= >>>>>>> markers
git add <resolved-file>
git commit                        # (if mid-merge) or:
git rebase --continue              # (if mid-rebase)
```
Abort options if it gets messy:
```bash
git merge --abort
git rebase --abort
```

### "fatal: refusing to merge unrelated histories"
Usually happens on first push to a repo initialized both locally and on GitHub (e.g. README created on GitHub, repo also initialized locally).
```bash
git pull origin main --allow-unrelated-histories
```

### Detached HEAD state
You checked out a commit/tag directly instead of a branch.
```bash
git checkout main                 # get back to a branch
# OR, if you made commits you want to keep:
git switch -c new-branch-name     # save the detached work into a branch
```

### Accidentally committed to the wrong branch
```bash
git log                            # note the commit hash
git reset --soft HEAD~1            # undo commit on wrong branch, keep changes staged
git stash                          # stash the changes
git switch correct-branch
git stash pop
git commit -m "message"
```

### Accidentally committed a large/sensitive file
```bash
git rm --cached path/to/file       # stop tracking, keep file locally
echo "path/to/file" >> .gitignore
git commit -m "remove tracked file"
```
If it needs removing from **history** entirely (e.g. leaked secret), use `git filter-repo` (preferred over old `filter-branch`) or BFG Repo-Cleaner, then force-push and rotate the leaked credential immediately.

### "Permission denied (publickey)"
SSH key isn't set up or added to the agent.
```bash
ssh-add -l                          # check loaded keys
ssh-add ~/.ssh/id_ed25519
ssh -T git@github.com               # test
```

### "Your branch is ahead of 'origin/main' by N commits"
Local commits haven't been pushed yet.
```bash
git push
```

### "Your branch and 'origin/main' have diverged"
Both sides have new commits.
```bash
git pull --rebase                   # cleanest, avoids merge commit
# or
git pull                            # creates a merge commit
```

### Undo a `git add` (unstage)
```bash
git restore --staged file.txt       # newer syntax
git reset HEAD file.txt             # older syntax
```

### Discard local changes to a file
```bash
git restore file.txt                # newer syntax
git checkout -- file.txt            # older syntax
```

### Wrong commit message, already pushed
```bash
git commit --amend -m "correct message"
git push --force-with-lease
```
(Only safe if no one else has pulled that commit yet.)

### Line ending warnings (CRLF/LF, Windows/Mac/Linux teams)
```bash
git config --global core.autocrlf input    # Mac/Linux
git config --global core.autocrlf true     # Windows
```

---

## 5. Working with Remotes

```bash
git remote -v                              # list remotes
git remote add origin <url>                # add a remote
git remote set-url origin <new-url>        # change remote URL (e.g. https → ssh)
git remote remove upstream                 # remove a remote
```

**Keeping a fork in sync with upstream:**
```bash
git remote add upstream git@github.com:original-owner/repo.git
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

---

## 6. Pull Requests — Creating, Reviewing, Checking Status

### Using GitHub CLI (`gh`) — fastest day-to-day option
```bash
gh auth login                              # one-time setup

gh pr create --base main --head feature/xyz --title "Add X" --body "Description"
gh pr create --fill                        # auto-fill title/body from commits
gh pr list                                 # list open PRs in repo
gh pr list --state all                     # include closed/merged
gh pr view 123                             # view PR details in terminal
gh pr view 123 --web                       # open PR in browser
gh pr checkout 123                         # check out someone else's PR locally
gh pr diff 123                             # view PR diff in terminal
gh pr status                               # PRs relevant to you (created, requested review, etc.)
gh pr checks 123                           # CI/status checks for a PR
gh pr review 123 --approve -b "LGTM"
gh pr review 123 --request-changes -b "Please fix X"
gh pr comment 123 -b "Looks good, one nit"
gh pr merge 123 --squash                   # merge (squash/merge/rebase)
gh pr close 123
gh pr ready 123                            # mark draft PR as ready for review
```

### Reviewing a PR locally (without `gh`)
```bash
git fetch origin pull/123/head:pr-123      # fetch PR #123 into a local branch
git switch pr-123
# test, review code, run it locally
```

### Good PR review checklist
- Does the PR description explain **why**, not just what?
- Is the diff scoped to one concern (not mixing refactors + features)?
- Do tests exist/pass for the change? Check `gh pr checks <id>`.
- Any leftover debug code, commented-out blocks, or TODOs?
- Are commit messages meaningful (matters more if squash-merge isn't used)?
- Does it follow the repo's style/lint rules? (CI usually catches this)
- Check for secrets/credentials accidentally included.
- For UI changes: is there a screenshot/recording in the description?

### Common PR-related commands
```bash
git log main..feature/xyz --oneline        # commits in branch not yet in main
git diff main...feature/xyz                # full diff of what the PR would introduce
git rebase -i main                         # clean up commits before opening PR
```

---

## 7. Tags & Releases

```bash
git tag                                    # list tags
git tag v1.0.0                             # lightweight tag
git tag -a v1.0.0 -m "Release 1.0.0"       # annotated tag (preferred for releases)
git push origin v1.0.0                     # push one tag
git push origin --tags                     # push all tags
git tag -d v1.0.0                          # delete local tag
git push origin --delete v1.0.0            # delete remote tag

gh release create v1.0.0 --notes "Release notes here"
```

---

## 8. Inspecting History & Debugging

```bash
git log --oneline --graph --decorate --all       # visual history across branches
git log -p -- file.txt                            # full diff history of a file
git log --author="Name"                           # commits by a specific author
git blame file.txt                                # who changed each line, and when
git show <commit-hash>                            # full details of one commit
git bisect start                                  # binary search for a bug-introducing commit
git bisect bad                                    # current commit is broken
git bisect good <commit-hash>                     # known-good commit
# git checks out midpoints; mark each as good/bad until it finds the culprit
git bisect reset                                  # end bisect session
```

---

## 9. .gitignore Basics

```gitignore
node_modules/
.env
*.log
dist/
.DS_Store
```
```bash
git rm -r --cached node_modules       # if already tracked, untrack before ignoring
```

---

## 10. Quick Reference — "I want to..."

| Goal | Command |
|---|---|
| Save WIP without committing | `git stash` |
| Undo last commit, keep changes | `git reset --soft HEAD~1` |
| Undo last commit, discard changes | `git reset --hard HEAD~1` |
| Undo a pushed commit safely | `git revert <hash>` |
| Get someone's PR locally | `gh pr checkout <id>` |
| See what a PR changes | `gh pr diff <id>` |
| Check if CI passed on a PR | `gh pr checks <id>` |
| Sync fork with upstream | `fetch upstream` → `merge upstream/main` |
| Clean up messy commits before PR | `git rebase -i main` |
| Force-push safely | `git push --force-with-lease` |
| Remove a file from git but keep locally | `git rm --cached file` |
| Find which commit broke something | `git bisect` |

---

*Keep this file updated as new errors/solutions come up — treat it as a living doc.*
