---
name: Storing Data in the Warehouse
dependsOn: [Working_Safely_with_Git.working_safely_with_git.03_gitignore]
tags: [git, data]
learningOutcomes:
  - Explain why data belongs in the warehouse rather than in a repository
  - Identify what belongs in the warehouse
  - Request access to the data warehouse
---

# Storing Data in the Warehouse

At CEMA we work with some of the most sensitive data in Kenya: HIV
estimates, malaria case counts from KHIS, DHS survey households, and animal
disease reports from the Ministry of Agriculture and Livestock Development.
Counties, ministries, and study participants trust us with it.

This lesson explains where that data should live, why, and how you get
access to it. The rule is simple:

> **GitHub stores code. The CEMA Data Warehouse stores data.**

## Why data lives in the warehouse, not in repos

### Four situations that sound familiar

These are *not* real CEMA incidents. They are everyday situations, and each
one leads to a leak.

- **"It's a private repo, it's fine."** Wanjiru commits
  `khis_malaria_2024.xlsx` to her private thesis repository so her supervisor
  can run the code. Two years later she makes the repository public so a
  journal can review the code. The data goes public with it.
- **"I deleted it straight away."** Otieno notices a line list with phone
  numbers in his commit, deletes it, and commits again. The file is still in
  the history. Everyone who cloned the repository has a copy, and GitHub
  shows it to anyone who opens the old commit.
- **"I'll just put the token in for now."** Achieng pastes her warehouse API
  token into `analysis.R` to get a quick result before a county meeting, then
  pushes it. Her account has HBTAP access. Anyone who finds that token now
  sees everything she can see.
- **"The consultant needed the data."** A collaborator from another
  institution is added to the GitHub repository to help with a model. The
  contract ends, but they still have a full clone, including every dataset
  that was ever committed.

### It happens to large organisations too

Each of these has happened, at scale, to organisations with full security
teams:

- **Uber (2016).** Attackers got into a **private** GitHub repository and
  found cloud credentials in the code. They downloaded data on about
  57 million riders and drivers. Uber paid a US$148 million settlement in
  2018, partly because it hid the breach for a year.
- **Toyota (2017 to 2022).** A contractor published source code to a public
  GitHub repository. It contained a key to a customer data server, and about
  296,000 customers' email addresses were exposed for almost five years
  before anyone noticed.
- **US healthcare (2020).** A researcher searching GitHub found passwords and
  data belonging to nine hospitals and health insurers. Records of an
  estimated 150,000 to 200,000 patients were open to anyone.
- **Mercedes-Benz (2024).** An employee's GitHub token left in a public
  repository gave full access to the company's internal code server.

Security company GitGuardian found over 10 million passwords, tokens, and
keys in public GitHub code in 2022, and 12.8 million in 2023. Bots search
GitHub around the clock for exactly these. The warehouse sees the same bots
every day, asking for files such as `/.env` and `/.git/config`: the files
people accidentally publish.

What these incidents teach us:

- **Private is not safe.** Repositories get made public, shared, forked, and
  cloned onto laptops that get lost.
- **Git never forgets, and nobody warns you.** A deleted file stays in the
  history.
- **A token is your account.** It carries all of your access.
- **Health data is the target**, and once copied it cannot be recalled.

### See it for yourself: "but I deleted it!"

Run this in a scratch folder. It uses fake data only.

```bash
mkdir leak-demo && cd leak-demo && git init -q

# 1. commit a (fake) line list
printf 'patient_id,name,phone\n1001,Jane Wanjiku,0712000000\n' > linelist.csv
git add linelist.csv && git commit -qm "add data"

# 2. realise the mistake and delete it
git rm -q linelist.csv && git commit -qm "remove data"
ls                                   # the folder is empty...

# 3. ...but anyone with the repository can bring it back
git log --all --oneline -- linelist.csv
git show HEAD~1:linelist.csv
```

The last command prints the "deleted" patient record. The same is true for a
token:

```bash
echo 'token <- "a1b2c3d4e5"' > analysis.R
git add analysis.R && git commit -qm "analysis"
echo 'token <- Sys.getenv("CEMA_WAREHOUSE_TOKEN")' > analysis.R
git commit -qam "hide token"
git log -p | grep a1b2c3d4e5         # still there
```

### Access control and auditability

Compare a data file in a GitHub repository with the same data in the
warehouse:

| Question          | File in a GitHub repo | CEMA warehouse              |
|-------------------|-----------------------|-----------------------------|
| Who can see it?   | All repo members      | Approved accounts only      |
| Someone leaves?   | They keep their copy  | Account deactivated at once |
| A token leaks?    | Nothing to revoke     | New token; old one dies     |
| Who looked at it? | No idea               | Every read is logged        |
| Which version?    | Hard to say           | Numbered, pinnable versions |
| Where from?       | Not recorded          | Source, steward, code repo  |

Size matters too: GitHub rejects files over 100 MB, and every version of every
file is stored forever, so repositories with data in them become slow to
clone.

### The law

Under the **Kenya Data Protection Act, 2019**:

- Health data is **sensitive personal data** and needs extra protection.
  Most of what we handle at CEMA qualifies.
- CEMA must keep personal data secure. A copy in a GitHub repository is a
  copy nobody controls.
- A breach must be reported to the **Office of the Data Protection
  Commissioner (ODPC) within 72 hours**.
- The ODPC can fine up to KES 5 million or 1% of annual turnover, whichever
  is lower. Unlawful disclosure can also be a criminal offence.

On top of that, your **ethics approval**, **data-sharing agreements** with
counties and ministries, and the **DHS terms of use** say where data may be
kept. GitHub is never on that list, and a leak can cost CEMA its next
data-sharing agreement.

::::challenge{id=git_find_the_leak title="Find the leak"}

Your facilitator shares a practice repository with a fake token and a fake
line list hidden in its history. Using `git log --all --stat`, `git log -p`,
and `git show <commit>:<file>`, find:

1. which commit added the data file, and who made it;
2. the token;
3. how many commits later each one was "removed".

If this were your thesis repository and it had been public for a month, what
would you do?

:::solution

```bash
git log --all --stat              # find the commit that added the file
git log -p | grep -i token        # find the token in the diffs
git show <commit>:<file>          # print the "deleted" file
```

If it were real: **generate a new warehouse token immediately** (the old one
stops working at once), then **tell the data steward and the warehouse
admins the same day**. The history has to be rewritten and everyone who
cloned the repository must delete their copy. Do not try to fix it quietly.

:::

::::

## What belongs there

### Where does my file go?

Rule of thumb: **if it came from a ministry, a county, a survey, or a
participant, it goes in the warehouse.**

**Warehouse:**

- KHIS and DHIS2 extracts (malaria, HIV, immunisation, and so on).
- Your cleaned analysis datasets, with a link to your GitHub repository.
- County and sub-county tables for reports. Small counts can still identify
  people.
- Animal surveillance and livestock data from the ministry.

**Warehouse, with extra protection:**

- Line lists, cohort, or lab data with names, phone numbers, or IDs: tick
  **Restricted (HBTAP)** when uploading, after removing what you do not
  need.
- DHS recode files go in the **DHS section**. DHS terms forbid sharing them
  anywhere else.
- Ward and county boundaries and rasters go in the **GIS section**.

**GitHub:**

- Your R, Python, and Stata code, R Markdown, Quarto, and READMEs.
- Data dictionaries (column names and meanings only).
- A tiny, fully **synthetic** dataset so your tests run, clearly named, for
  example `example_synthetic.csv`.

**Neither:**

- Your API token and passwords: only in `.Renviron` or `.env` on your laptop.
- Consent forms and scanned IDs: follow your study's records procedure.

### Keep only what the analysis needs

Before you upload, ask: *does my analysis need this column?*

| Instead of               | Keep                                    |
|--------------------------|-----------------------------------------|
| Name, phone, national ID | A study ID (key kept by the study team) |
| Exact date of birth      | Age or year of birth                    |
| Household GPS point      | Village, ward, or sub-county            |
| Free-text clinical notes | Coded variables                         |

Still contains personal data? Tick **Restricted (HBTAP)** when uploading.

### File formats

- **Data you want to query from R, Python, or Stata:** upload as `.csv`,
  `.xlsx`, `.xls`, or `.parquet`.
- **Documents to keep with the data** (codebook, protocol, report): `.pdf`,
  `.docx`, `.pptx`, `.txt`, `.png`, `.jpg`, or `.zip`.

RDS and Stata files cannot be uploaded directly. Convert them first:

```r
library(arrow)
write_parquet(readRDS("data/clean_cases.rds"), "data/clean_cases.parquet")

survey <- haven::zap_labels(haven::read_dta("data/survey.dta"))
write_parquet(survey, "data/survey.parquet")
```

```stata
use "data/survey.dta", clear
export delimited using "data/survey.csv", replace nolabel
```

Value labels are dropped during conversion, so upload your **codebook**
alongside the data.

::::challenge{id=git_where_does_it_go title="Where does it go?"}

For each file, decide: *warehouse (open)*, *warehouse (restricted)*,
*GitHub*, or *neither*.

1. `khis_malaria_cases_county_2024.xlsx`
2. `hiv_cohort_linelist_with_phone_numbers.csv`
3. `clean_and_model.R`
4. `.Renviron`
5. `kenya_wards.shp`
6. `fake_10_rows_for_tests.csv`
7. `KEHR8AFL.DTA` (DHS household recode)
8. `rvf_outbreak_reports_by_county.rds`

:::solution

1. Warehouse (open).
2. Warehouse (restricted). Drop the phone numbers if the analysis does not
   need them.
3. GitHub.
4. Neither. It holds your token and stays on your laptop.
5. Warehouse, GIS section.
6. GitHub, if it really is fake.
7. Warehouse, DHS section.
8. Warehouse, converted to Parquet first.

:::

::::

## How to get access

### 1. Get an account

1. Go to <https://warehouse.cema.africa/register>.
2. Register with your `@cema.africa`, `@uonbi.ac.ke`, or
   `@students.uonbi.ac.ke` email address. Collaborators from other
   institutions ask the warehouse admins to add their email first.
3. Wait for approval. **A warehouse admin approves every new account**, and
   you cannot log in until then.
4. Forgot your password? Use **Forgot password** on the login page. The link
   works once and expires after 24 hours.

### 2. Ask for the access your project needs

- **Any approved user** can browse and download open datasets, read them
  from R, Python, or Stata, and upload datasets.
- **HBTAP study members** can also see restricted (HBTAP) datasets. Ask the
  warehouse admins; your PI confirms you are on the study.
- **Private DHS data** and **uploading DHS folders** need extra permission
  from the warehouse admins.
- **Data stewards** see and manage the version history of their datasets.
  You become a steward when you are named as one on upload.

Ask for the access your project needs, no more. Access is removed when you
leave a project or CEMA.

### 3. Generate your API token

Open **Profile** and click **Generate token**. Your scripts use this token
to read data from the warehouse.

- **It is your account.** Whoever holds it sees everything you can,
  including HBTAP data if you have access.
- Keep it in `.Renviron` or `.env` only. Never put it in code, notebooks,
  WhatsApp, Slack, or email, and never share it with a colleague; they should
  get their own account.
- **Think it leaked?** Click **Generate token** again. The old token stops
  working immediately. Then tell the warehouse admins.

The next lesson shows how to use the token to connect from R and Python.

### 4. Upload your data

1. Remove the columns you do not need and convert RDS or Stata files.
2. Open **Upload** and choose the right category.
3. Give the dataset a clear name and description. Your colleagues will
   search for it.
4. Tick **Restricted (HBTAP)** if it holds personal data.
5. Fill in the **provenance**: where the data came from, how you processed
   it, your **GitHub repository link**, and the **data steward**.
6. Uploading an update creates a **new version**. Old versions are kept, so
   earlier analyses still reproduce.
7. Once your code reads the data from the warehouse, **delete the copies**
   on your laptop, in email, and on WhatsApp.

::::challenge{id=git_warehouse_setup title="Get set up"}

1. Log in to the warehouse and generate your token.
2. Save it in `.Renviron` (R) or `.env` (Python) in your course repository,
   then run `git status` to check that the file does not appear.
3. Convert a practice `.rds` or `.dta` file to Parquet and upload it to the
   **Training** category, linking your course repository in the provenance.
4. Delete your local copy. In the next lesson you will read it back from the
   warehouse.

:::solution

```bash
echo "CEMA_WAREHOUSE_TOKEN=<your-token>" > .Renviron
echo ".Renviron" >> .gitignore
git status        # .Renviron must not be listed
```

```r
arrow::write_parquet(readRDS("practice.rds"), "practice.parquet")
```

Upload `practice.parquet` through the **Upload** page, fill in the
provenance, then delete both local files.

:::

::::

## Remember

```text
DATA          -> the warehouse (restricted if it is about people)
CODE          -> GitHub, reading its data from the warehouse
TOKEN         -> .Renviron / .env only; leaked? regenerate and tell admins
"I DELETED IT" -> it is still in the Git history; report it the same day
```

**Questions or a possible leak?** Contact the warehouse admins straight away.
Reporting early is never the wrong call.

### Sources

- Uber: US FTC complaint and the 2018 multistate settlement announcements.
- Toyota: Toyota Motor Corporation statement on T-Connect, October 2022.
- US healthcare: Jelle Ursem and DataBreaches.net, *No need to hack when it's
  leaking*, August 2020.
- Mercedes-Benz: RedHunt Labs disclosure, January 2024.
- Secret counts: GitGuardian, *State of Secrets Sprawl* reports, 2023 and
  2024.
- Kenya Data Protection Act, 2019, and ODPC enforcement notices.
