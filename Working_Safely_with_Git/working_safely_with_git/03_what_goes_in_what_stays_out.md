---
name: What Goes In, What Stays Out
dependsOn: [Working_Safely_with_Git.working_safely_with_git.02_branches_and_pull_requests]
tags: [git, security]
learningOutcomes:
  - Identify which files belong in a repository
  - Identify secrets and sensitive data that must never be committed
  - Recognise personal and identifying information and why it cannot be committed
  - Keep credentials out of scripts by reading them from the environment
  - Use the .env.example pattern to share configuration safely
---

# What Goes In, What Stays Out

A repository holds the **instructions** for a project (the code and how to run it), not the **inputs** (credentials and data).
Anyone with access to the repo, now or later, can read everything in its history.

## ✅ What goes in

- Source code: R, Python, SQL and shell scripts
- Documentation: `README.md` and notes on how to run the project
- Dependency files: `renv.lock`, `requirements.txt`, `pyproject.toml`
- Configuration **templates** with placeholder values, e.g. `.env.example`
- Small **synthetic** or public sample data used for tests
- A `.gitignore` file (covered in the next lesson)

## ❌ What stays out

| Never commit | Examples | Where it belongs |
|---|---|---|
| Secrets | API keys, passwords, tokens, `.env`, `.Renviron`, `credentials.json`, `*.pem` | **Passbolt** |
| Sensitive data | CSV, Excel, RDS, `.dta`, `.sav` or Parquet files with personal or health data | **The data warehouse** (see *Storing Data in the Warehouse*) |
| Data exports & dumps | `.sql` dumps, `.sqlite`/`.db` files, `.RData` workspaces | The data warehouse |
| Generated outputs | Large result files, rendered reports, model outputs | Shared storage, or re-generate them from code |

**Rule of thumb:** if you wouldn't post it publicly, it doesn't go in Git, even in a private repo.

## Sensitive data and personal information

Much of the data CEMA works with describes **people**: patients, households, health workers, survey respondents.
Committing it to a repository is not just untidy, it is a data protection problem.
Kenya's Data Protection Act treats health data as a special category, and a commit is a disclosure you cannot take back.

Treat a file as sensitive if it contains any of:

- **Direct identifiers:** names, national ID or passport numbers, phone numbers, email addresses, patient or beneficiary IDs, physical addresses
- **Health information:** diagnoses, test results, treatment or medication records, HIV or TB status
- **Location detail:** household GPS coordinates, village or sub-location names tied to individual records
- **Demographic detail:** exact dates of birth, dates of admission, discharge or death

### "De-identified" does not mean safe to commit

Removing the name column is not enough.
A record with a facility, an exact visit date, an age and a sex is often enough to identify one person, especially in a small catchment area.
Linking two such files makes it easier still.

So the rule is about the file, not the column names:

> If the rows describe real people, the file belongs in the **data warehouse**, not in Git. Even aggregated or "anonymised" extracts stay out unless the data owner has cleared them for release.

### A private repo is not a safe place

- Every member of the organization, and anyone added later, can read the whole history.
- Repositories get cloned onto laptops, forked, and occasionally flipped to public by mistake.
- A collaborator who leaves the team keeps any clone they already have.

If you need realistic data for tests or teaching, generate **synthetic** data: same column names and types, invented values. That can be committed.

## Credentials belong outside your code

The most common way a secret reaches GitHub is not a stray `.env` file, it is a **credential typed directly into a script**.
It is quick, it works on your machine, and it goes unnoticed until the script is committed.

Here is the pattern to avoid. This R script pulls metadata from a DHIS2 instance:

```r
# ❌ BAD: credentials hardcoded in the script
library(httr)

base_url <- "https://hmis.example.org"
username <- "cema_api_user"
password <- "S3cr3t-Passw0rd!"

des_res <- GET(
  paste0(base_url, "/api/dataElements"),
  query = list(
    fields = ":all",
    paging = "false"
  ),
  authenticate(username, password)
)
```

Commit that once and the credential is in the history permanently, together with the server it opens.
Deleting the lines in a later commit does **not** remove them, and a password left blank on your copy (`password <- " "`) tells the next person exactly which variable to fill in, so it is only safe until someone pastes a real value in and commits.

Read the values from the environment instead:

```r
# ✅ GOOD: credentials read from the environment
library(httr)

base_url <- Sys.getenv("HMIS_BASE_URL")
username <- Sys.getenv("HMIS_USERNAME")
password <- Sys.getenv("HMIS_PASSWORD")

if (password == "") {
  stop("HMIS_PASSWORD is not set. Copy .env.example to .Renviron and fill it in from Passbolt.")
}

des_res <- GET(
  paste0(base_url, "/api/dataElements"),
  query = list(
    fields = ":all",
    paging = "false"
  ),
  authenticate(username, password)
)
```

The real values live in an uncommitted `.Renviron` in the project root:

```text
# .Renviron (NOT committed): real values, stored in Passbolt
HMIS_BASE_URL=https://hmis.example.org
HMIS_USERNAME=cema_api_user
HMIS_PASSWORD=S3cr3t-Passw0rd!
```

Restart the R session after editing `.Renviron`; R reads it only at startup.
In Python the equivalent is `os.environ["HMIS_PASSWORD"]`, with the values in a `.env` file loaded by `python-dotenv`.

Two habits that catch this before it is too late:

- Search a script for `password`, `token`, `api_key` and `secret` before you commit it.
- Read the **Files changed** tab of your pull request. A hardcoded credential is obvious there.

If a credential has already been committed, do not just delete the line. Rotate it first, then follow *When Something Goes Wrong*.

## Sharing configuration safely: `.env.example`

Your code often needs secrets, such as a database password.
Keep the real values in a `.env` file that is **never committed**, and commit a `.env.example` that shows which values are needed:

```bash
# .env.example (committed): placeholders only
DB_HOST=your-database-host
DB_USER=your-username
DB_PASSWORD=get-this-from-passbolt
```

```bash
# .env (NOT committed): real values, stored in Passbolt
DB_HOST=10.0.0.12
DB_USER=jdoe
DB_PASSWORD=s3cr3t-value
```

A new team member copies `.env.example` to `.env` and fills it in from **Passbolt**, CEMA's password manager.

## Watch out for `git add .`

`git add .` and `git add -A` stage **everything** in the folder, including any data or `.env` file you saved there.
Add files by name, or run `git status` first and read the list.

::::challenge{id=sort_the_files title="Commit or don't commit?"}
A project folder contains these files.
Sort them into **commit** and **don't commit**, and say where each "don't commit" file belongs.

```text
analysis.R
.env
.env.example
README.md
patients_2026.csv
requirements.txt
.Renviron
synthetic_test_cases.csv
outputs/final_report.html
renv.lock
model_fit.RData
```

:::solution

**Commit:**

- `analysis.R`: code
- `.env.example`: template with placeholders only
- `README.md`: documentation
- `requirements.txt`, `renv.lock`: dependencies
- `synthetic_test_cases.csv`: synthetic test data, **provided it is truly synthetic**

**Don't commit:**

- `.env`, `.Renviron`: contain secrets → **Passbolt**
- `patients_2026.csv`: sensitive data → **data warehouse**
- `model_fit.RData`: a workspace that can contain data and credentials → re-create it from code
- `outputs/final_report.html`: generated output → shared storage, or re-generate it

:::
::::
