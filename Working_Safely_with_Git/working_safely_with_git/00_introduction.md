---
name: Introduction
dependsOn: [Working_Safely_with_Git.working_safely_with_git]
tags: [git, github]
learningOutcomes:
  - Identify who this course is for and what it covers
  - Find your way around the CEMA GitHub organisations
  - Set up Git and GitHub access before the first lesson
---

# Introduction

Most people at CEMA meet Git because a colleague sent them a repository link.
You clone it, you edit a script, you commit, and it works. Nothing in that
experience warns you that **everything you commit is permanent**, or that a
single `git add .` can publish a line list of patients to everyone in the
organisation.

This course is about the handful of habits that prevent that. It is short on
Git theory and long on what to do on a Tuesday afternoon with a `data/`
folder full of KHIS extracts.

## Who this course is for

Anyone at CEMA who commits to a repository: epidemiologists, data scientists,
analysts, M&E staff, interns and SAIT.

It assumes you have **used Git at least once**, even if you did not enjoy it.
You should recognise `git clone`, `git commit` and `git push`, and know how to
open a terminal. Lesson 01 is a refresher; if the commands there are unfamiliar,
work through it slowly rather than skipping it.

You do **not** need to know anything about branches, pull requests, rebasing or
merge conflicts. Those are deliberately out of scope.

### What you need before you start

| You need | How to get it |
|---|---|
| A GitHub account, with 2FA enabled | github.com, using your CEMA email address |
| Membership of the CEMA organisation | Ask your line manager to request it |
| Git installed locally | `git --version` should print a version |
| A Passbolt account | [passbolt-dev.cema.africa](https://passbolt-dev.cema.africa), CEMA's password manager |
| A warehouse account and API token | Covered in lessons 04 and 05 |

> **To confirm before delivery:** the exact names of the CEMA GitHub
> organisations, and who approves membership. The table above
> names the one organisation referenced in these materials
> ([CEMA-DataScience](https://github.com/CEMA-DataScience)); if your team's work
> lives elsewhere, tell the course maintainer so this table can be corrected.

## The CEMA GitHub organisations

CEMA's code lives in GitHub **organisations**, not in personal accounts. An
organisation owns its repositories, which means work survives people leaving,
and access is granted by role rather than by whoever happens to own the repo.

- [**CEMA-DataScience**](https://github.com/CEMA-DataScience) — the organisation
  used throughout this course. Analysis code, pipelines, dashboards and
  teaching material.

Two consequences worth internalising now:

- **Private is not secret.** Every member of the organisation can read every
  repository they are given access to, including its entire history, and
  members added years from now inherit that access.
- **Personal repositories are not a workaround.** Pushing CEMA work to your own
  account puts it outside CEMA's control and outside its backups. If it is CEMA
  work, it belongs in a CEMA organisation.

## Learning outcomes

By the end of this course you will be able to:

- Explain why anything committed to Git stays in its history permanently
- Decide what belongs in a repository and what does not
- Recognise personal and sensitive data, and keep it out of Git
- Keep credentials out of your code by reading them from the environment
- Use a `.gitignore` to keep secrets and data files out of Git
- Know where secrets and data should be stored at CEMA
- Read analysis data from the CEMA Data Warehouse instead of committing it
- Follow the standard procedure if a secret or a data file is ever committed

## How the course is structured

| Lesson | What it covers |
|---|---|
| 01 Git Refresher | The everyday commands, and why Git history is permanent |
| 02 What Goes In, What Stays Out | Code in, secrets and data out, and what to do when that goes wrong |
| 03 Protecting Your Repo with `.gitignore` | Making the rules automatic |
| 04 Storing Data in the Warehouse | Where CEMA data belongs, and how to get access |
| 05 Using Data from the Warehouse | Reading data from the API instead of committing CSVs |
| 06 Deploy Keys *(optional)* | Giving a server read-only access to a private repo |
| 07 Appendix | Cheat sheet, links and sign-off |

Lessons 01 to 03 are the core: if you only take away the permanence of history
and a working `.gitignore`, the course has done its job.

## What this course does not cover

- **Branches, pull requests and code review.** These matter once several people
  edit one project at the same time, and they are the subject of the follow-on
  course. Everything in this course works on `main`.
- **Branch protection.** We cannot enforce it on our current hosting, so it is
  deferred along with pull requests.
- **Deploy keys** (lesson 06) are **optional**. That lesson is for SAIT and for
  anyone on the Data Science team who deploys Shiny applications; the general
  epidemiology team can skip it.
- **Rewriting history** to remove a committed secret. Lesson 02 tells you who to
  ask and why you should not attempt it alone.
