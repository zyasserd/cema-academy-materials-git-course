---
name: Using Data from the Warehouse
dependsOn: [Working_Safely_with_Git.working_safely_with_git.04_storing_data_in_the_warehouse]
tags: [git, github, data-warehouse, api, reproducibility]
learningOutcomes:
  - Discover datasets and their columns through the warehouse API
  - Read warehouse datasets into R and Python with a pinned version and selected columns
  - Interpret warehouse API errors and fix common problems
  - Migrate an existing repository from committed CSV files to warehouse API calls
---

# Using Data from the Warehouse

> **Before you start.** This lesson assumes you have an approved warehouse
> account and an API token stored in `.Renviron` or `.env`, as set up in
> lesson 04. If `Sys.getenv("CEMA_WAREHOUSE_TOKEN")` is empty, go back there.

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
Lesson 04 explained why that is unsafe. This lesson replaces those files with
code that reads the data straight from the warehouse, so the repository holds
the *recipe* and never the *ingredients*.

## How the warehouse API works

Your script asks for a dataset each time it runs, and the warehouse sends back
the rows:

```text
 your script --HTTPS: dataset slug + options + token--> warehouse.cema.africa
             <------------- CSV or JSON rows ---------
```

Anyone with warehouse access who clones your repository can reproduce your work
with their own token. Anyone without access sees no data at all.

A request is just a web address:

```text
https://warehouse.cema.africa/api/v2/dataset/sth-subcounty-2023
    ?fmt=csv&version=1&cols=county,year&token=XXXX
```

- `sth-subcounty-2023` is the dataset's **slug**: the last part of its page URL
  in the warehouse (`/datasets/sth-subcounty-2023`).
- Everything after `?` are **options**, joined with `&`.
- `token=XXXX` proves who you are.

## Finding a dataset

While logged in to the warehouse in your browser, your session counts as
authentication, so you can explore without a token:

| Open in the browser | Shows |
|---|---|
| `/api/v2/datasets` | Every dataset you can access |
| `/api/v2/dataset/<slug>/columns` | A dataset's columns and their types |
| `/api/v2/dataset/<slug>?limit=5` | Its first five rows, as JSON |
| [`/api/docs`](https://warehouse.cema.africa/api/docs) | The interactive API reference |

Each entry in the dataset list has a `storage_type`. `database` means tabular
data you can read through the API; `sftp` means a stored file (PDF, Word,
archive) to download from the dataset page instead.

## The options that matter

These work on `/api/v2/dataset/<slug>`:

| Option    | Example                  | Meaning                             |
|-----------|--------------------------|-------------------------------------|
| `fmt`     | `fmt=csv`                | `csv`, `tsv`, or `json` (default)   |
| `version` | `version=2`              | Pin a version (default: latest)     |
| `cols`    | `cols=county,year,cases` | Fetch only these columns            |
| `limit`   | `limit=1000`             | First *n* rows (1 to 100,000)       |
| `sample`  | `sample=500`             | Random *n* rows (1 to 10,000)       |

Three habits carry most of the value:

- **Always `fmt=csv` in scripts.** CSV is streamed, so large tables work and
  there is no row cap. JSON is loaded into memory first.
- **Pin `version` for any result that goes into a report or paper.** The
  warehouse keeps every version, so code re-run next year returns the same rows.
- **Use `cols`** to fetch only what you need. Faster, and it keeps less personal
  data on your laptop.

`limit` and `sample` cannot be combined. Leave both out to get all rows.

## Reading data in R

Save this helper as `R/warehouse.R`. It builds the request, reads the CSV
response, and turns warehouse errors into readable messages without ever
printing your token. It needs `curl`, `jsonlite` and `data.table`.

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

Then, in your analysis:

```r
source("R/warehouse.R")

warehouse_columns("sth-subcounty-2023")            # what is in it?
peek <- read_warehouse("sth-subcounty-2023", sample = 500)   # explore

sth <- read_warehouse(                             # analyse
  "sth-subcounty-2023",
  version = 1,
  cols = c("county", "subcounty", "year", "prevalence")
)
```

## Reading data in Python

The same three functions, as `warehouse.py`. It needs
`pip install requests pandas python-dotenv`.

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
    return pd.DataFrame(_get(f"/api/v2/dataset/{slug}/columns").json()["columns"])


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

```python
from warehouse import read_warehouse, warehouse_columns

sth = read_warehouse(
    "sth-subcounty-2023",
    version=1,
    cols=["county", "subcounty", "year", "prevalence"],
)
```

## When it does not work

The warehouse answers errors with JSON that usually says what to fix, and the
helpers above show that message.

| Status | Meaning                        | Fix                                |
|--------|--------------------------------|------------------------------------|
| `400`  | Bad option or a file dataset   | Read `error`, `hint`, `valid_cols` |
| `401`  | Missing, wrong, or old token   | Check `.Renviron`/`.env`, restart  |
| `403`  | No access (HBTAP, private DHS) | Ask the data steward or an admin   |
| `404`  | Wrong slug or version          | Check `/api/v2/datasets`           |
| `500`  | Server problem                 | Retry later, then tell the admins  |

A `401` with the token sitting right there in `.Renviron` almost always means R
has not been restarted. To test a request by hand, paste it **without the
token** into a browser tab where you are logged in; your session stands in for
it. Never paste a token into a terminal command, where it lands in your shell
history.

## Caching, and what to put in the README

For a big table you re-run often, download it once into the git-ignored `data/`
folder, with the version in the filename so the cache cannot go stale unnoticed:

```r
cache <- "data/sth-subcounty-2023_v1.csv"
if (!file.exists(cache)) {
  dir.create("data", showWarnings = FALSE)
  data.table::fwrite(read_warehouse("sth-subcounty-2023", version = 1), cache)
}
sth <- data.table::fread(cache)
```

Intermediate outputs belong in the same ignored folders. Derived datasets that
others will reuse go **back into the warehouse** with their provenance filled
in, not into the repository.

Then tell the next reader what the results rest on:

```markdown
## Data

All data is read from the CEMA Data Warehouse. No data is stored in this
repository.

| Dataset (warehouse slug) | Version | Steward |
|--------------------------|---------|---------|
| sth-subcounty-2023       | 1       | <name>  |

## How to run

1. Copy `.Renviron.example` to `.Renviron` and paste in your warehouse token.
2. Restart R and run `analysis.R`.
```

## Migrating a repository that already holds data

1. **Check the data is in the warehouse.** If not, upload it with its
   provenance, as in lesson 04.
2. **Switch the code** from `read.csv("data/...")` to
   `read_warehouse("<slug>", version = N)`, and confirm row counts and key
   totals match the old file.
3. **Stop tracking the files:** `git rm --cached -r data/`, add `data/` to
   `.gitignore`, commit.
4. **If any file held personal data, or a token was ever committed**, it is
   still in the history. Rotate the token immediately, then follow *If it has
   already been committed* in lesson 02: rotate, report, clean up.

Before you push, check that `git status` lists no `.csv`, `.xlsx`, `.rds`,
`.dta` or `.parquet` files, that every dataset feeding a result has a pinned
`version`, and that the code runs from a fresh clone using someone else's token.

::::challenge{id=git_warehouse_migration title="Swap a CSV for the warehouse"}

Your project has the following `analysis.R`, and `data/sth.csv` is committed to
the repository:

```r
sth <- read.csv("data/sth.csv")
totals <- aggregate(prevalence ~ county, data = sth, FUN = mean)
```

The same data is in the warehouse as `sth-subcounty-2023`, version 1.

1. Rewrite the analysis to read from the warehouse, pinned to version 1 and
   fetching only the columns it needs.
2. Stop tracking the CSV and prevent data files from being committed again.
3. Swap repositories with a partner. Clone theirs, add your own token, and run
   it. It should work without them sending you any file.

:::solution

Save the helper from this lesson as `R/warehouse.R`, then change `analysis.R`
to:

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
printf 'data/\noutput/\n*.csv\n' >> .gitignore
git add .gitignore .Renviron.example R/warehouse.R analysis.R
git status        # .Renviron and data/ must not be listed
git commit -m "Read STH data from the warehouse instead of committed CSV"
git push
```

Your partner clones the repository, copies `.Renviron.example` to `.Renviron`,
pastes their own token, restarts R, and runs `analysis.R`.

If `sth.csv` held identifiable data, it is still in the history: rotate nothing
(there is no token), but tell the data steward and warehouse admins the same
day, and follow the procedure in lesson 02.

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
