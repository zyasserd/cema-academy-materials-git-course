---
name: Collaborating with Branches & Pull Requests
dependsOn: [Working_Safely_with_Git.working_safely_with_git.01_git_refresher]
tags: [git, github]
learningOutcomes:
  - Explain why work should not be pushed directly to main or a deployment branch
  - Work on a feature branch and open a pull request
  - Review and respond to a pull request
---

# Collaborating with Branches & Pull Requests

`main` (and any deployment branch such as `production` or `deploy`) is the version of the project that other people rely on.
Often it is what runs on a server or produces published results.
Pushing straight to it skips every check: nobody reviews the change, and a mistake reaches everyone immediately.

**The rule: never push directly to `main` or a deployment branch. Every change goes through a branch and a pull request.**

## The branch workflow

```bash
git switch main
git pull                          # start from the latest main
git switch -c fix-date-parsing    # one branch per piece of work
# ...edit, then commit as often as you like...
git add R/clean_dates.R
git commit -m "Handle dd/mm/yyyy dates"
git push -u origin fix-date-parsing
```

Name branches after what they do (`fix-date-parsing`, `add-county-map`), not after people.

## Opening a pull request

1. After you push, GitHub shows a **Compare & pull request** button on the repo page. Click it.
2. Check that the **base** is `main` and the **compare** branch is your branch.
3. Write a short title and description: what changed, why, and how to check it.
4. Under **Reviewers**, add at least one colleague.
5. Click **Create pull request**.

Before you request a review, read the **Files changed** tab yourself.
This is the easiest place to spot a data file or a `.env` file that shouldn't be there.

## Reviewing a pull request

When you are the reviewer:

- Read the **Files changed** tab and check every file belongs in the repo.
- Leave comments on specific lines and ask questions.
- Finish with **Approve** or **Request changes**.

Merge only after approval.
Once the branch is merged, delete it on GitHub.

## Protected branches

Many CEMA repos have **branch protection** on `main`.
If you try to push to it directly, you'll see an error like:

```text
remote: error: GH006: Protected branch update failed for refs/heads/main.
```

This is expected.
Push your work to a branch and open a pull request instead.
If a repo you maintain has no protection on `main`, ask an organization admin to turn it on.

## Keeping your branch up to date

If `main` has moved on while you were working:

```bash
git switch main
git pull
git switch fix-date-parsing
git merge main        # bring the latest main into your branch
# resolve any conflicts, then commit and push
```

::::challenge{id=open_a_pr title="Open your first pull request"}
In a practice repository on GitHub (create one in your own account, or use one your team provides):

1. Create a branch called `update-readme`.
2. Add a line to `README.md`, commit it and push the branch.
3. Open a pull request into `main` and request a review from a colleague.
4. Once it's approved, merge it and delete the branch.

:::solution
You have succeeded if:

- No commit was pushed directly to `main`. The **Commits** list on `main` shows a merge from the pull request.
- The pull request shows one approving review.
- The `update-readme` branch was deleted after the merge.

If `git push` was rejected with a "protected branch" error, you pushed to `main` by mistake.
Run `git switch -c update-readme`, then push that branch instead.

:::
::::
