---
name: Git Refresher
dependsOn: [Working_Safely_with_Git.working_safely_with_git.00_introduction]
tags: [git]
learningOutcomes:
  - Describe repositories, commits, branches and remotes
  - Run the everyday Git loop from clone to push
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
| **Branch** | An independent line of work inside a repo |
| **Remote** | A copy of the repo hosted elsewhere, e.g. on GitHub (`origin`) |
| **Staging area** | Where you put the changes you want in your next commit |

## The everyday loop

```bash
git clone https://github.com/CEMA-DataScience/example-repo.git   # get a copy
git switch -c my-change      # start a branch
# ...edit files...
git status                   # see what changed
git diff                     # see exactly how it changed
git add analysis.R           # stage specific files
git commit -m "Add incidence plot"
git push -u origin my-change # send your branch to GitHub
git pull                     # get other people's changes
```

Make a habit of running `git status` and `git diff` **before every commit**.
They are the last chance to spot a file that shouldn't be there.

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
