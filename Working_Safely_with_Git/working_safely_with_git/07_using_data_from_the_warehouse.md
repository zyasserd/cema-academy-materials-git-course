---
name: Using Data from the Warehouse Instead of CSVs on GitHub
dependsOn: [Working_Safely_with_Git.working_safely_with_git.06_when_something_goes_wrong]
tags: [git, github, data-warehouse, api, reproducibility, data-protection]
learningOutcomes:
  - Explain why analysis data should be read from the CEMA Data Warehouse rather than committed to GitHub.
  - Generate a warehouse API token and store it outside the repository.
  - Discover datasets and their columns through the warehouse API.
  - Read warehouse datasets into R and Python with a pinned version and selected columns.
  - Interpret warehouse API errors and fix common problems.
  - Migrate an existing repository from committed CSV files to warehouse API calls.
---

# Using Data from the Warehouse Instead of CSVs on GitHub

## The one rule

> **GitHub stores code. The CEMA Data Warehouse stores data.**
> Your repository holds the *recipe*, never the *ingredients*.

A very common project layout looks like this:

```text
malaria-analysis/
  analysis.R
  data/
    khis_malaria_2024_03.csv
    khis_malaria_2024_03_clean.csv
    khis_malaria_2024_04_FINAL.xlsx
```

Someone exports a table, commits it, and everyone analyses their own copy.
The previous lesson, *Storing Data in the Warehouse*, explained why this is
unsafe and what belongs in the warehouse. This lesson shows how to replace
those files with code that reads the data straight from the warehouse.

## How the warehouse API works

The [CEMA Data Warehouse](https://warehouse.cema.africa) has a REST API. Your
script asks for a dataset each time it runs, and the warehouse sends back the
rows:

```text
 your script --HTTPS: dataset slug + options + token--> warehouse.cema.africa
             <------------- CSV or JSON rows ---------
```

Anyone with warehouse access who clones your repository can reproduce your
work with their own token. Anyone without access sees no data at all.

A request is just a web address:

```text
https://warehouse.cema.africa/api/v2/dataset/sth-subcounty-2023
    ?fmt=csv&version=1&cols=county,year&token=XXXX
```

- `sth-subcounty-2023` is the dataset's **slug**: the last part of its page
  URL in the warehouse (`/datasets/sth-subcounty-2023`).
- Everything after `?` are **options**, joined with `&`.
- `token=XXXX` proves who you are.

## Step 1: Get an account and a token

You need an approved warehouse account and an API token (**Profile >
Generate token**). See *How to get access* in the previous lesson.

## Step 2: Store the token outside your code

### R: `.Renviron`

Create a file called `.Renviron` in the project root:

```text
CEMA_WAREHOUSE_TOKEN=paste-your-token-here
```

Restart R, then check that it was picked up:

```r
nchar(Sys.getenv("CEMA_WAREHOUSE_TOKEN")) > 0
```

### Python: `.env`

Create a file called `.env` in the project root with the same line. The
`python-dotenv` package loads it.

### Ignore the real file, commit a template

Add the secrets files and data formats to `.gitignore`:

```text
# secrets
.Renviron
.env

# data: lives in the warehouse, never in Git
data/
output/
*.csv
*.xlsx
*.rds
*.dta
*.parquet
```

Commit **example** files with no real value, so collaborators know which
variable to set:

```bash
echo "CEMA_WAREHOUSE_TOKEN=" > .Renviron.example
echo "CEMA_WAREHOUSE_TOKEN=" > .env.example
git add .gitignore .Renviron.example .env.example
git status        # .Renviron and .env must NOT be listed
git commit -m "Add warehouse token templates and ignore rules"
```

## Step 3: Discover datasets in the browser

While logged in to the warehouse in your browser, your session counts as
authentication, so you can explore the API without a token:

- `https://warehouse.cema.africa/api/v2/datasets` lists every dataset you
  can access.
- `https://warehouse.cema.africa/api/v2/dataset/<slug>/columns` lists a
  dataset's columns and their types.
- `https://warehouse.cema.africa/api/v2/dataset/<slug>?limit=5` shows the
  first five rows as JSON.

There is also an interactive reference at
<https://warehouse.cema.africa/api/docs>.

Each entry in the dataset list includes a `storage_type`:

- `database`: tabular data that you can read through the API.
- `sftp`: a stored file (PDF, Word document, archive). Download it from the
  dataset page in your browser instead.

## Step 4: Know the options

These options work on `/api/v2/dataset/<slug>`:

| Option    | Example                  | Meaning                             |
|-----------|--------------------------|-------------------------------------|
| `fmt`     | `fmt=csv`                | `csv`, `tsv`, or `json` (default)   |
| `version` | `version=2`              | Pin a version (default: latest)     |
| `cols`    | `cols=county,year,cases` | Fetch only these columns            |
| `limit`   | `limit=1000`             | First *n* rows (1 to 100,000)       |
| `sample`  | `sample=500`             | Random *n* rows (1 to 10,000)       |

- **Always use `fmt=csv` in scripts.** CSV is streamed, so large tables work
  and there is no row cap. JSON is loaded into memory first.
- **Pin `version` for any result that goes into a report or paper.** The
  warehouse keeps every version, so anyone re-running your code next year
  gets exactly the same rows.
- **Use `cols`** to fetch only what you need. It is faster and keeps less
  data, especially personal data, on your laptop.
- `limit` and `sample` cannot be combined. Leave both out to get all rows.

## Step 5: Read data from R

Save this helper as `R/warehouse.R` in your project. It builds the request,
reads the CSV response, and turns warehouse errors into readable messages
without ever printing your token. It needs the `curl`, `jsonlite`, and
`data.table` packages.

```r
WAREHOUSE_URL <- "https://warehouse.cema.africa"

.warehouse_get <- function(path, params = list()) {
  token <- Sys.getenv("CEMA_WAREHOUSE_TOKEN")
  if (!nzchar(token)) {
    stop("CEMA_WAREHOUSE_TOKEN is not set. Add it to .Renviron, restart R.")
  }
  params <- c(Filter(Negate(is.null), params), token = token)
  query <- paste(
    names(params),
    vapply(params, function(v) curl::curl_escape(as.character(v)), ""),
    sep = "=", collapse = "&"
  )
  res <- curl::curl_fetch_memory(paste0(WAREHOUSE_URL, path, "?", query))
  body <- rawToChar(res$content)
  Encoding(body) <- "UTF-8"
  if (res$status_code >= 400) {
    detail <- tryCatch({
      j <- jsonlite::fromJSON(body)
      paste(c(j$error, j$hint), collapse = " | ")
    }, error = function(e) substr(body, 1, 200))
    detail <- gsub(token, "<token>", detail, fixed = TRUE)
    stop(sprintf("Warehouse returned %d for %s: %s",
                 res$status_code, path, detail), call. = FALSE)
  }
  body
}

list_datasets <- function() {
  jsonlite::fromJSON(.warehouse_get("/api/v2/datasets"), flatten = TRUE)
}

warehouse_columns <- function(slug) {
  path <- sprintf("/api/v2/dataset/%s/columns", slug)
  jsonlite::fromJSON(.warehouse_get(path))$columns
}

read_warehouse <- function(slug, version = NULL, cols = NULL,
                           limit = NULL, sample = NULL) {
  body <- .warehouse_get(sprintf("/api/v2/dataset/%s", slug), list(
    fmt = "csv", version = version,
    cols = if (!is.null(cols)) paste(cols, collapse = ","),
    limit = limit, sample = sample
  ))
  data.table::fread(text = body, showProgress = FALSE)
}
```

Then use it in your analysis:

```r
source("R/warehouse.R")

# 1. discover
ds <- list_datasets()
ds[ds$storage_type == "database", c("slug", "display_name", "version")]
warehouse_columns("sth-subcounty-2023")

# 2. explore with a small random sample
peek <- read_warehouse("sth-subcounty-2023", sample = 500)

# 3. analyse: full data, pinned version, only the columns you need
sth <- read_warehouse(
  "sth-subcounty-2023",
  version = 1,
  cols = c("county", "subcounty", "year", "prevalence")
)
```

## Step 6: Read data from Python

Save this helper as `warehouse.py`. It needs `requests`, `pandas`, and
`python-dotenv`:

```bash
pip install requests pandas python-dotenv
```

```python
"""Read data from the CEMA Data Warehouse API."""
import io
import os

import pandas as pd
import requests
from dotenv import load_dotenv

load_dotenv()

WAREHOUSE_URL = "https://warehouse.cema.africa"


def _get(path, params=None):
    """GET a warehouse endpoint, raising a readable error on failure."""
    token = os.environ.get("CEMA_WAREHOUSE_TOKEN")
    if not token:
        raise RuntimeError("CEMA_WAREHOUSE_TOKEN is not set. Add it to .env.")
    params = {k: v for k, v in (params or {}).items() if v is not None}
    params["token"] = token
    r = requests.get(WAREHOUSE_URL + path, params=params, timeout=600)
    if not r.ok:
        # never print r.url: it contains the token
        try:
            j = r.json()
            keys = ("error", "hint", "valid_cols")
            detail = " | ".join(str(j[k]) for k in keys if k in j)
        except ValueError:
            detail = r.text[:200]
        detail = detail.replace(token, "<token>")
        raise RuntimeError(
            f"Warehouse returned {r.status_code} for {path}: {detail}"
        )
    return r


def list_datasets():
    """All datasets you can access."""
    return pd.json_normalize(_get("/api/v2/datasets").json())


def warehouse_columns(slug):
    """Column names and types of one dataset."""
    r = _get(f"/api/v2/dataset/{slug}/columns")
    return pd.DataFrame(r.json()["columns"])


def read_warehouse(slug, version=None, cols=None, limit=None, sample=None):
    """Rows of a tabular dataset. Pin `version` for reproducible work."""
    r = _get(f"/api/v2/dataset/{slug}", {
        "fmt": "csv",
        "version": version,
        "cols": ",".join(cols) if cols else None,
        "limit": limit,
        "sample": sample,
    })
    return pd.read_csv(io.StringIO(r.content.decode("utf-8")))
```

Use it in your analysis:

```python
from warehouse import list_datasets, read_warehouse, warehouse_columns

ds = list_datasets()
print(ds[ds.storage_type == "database"][["slug", "display_name"]])
print(warehouse_columns("sth-subcounty-2023"))

peek = read_warehouse("sth-subcounty-2023", sample=500)

sth = read_warehouse(
    "sth-subcounty-2023",
    version=1,
    cols=["county", "subcounty", "year", "prevalence"],
)
print(sth.head())
```

## Step 7: Document the data in your README

List every dataset the project uses, with its pinned version, so a reader
knows exactly what the results are based on:

```markdown
## Data

All data is read from the CEMA Data Warehouse. No data is stored in this
repository.

| Dataset (warehouse slug) | Version | Steward |
|--------------------------|---------|---------|
| sth-subcounty-2023       | 1       | <name>  |

## How to run

1. Generate an API token on your warehouse profile page.
2. Copy `.Renviron.example` to `.Renviron` (R) or `.env.example` to `.env`
   (Python) and paste your token in.
3. Restart R and run `analysis.R`.
```

## When it does not work

The warehouse answers errors with JSON that usually says what to fix. The
helpers above show that message.

| Status | Meaning                        | Fix                                |
|--------|--------------------------------|------------------------------------|
| `400`  | Bad option or a file dataset   | Read `error`, `hint`, `valid_cols` |
| `401`  | Missing, wrong, or old token   | Check `.Renviron`/`.env`, restart  |
| `403`  | No access (HBTAP, private DHS) | Ask the data steward or an admin   |
| `404`  | Wrong slug or version          | Check `/api/v2/datasets`           |
| `500`  | Server problem                 | Retry later, then tell the admins  |

Debugging tips:

- `Sys.getenv("CEMA_WAREHOUSE_TOKEN")` is empty? Restart R after editing
  `.Renviron`.
- Paste the request URL **without the token** into a browser tab where you
  are logged in to the warehouse; your session works in place of the token.
- Do not paste the token into terminal commands; it ends up in your shell
  history.

## Caching large datasets locally

For a big table you re-run often, download it once into the git-ignored
`data/` folder. Put the version in the file name so the cache can never go
silently stale:

```r
cache <- "data/sth-subcounty-2023_v1.csv"

if (!file.exists(cache)) {
  dir.create("data", showWarnings = FALSE)
  data.table::fwrite(read_warehouse("sth-subcounty-2023", version = 1), cache)
}

sth <- data.table::fread(cache)
```

Delete local copies when the project ends.

## Where outputs and derived data go

- **Intermediate files** (cleaned tables, model fits, figures) go in `data/`
  or `output/`. Both are ignored, so they stay on your machine.
- **Derived datasets that others will reuse** are **uploaded to the
  warehouse** (Upload page), not emailed or committed. Fill in the
  provenance, including your **GitHub repository link**, so the data points
  back to the code that made it. Uploading an update creates a new version
  and keeps the old ones.
- A **tiny, fully synthetic** dataset needed for tests may live in the
  repository, clearly named (for example `tests/example_synthetic.csv`).
  Never commit a "sample" of real records.

## Migrating an existing repository

If your repository already contains data files:

1. **Make sure the data is in the warehouse.** If it is not, upload it (as
   `.csv`, `.xlsx`, or `.parquet`) with its provenance filled in.
2. **Switch the code** from `read.csv("data/...")` to
   `read_warehouse("<slug>", version = N)`. Check that row counts and key
   totals match the old file.
3. **Stop tracking the files** and ignore them:

   ```bash
   git rm --cached -r data/
   echo "data/" >> .gitignore
   git add .gitignore R/warehouse.R analysis.R
   git commit -m "Read data from the warehouse instead of committed files"
   ```

4. **If the files contained personal or sensitive data**, removing them from
   the latest commit is not enough; they are still in the history. Treat it
   as a data incident: tell the data steward and the warehouse admins the
   same day. The history must be rewritten and everyone must re-clone (see
   *Ignoring Files and Protecting Data*).
5. **If a token was ever committed**, click **Generate token** on your
   warehouse profile immediately. The old token stops working at once.

## Checklist before you push

- [ ] `git status` shows no data files (`.csv`, `.xlsx`, `.rds`, `.dta`,
      `.parquet`) staged.
- [ ] The token comes from `.Renviron` / `.env`, never from code.
- [ ] `fmt=csv` is used for anything bigger than a quick look.
- [ ] Every dataset that feeds a result is read with a pinned `version`.
- [ ] `cols` fetches only the columns you need.
- [ ] The README lists dataset slugs and versions.
- [ ] The code runs from a fresh clone with someone else's token.

## Practice

::::challenge{id=git_warehouse_migration title="Swap a CSV for the warehouse"}

Your project has the following `analysis.R`, and `data/sth.csv` is committed
to the repository:

```r
sth <- read.csv("data/sth.csv")
totals <- aggregate(prevalence ~ county, data = sth, FUN = mean)
```

The same data is in the warehouse as `sth-subcounty-2023`, version 1.

1. Store your token safely and add a template for collaborators.
2. Rewrite the analysis to read from the warehouse, pinned to version 1 and
   fetching only the columns it needs.
3. Stop tracking the CSV and prevent data files from being committed again.
4. Swap repositories with a partner. Clone theirs, add your own token, and
   run it. It should work without them sending you any file.

:::solution

```bash
echo "CEMA_WAREHOUSE_TOKEN=<your-token>" > .Renviron
echo "CEMA_WAREHOUSE_TOKEN=" > .Renviron.example
printf '.Renviron\n.env\ndata/\noutput/\n*.csv\n' >> .gitignore
```

Save the helper from Step 5 as `R/warehouse.R`, then change `analysis.R` to:

```r
source("R/warehouse.R")

sth <- read_warehouse(
  "sth-subcounty-2023",
  version = 1,
  cols = c("county", "prevalence")
)
totals <- aggregate(prevalence ~ county, data = sth, FUN = mean)
```

```bash
git rm --cached data/sth.csv
git add .gitignore .Renviron.example R/warehouse.R analysis.R
git status        # .Renviron and data/ must not be listed
git commit -m "Read STH data from the warehouse instead of committed CSV"
git push
```

Your partner clones the repository, copies `.Renviron.example` to
`.Renviron`, pastes their own token, restarts R, and runs `analysis.R`.

If `sth.csv` held identifiable data, it is still in the history: tell the
data steward and warehouse admins.

:::

::::

## Quick reference

```text
BASE       https://warehouse.cema.africa
LIST       /api/v2/datasets
COLUMNS    /api/v2/dataset/<slug>/columns
ROWS       /api/v2/dataset/<slug>?fmt=csv&version=N&cols=a,b&token=$TOKEN
DOCS       /api/docs
TOKEN      .Renviron / .env only; leaked? Generate token again, tell admins
```
