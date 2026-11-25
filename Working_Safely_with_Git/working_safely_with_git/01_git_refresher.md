---
name: Git Refresher
dependsOn: [Working_Safely_with_Git.working_safely_with_git.00_introduction]
tags: [git]
learningOutcomes:
  - Describe repositories, commits and remotes
  - Run the everyday Git loop from clone to push
  - Inspect what you are about to commit before you commit it
  - Undo a change that has not been committed yet
  - Recover when a push is rejected because someone else pushed first
  - Explain why anything committed stays in Git history
---

# Git Refresher

This lesson is a quick reminder of the Git basics used in the rest of the course.
If you already use Git every day, skim it and do the challenge at the end. The challenge covers the most important idea in this course.

## Key terms

| Term | What it means |
|---|---|
| **Repository (repo)** | A project folder whose full history Git tracks |
| **Commit** | A saved snapshot of the project at a point in time |
| **Remote** | A copy of the repo hosted elsewhere, e.g. on GitHub (`origin`) |
| **Staging area** | Where you put the changes you want in your next commit |

## The everyday loop

```bash
git clone https://github.com/CEMA-DataScience/example-repo.git   # get a copy
# ...edit files...
git status                   # see what changed
git diff                     # see exactly how it changed
git add analysis.R           # stage specific files
git commit -m "Add incidence plot"
git push                     # send your commits to GitHub
git pull                     # get other people's changes
```

That is the whole loop. Most days you will run nothing else.

## Look before you commit

The commands in the middle of that loop are the ones worth slowing down for.
They are your last chance to spot a file that should not be there, and they
cost a few seconds:

| Command | Answers |
|---|---|
| `git status` | Which files have changed, and which are staged for the next commit? |
| `git diff` | How did the files I have **not** staged change? |
| `git diff --staged` | What exactly am I about to commit? |
| `git log --oneline` | What has been committed before? |

`git status` is the one to internalise. Its output has three parts, and the
middle one is where accidents live:

```text
$ git status
On branch main
Changes to be committed:        # staged: these go into the next commit
        modified:   analysis.R

Changes not staged for commit:  # changed, but not staged
        modified:   README.md

Untracked files:                # Git has never seen these
        data/patients_2026.csv
        .env
```

Those last two lines are exactly what the rest of this course is about. Git is
telling you it has noticed a data file and a secrets file, and is waiting to be
told what to do with them. `git add .` at this point would stage both.

## Undoing things before you commit

Nothing here has been committed yet, so nothing is permanent and everything is
easy to fix:

```bash
git restore README.md          # throw away unstaged changes to a file
git restore --staged .env      # unstage a file, keep it on disk
git rm --cached .env           # stop tracking a file, keep it on disk
```

`git restore --staged` is the one to reach for the moment you notice you have
staged something you did not mean to. **Before** a commit, this is a non-event.
After a commit, it is a different lesson.

## When someone else pushed first

Everyone here works on `main`, so sooner or later two people commit at once and
your push is rejected:

```text
$ git push
 ! [rejected]        main -> main (fetch first)
error: failed to push some refs to 'github.com:CEMA-DataScience/example-repo.git'
hint: Updates were rejected because the remote contains work that you do not
hint: have locally.
```

Nothing is broken and nothing is lost. GitHub has commits you do not. Bring
them in, then push again:

```bash
git pull        # fetch their commits and combine them with yours
git push        # now your commits go up too
```

If you both changed **different** files, that is the end of it. If you both
changed the **same lines** of the same file, Git stops and asks you to choose:

```text
Auto-merging analysis.R
CONFLICT (content): Merge conflict in analysis.R
```

`git status` lists the conflicted files. Open each one and you will find both
versions marked:

```text
<<<<<<< HEAD
incidence <- cases / population * 1000      # your version
=======
incidence <- cases / population * 100000    # their version
>>>>>>> origin/main
```

Delete the markers and the version you do not want, keep the line you do, then:

```bash
git add analysis.R
git commit            # completes the merge
git push
```

If it looks like a mess, ask a colleague before experimenting. Conflicts are
normal, and pulling **before** you start work each day makes them rare.

> **A note on branches.** You may have heard of branches, and you will see a
> `main` branch in every repository you clone. Working on *other* branches, and
> reviewing changes through pull requests, matters once several people edit the
> same project at once. That is the subject of the follow-on course, and this one
> does not need it: everything here works on `main`.

## Git never forgets

Git keeps every commit ever made.
If you delete a file in a later commit, it disappears from your folder, but **the earlier commit still contains it**.
Anyone with access to the repo can get it back.

This is why the rest of the course is about stopping the wrong files from being committed in the first place.

::::challenge{id=git_never_forgets title="Is it really gone?"}
In a new, empty practice folder:

1. Run `git init`.
2. Create a file called `notes.txt` containing the text `password=hunter2` and commit it.
3. Delete the file with `git rm notes.txt` and commit again.
4. Check that `notes.txt` is no longer in your folder.

Can you recover the contents of `notes.txt` using only Git?

:::solution

```bash
git log --oneline -- notes.txt   # shows both commits that touched the file
git show HEAD~1:notes.txt        # prints the file as it was one commit ago
```

The output is `password=hunter2`.
Deleting the file didn't remove it from history.
Once the repo is pushed to GitHub, everyone with access to it has that history too.

:::
::::
