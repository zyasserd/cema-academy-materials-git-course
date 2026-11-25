---
name: Appendix
dependsOn: [Working_Safely_with_Git.working_safely_with_git.06_deploy_keys]
tags: [git, github]
learningOutcomes:
  - Use the command cheat sheet as a day-to-day reference
  - Find the CEMA services and external references used in this course
  - Complete the course sign-off step
---

# Appendix

## Cheat sheet

### The everyday loop

| Command | What it does |
|---|---|
| `git clone <url>` | Get a copy of a repository |
| `git status` | Show what has changed and what is staged |
| `git diff` | Show exactly how unstaged files changed |
| `git diff --staged` | Show what you are about to commit — **run this every time** |
| `git add <file>` | Stage one named file |
| `git commit -m "message"` | Save a snapshot of the staged changes |
| `git push` | Send your commits to GitHub |
| `git pull` | Bring other people's commits into your copy |
| `git log --oneline` | List the commits, most recent first |

Prefer `git add <file>` to `git add .`. Naming files is the cheapest protection
in this entire course.

### Checking what is ignored

| Command | What it does |
|---|---|
| `git check-ignore -v <file>` | Show which `.gitignore` rule ignores a file; prints nothing if it is **not** ignored |
| `git status --ignored` | List ignored files alongside the usual output |
| `git rm --cached <file>` | Stop tracking a file, keep it on disk |
| `git log --oneline -- <file>` | Show every commit that touched a file |
| `git show <commit>:<file>` | Print a file as it was at that commit |

### Starter `.gitignore`

The full version, with explanations, is in lesson 03.

```text
# --- Secrets ---
.env
.env.*
!.env.example
.Renviron
*.pem
*.key
credentials*.json

# --- Data files ---
data/
*.csv
*.xlsx
*.xls
*.rds
*.RData
*.dta
*.sav
*.parquet
*.sqlite
*.db

# --- R ---
.Rhistory
.Rproj.user/

# --- Python ---
__pycache__/
.ipynb_checkpoints/
.venv/

# --- OS ---
.DS_Store
Thumbs.db
```

### Reading a secret from the environment

```r
# R: value lives in .Renviron (never committed)
token <- Sys.getenv("CEMA_WAREHOUSE_TOKEN")
if (token == "") stop("CEMA_WAREHOUSE_TOKEN is not set. Get it from Passbolt.")
```

```python
# Python: value lives in .env (never committed)
import os
token = os.environ["CEMA_WAREHOUSE_TOKEN"]
```

### If you have committed a secret

1. **Rotate** it at the source, then store the new value in Passbolt.
2. **Report** it the same day: line manager, SAIT, and the Data Protection
   Officer if personal data was involved.
3. **Clean up** with help — do not rewrite history alone.

The full procedure is in lesson 02, *If it has already been committed*.

## Where things belong

| What | Where | Never |
|---|---|---|
| Code, docs, `.gitignore`, `.env.example` | GitHub | — |
| Passwords, tokens, API keys, `.pem` files | Passbolt | In a repository, in a script, in Slack or email |
| Data about people | The CEMA Data Warehouse | In a repository, even privately |
| Generated outputs and large result files | Shared storage, or regenerate from code | In a repository |

## CEMA services

| Service | What it is for |
|---|---|
| [CEMA-DataScience on GitHub](https://github.com/CEMA-DataScience) | Code repositories |
| [CEMA Data Warehouse](https://warehouse.cema.africa) | Datasets, API tokens — lessons 04 and 05 |
| [Passbolt](https://passbolt-dev.cema.africa) | CEMA's password manager, for every credential in this course |
| SAIT | Servers, service accounts, deploy keys, history rewrites |

> **To confirm before delivery:** how to contact SAIT (ticket queue, Slack
> channel or email), the name and contact details of CEMA's Data Protection
> Officer, and the warehouse access-request route. These are referred to by name
> throughout the course but are not written down anywhere in it. Also confirm
> that `passbolt-dev.cema.africa` is the instance staff should actually use —
> the hostname reads like a development deployment.

## Further reading

- [Pro Git](https://git-scm.com/book/en/v2) — the standard free Git book;
  chapters 1 to 3 cover everything in lesson 01
- [GitHub: ignoring files](https://docs.github.com/en/get-started/getting-started-with-git/ignoring-files)
- [`gitignore.io`](https://www.toptal.com/developers/gitignore) — generate a
  starting `.gitignore` for R, Python or any other stack
- [GitHub: removing sensitive data from a repository](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)
- [`git-filter-repo`](https://github.com/newren/git-filter-repo) — the tool SAIT
  uses for history rewrites
- [GitHub: managing deploy keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)
- [Kenya Data Protection Act, 2019](https://www.odpc.go.ke/dpa-act/) — the legal
  background to lesson 02

## Sign-off

Work through this checklist on a repository you actually use. It is the
practical test that the course has landed.

- [ ] I can explain, in one sentence, why deleting a file does not remove it
      from Git history.
- [ ] The repository has a `.gitignore` covering secrets and data files, and it
      is committed.
- [ ] `git check-ignore -v` confirms that a `.csv` and a `.env` in that
      repository are both ignored.
- [ ] No credential appears in any script in the repository. I have searched it
      for `password`, `token`, `api_key` and `secret`.
- [ ] Every credential the project needs is in Passbolt, and the repository has
      a `.env.example` (or `.Renviron.example`) listing the names only.
- [ ] No file describing real people is tracked in the repository. Where there
      was one, the analysis now reads it from the warehouse.
- [ ] I know who to tell if I commit a secret, and I know that rotating it comes
      first.

> **To confirm before delivery:** how sign-off is actually recorded — self-
> assessment, a short session with a line manager, or a trainer-signed checklist
> — and whether completion is tracked anywhere.

Once the checklist is complete, the follow-on course covers collaboration:
branches, pull requests, review, and enforcing those rules through branch
protection on self-hosted GitLab or Gitea.
