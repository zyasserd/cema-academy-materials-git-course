---
name: Protecting Your Repo with .gitignore
dependsOn: [Working_Safely_with_Git.working_safely_with_git.03_what_goes_in_what_stays_out]
tags: [git, security]
learningOutcomes:
  - Explain what a .gitignore file does and where it lives
  - Add a starter .gitignore to a CEMA project
  - Check that sensitive files are ignored and stop tracking files committed by mistake
---

# Protecting Your Repo with `.gitignore`

A `.gitignore` is a plain text file listing the files Git should **ignore**.
Ignored files don't appear in `git status` and aren't picked up by `git add .`, so they can't be committed by accident.

It lives in the **root of the repository** and is committed like any other file.
That way, everyone who clones the repo gets the same protection.

## Pattern basics

| Pattern | Ignores |
|---|---|
| `.env` | Any file named `.env` |
| `*.csv` | Every file ending in `.csv` |
| `data/` | The whole `data` folder |
| `/secrets.txt` | `secrets.txt` in the repo root only |
| `!.env.example` | **Exception:** don't ignore this file, even though `.env.*` matched it |
| `# comment` | Nothing; it's a comment |

## A starter `.gitignore` for CEMA projects

Copy this into a file called `.gitignore` in the root of your repo:

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

Then commit it:

```bash
git add .gitignore
git commit -m "Add .gitignore"
```

If a project really needs a small, non-sensitive data file in the repo (for example, synthetic test data), add an exception:

```text
!tests/fixtures/synthetic_test_cases.csv
```

## Checking it works

```bash
git status                       # ignored files should not be listed
git check-ignore -v patients.csv # shows which rule ignores the file
```

If `git check-ignore` prints nothing, the file is **not** ignored.

## Files that are already tracked

`.gitignore` only affects files Git isn't tracking yet.
If a file was committed **before** you added the rule, Git keeps tracking it.
To stop tracking it without deleting your local copy:

```bash
git rm --cached patients.csv
git commit -m "Stop tracking patients.csv"
```

This removes the file from **future** commits, but it is **still in the history** (remember the Git Refresher challenge).
If the file contained a secret or sensitive data, follow *When Something Goes Wrong*.

::::challenge{id=gitignore_check title="Prove your .gitignore works"}
In your practice repository:

1. Add the starter `.gitignore` above and commit it.
2. Create three files: `.env`, `.env.example` and `patients.csv`.
3. Run `git status`.

Which files does Git offer to commit? Use `git check-ignore -v` to confirm why.

:::solution
`git status` shows only `.env.example` as a new file.
`.env` and `patients.csv` don't appear.

```bash
$ git check-ignore -v .env patients.csv
.gitignore:2:.env	.env
.gitignore:12:*.csv	patients.csv

$ git check-ignore -v .env.example
.gitignore:4:!.env.example	.env.example
```

Lines starting with `!` mean the file matched an exception rule, so it is **not** ignored.
`.env.example` can be committed.

:::
::::
