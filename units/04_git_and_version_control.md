# Unit 04: Git & Version Control

## Overview

Git is the version control system used by virtually every software project on Linux. It tracks every change you make to your code, lets you experiment safely on branches, and enables collaboration with other developers. For a systems programmer, git is also a research tool: `git log`, `git blame`, and `git bisect` let you understand why code is the way it is and pinpoint exactly which commit introduced a bug. This unit covers git from first principles to the workflows used in professional open-source and commercial Linux development.

## Prerequisites

- Unit 03 (The Art of the Command Line) — comfortable in the terminal, text editor proficiency

## Learning Objectives

- Initialize repositories and understand the working tree, index (staging area), and commit history
- Commit changes with meaningful messages
- Work with branches: create, switch, merge, and rebase
- Collaborate using remote repositories: clone, push, pull, fetch
- Resolve merge conflicts
- Use `git stash`, `git bisect`, `git blame`, and `git reflog` for advanced workflows
- Write a useful `.gitignore` for C projects
- Understand git's object model (blobs, trees, commits, refs)

## Reading / Resources

- `man git`, `man git-log`, `man git-rebase`, etc.
- Pro Git book (free): https://git-scm.com/book/en/v2 — Chapters 1–3, 6–7
- `git help <command>` — built-in help for any subcommand

## Concepts

### The Three Areas

Git manages three areas:

```
Working Directory   →  (git add)  →  Index (Staging Area)  →  (git commit)  →  Repository
     (your files)                        (what will be                           (commit history)
                                          in next commit)
```

A **commit** is a snapshot of the entire project at a point in time. Git stores the full content, not diffs. The history is a linked list of commits, each pointing to its parent.

### Initial Setup

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global core.editor "vim"
git config --global init.defaultBranch main
git config --global color.ui auto
```

### Core Workflow

```bash
git init                    # create a new repository
git clone URL               # clone an existing repository

git status                  # show working directory and staging area state
git diff                    # unstaged changes
git diff --staged           # staged changes (what's in the next commit)

git add file.c              # stage a file
git add -p                  # interactively stage hunks (very useful)
git add .                   # stage everything in current directory

git commit -m "message"     # commit staged changes
git commit                  # opens editor for message

git log                     # show commit history
git log --oneline           # compact view
git log --oneline --graph --all   # full branch graph
git log -p                  # show patches (diffs) with each commit
git log --author="name"     # filter by author
git log --since="2 weeks"   # filter by date
git log -- file.c           # history of a specific file
```

### Writing Good Commit Messages

A commit message has two parts: a subject line and an optional body.

```
Fix null pointer dereference in connection handler

When a client disconnects during the handshake phase, recv()
returns 0 but the code continued processing the NULL buffer.
Add an explicit check after recv() and close the fd cleanly.

Fixes: #142
```

Rules:
- Subject line: 50 characters max, imperative mood ("Fix", "Add", "Remove"), no period
- Blank line between subject and body
- Body: explain *why*, not *what* (the diff shows what)

### Branches

```bash
git branch                  # list branches
git branch feature-x        # create branch
git switch feature-x        # switch to branch (modern syntax)
git switch -c feature-x     # create and switch in one step

git merge feature-x         # merge feature-x into current branch
git branch -d feature-x     # delete merged branch
git branch -D feature-x     # force delete unmerged branch
```

**Fast-forward merge**: if the current branch has not diverged, git just moves the pointer forward — no merge commit.

**Three-way merge**: if branches have diverged, git creates a merge commit with two parents.

### Rebase

Rebase replays your commits on top of another branch, creating a linear history:

```bash
git switch feature-x
git rebase main             # replay feature-x commits on top of main
```

Before:
```
main:      A - B - C
feature:   A - B - D - E
```

After rebase:
```
main:      A - B - C
feature:   A - B - C - D' - E'
```

**Golden rule**: never rebase commits that have been pushed to a shared remote. Rebase rewrites history — it creates new commits with new hashes.

Use `merge` for integrating shared branches. Use `rebase` to clean up local work before pushing.

### Resolving Merge Conflicts

When two branches modify the same lines, git cannot auto-merge:

```
<<<<<<< HEAD
int timeout = 30;
=======
int timeout = 60;
>>>>>>> feature-x
```

Edit the file to the correct state, remove the markers, then:

```bash
git add conflicted_file.c
git merge --continue        # or git commit
```

Tools: `git mergetool` opens a visual diff tool. `git diff --diff-filter=U` shows only conflicted files.

### Remotes

```bash
git remote -v               # list remotes
git remote add origin URL   # add a remote named "origin"

git fetch origin            # download remote changes (don't merge)
git pull                    # fetch + merge (or fetch + rebase with --rebase)
git push origin main        # push main branch to origin
git push -u origin feature  # push and set upstream tracking

git push origin --delete feature  # delete remote branch
```

### Useful Everyday Commands

```bash
# Stash: save work in progress without committing
git stash                   # save current changes
git stash list              # show stashed changes
git stash pop               # restore last stash
git stash pop stash@{2}     # restore specific stash

# Undo changes
git restore file.c          # discard unstaged changes (irreversible)
git restore --staged file.c # unstage a file
git revert HEAD             # create a new commit that undoes last commit (safe)
git reset HEAD~1            # undo last commit, changes become unstaged
git reset --soft HEAD~1     # undo last commit, keep changes staged
git reset --hard HEAD~1     # undo last commit AND discard changes (destructive)

# Cherry-pick: apply a specific commit to current branch
git cherry-pick abc123

# Show what changed in a commit
git show abc123
git show HEAD               # last commit
```

### git bisect — Binary Search for Bugs

When you know "this worked 3 months ago but doesn't now" and there are hundreds of commits in between:

```bash
git bisect start
git bisect bad              # current commit is broken
git bisect good v1.0        # this tag was known-good

# git checks out the midpoint commit
# Test your program, then:
git bisect good             # or git bisect bad

# Repeat until git identifies the first bad commit
git bisect reset            # return to HEAD when done
```

Automate with a test script:
```bash
git bisect run ./test_script.sh
```

### git blame — Who Wrote This Line?

```bash
git blame file.c            # show who last modified each line
git blame -L 40,60 file.c   # only lines 40-60
git log -S "search_string"  # find commits that added/removed this string
```

### .gitignore for C Projects

Create `.gitignore` in the project root:

```
# Build outputs
*.o
*.a
*.so
*.so.*

# Executables (no extension on Linux)
# List explicitly or use a build/ directory convention
build/
cmake-build-*/

# Debug/profiling output
*.d
gmon.out
callgrind.out.*
cachegrind.out.*
massif.out.*
core
core.*

# Editor files
.vscode/
.idea/
*.swp
*.swo
*~
.DS_Store

# Test output
*.gcno
*.gcda
*.gcov
```

### Git's Object Model

Understanding this eliminates confusion about what git can and cannot do:

```
blob    — file contents (just bytes, no filename)
tree    — directory: maps names to blobs and other trees
commit  — snapshot: points to a tree, has author/message/parent(s)
tag     — named pointer to a commit
ref     — a file containing a commit hash (branches, HEAD, tags)
```

Everything is content-addressed by SHA-1 hash. Two identical files = one blob. This is why git is fast and why you can never truly "lose" committed work — it stays in the object store until garbage collected.

```bash
git cat-file -t HEAD        # type of HEAD object (commit)
git cat-file -p HEAD        # print the commit object
git cat-file -p HEAD^{tree} # print the tree object
```

### Reflog — The Safety Net

The reflog records every movement of HEAD, even after reset and rebase:

```bash
git reflog                  # show all recent HEAD positions
git checkout HEAD@{5}       # go back to where HEAD was 5 moves ago
```

If you `git reset --hard` by accident, `git reflog` is how you recover.

## Exercises

### Exercise 1: First Repository

Initialize a git repository in a new directory, create a simple C `hello.c`, and build a history with at least 5 commits:
1. Initial hello world
2. Add a function
3. Fix a bug in the function
4. Add a Makefile
5. Add `.gitignore`

Use `git log --oneline --graph` to view the history. Use `git show` to inspect each commit's diff.

### Exercise 2: Branching and Merging

In the repository from Exercise 1:
1. Create a branch `feature/greeting` and add a `greet(const char *name)` function
2. Make 2–3 commits on this branch
3. Meanwhile, make 1 commit on `main` (modify the Makefile)
4. Merge `feature/greeting` into `main` — observe whether it is a fast-forward or three-way merge
5. Delete the feature branch

Repeat but use `rebase` instead of `merge`. Compare `git log --graph` output for both approaches.

### Exercise 3: Conflict Resolution

Set up a conflict:
1. Create file `config.h` with `#define TIMEOUT 30`
2. On `main`: change it to `60`
3. On `branch-b`: change it to `120`
4. Merge `branch-b` into `main`
5. Resolve the conflict by choosing `120`
6. Complete the merge and verify history

### Exercise 4: git bisect

Create a repository with 20 commits. In commit number 13, introduce a bug (e.g. a function returns the wrong value). Use `git bisect` with a test script that exits 0 for "good" and 1 for "bad" to find the exact commit that introduced the bug without looking at the commit messages.

### Exercise 5: Exploring a Real Project

Clone a real Linux C project (e.g. `git clone https://github.com/antirez/kilo`):
1. Use `git log --oneline` to see the history
2. Use `git log --author` to find commits by a specific person
3. Use `git blame` to find who wrote a specific function
4. Use `git log -S "function_name"` to find when a function was added
5. Use `git show <hash>` to read the full diff of an interesting commit

## What Comes Next

Unit 05 begins C programming. From now on, every exercise should be committed to a git repository — treat version control as part of your development workflow, not an afterthought.
