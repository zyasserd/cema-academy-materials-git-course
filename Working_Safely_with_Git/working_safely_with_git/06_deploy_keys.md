---
name: "Optional: Creating and Configuring GitHub Deploy Keys"
dependsOn: [Working_Safely_with_Git.working_safely_with_git.05_using_data_from_the_warehouse]
tags: [git, github, ssh, deployment, security]
learningOutcomes:
  - Explain what a deploy key is and when to use one instead of a personal access token
  - Generate a dedicated SSH key pair on a deployment server
  - Register a read-only deploy key on a GitHub repository
  - Configure an SSH host alias so Git uses the right key for the right repository
  - Verify GitHub's host key before trusting it
  - Apply the security rules for managing deploy keys across several repositories
---

# Creating and Configuring GitHub Deploy Keys

> **This lesson is optional, and audience-specific.**
>
> It is written for **SAIT**, who maintain CEMA's servers, and for anyone on the
> **Data Science team** who occasionally deploys Shiny applications for demos.
>
> If you do not set up servers or deployments, you can skip it: nothing else in
> the course depends on it. It is here so that the people who do need it have a
> written procedure to follow, rather than copying a personal access token onto
> a server.

A **deploy key** is an SSH key that grants a server access to **one specific** GitHub repository, without needing a personal GitHub account or a personal access token (PAT).

That restriction is the point. A PAT belongs to a person and usually reaches every repository that person can see, so a server holding a PAT is as privileged as the colleague who created it. If they leave CEMA and their account is deactivated, the deployment breaks. A deploy key is tied to the repository instead of to a person, and can be made read-only.

Use a deploy key when a production or analysis server needs to clone and pull code from a private repository.

## The example used in this lesson

Everything below uses a fictitious repository. Substitute your own values.

| Setting | Value |
|---|---|
| GitHub organisation | `example-org` |
| Repository | `inventory-api` |
| Repository URL | `git@github.com:example-org/inventory-api.git` |
| Deployment server | Ubuntu Linux |
| Deployment directory | `/srv/inventory-api` |

The goal: let the deployment server clone and pull this private repository securely.

## Step 1. Generate an SSH key pair on the server

Connect to the deployment server over SSH, then generate a dedicated Ed25519 key pair:

```bash
ssh-keygen -t ed25519 \
  -C "inventory-api-deploy" \
  -f ~/.ssh/inventory_api_deploy
```

When prompted for a passphrase, press <kbd>Enter</kbd> to leave it empty **if** the key is used for unattended deployments. A passphrase the server cannot supply on its own would block every automated pull.

This creates two files:

```text
~/.ssh/inventory_api_deploy       # Private key
~/.ssh/inventory_api_deploy.pub   # Public key
```

Generate the key **on the server that will use it**, so the private key never travels.

> **Never share the private key or upload it to GitHub.** Only the `.pub` file is registered as a deploy key. A private key is a secret: everything in *What Goes In, What Stays Out* applies to it, and it must never be committed.

## Step 2. Display and copy the public key

```bash
cat ~/.ssh/inventory_api_deploy.pub
```

The output resembles:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... inventory-api-deploy
```

Copy the entire line, including the `ssh-ed25519` prefix and the trailing comment.

## Step 3. Register the deploy key on GitHub

1. Open the repository at `https://github.com/example-org/inventory-api`.
2. Go to **Settings → Deploy keys**.
3. Click **Add deploy key**.
4. Fill in:
   - **Title:** `Production Server – inventory-api`
   - **Key:** the public key copied in Step 2
   - **Allow write access:** leave **unchecked**
5. Click **Add key**.

Leaving write access unchecked gives the server **read-only** access. A deployment server that only pulls code never needs to push, and a read-only key cannot be used to rewrite the repository if the server is compromised.

Give the key a title that says which machine it belongs to. When a server is decommissioned, that title is how you know which key to delete.

## Step 4. Configure SSH to use the deploy key

A server may hold several deploy keys, and `github.com` alone does not tell SSH which one to offer. Define a **host alias** per repository.

Open the SSH configuration file:

```bash
nano ~/.ssh/config
```

Add:

```text
Host github-inventory-api
    HostName github.com
    User git
    IdentityFile ~/.ssh/inventory_api_deploy
    IdentitiesOnly yes
```

`IdentitiesOnly yes` tells SSH to offer **only** the key named here. Without it, SSH may offer every key the agent holds and GitHub will accept the first one that works, which may not be the one you intended.

Set the permissions SSH expects:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/inventory_api_deploy
chmod 600 ~/.ssh/config
```

SSH refuses to use a private key that other users on the machine can read.

## Step 5. Verify GitHub's SSH host key

The server must trust GitHub's genuine host key, rather than blindly accepting whatever key is presented on first connection. Inspect the fingerprint:

```bash
ssh-keyscan -t ed25519 github.com 2>/dev/null \
  | ssh-keygen -lf -
```

Compare the result with GitHub's published Ed25519 fingerprint:
<https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints>

Once it matches, add the host key:

```bash
ssh-keyscan -t ed25519 github.com >> ~/.ssh/known_hosts
chmod 600 ~/.ssh/known_hosts
```

`ssh-keyscan` retrieves the key that was presented, but it does **not** authenticate it. Checking it against the official fingerprint is what makes this step worth doing: skip the comparison and you have automated away the warning that would otherwise tell you something is wrong.

## Step 6. Test the SSH connection

```bash
ssh -T git@github-inventory-api
```

A successful connection prints something like:

```text
Hi example-org/inventory-api! You've successfully authenticated,
but GitHub does not provide shell access.
```

GitHub naming the repository, rather than a username, confirms it recognised the **deploy key** and matched it to the right repository.

## Step 7. Clone the repository

```bash
sudo mkdir -p /srv/inventory-api
sudo chown "$USER:$USER" /srv/inventory-api

git clone \
  git@github-inventory-api:example-org/inventory-api.git \
  /srv/inventory-api
```

Note that the clone URL uses `github-inventory-api`, **not** `github.com`. That is what selects the SSH configuration from Step 4. Using `github.com` here is the most common mistake in this process: the alias is bypassed, the deploy key is never offered, and the clone fails with a permission error.

## Step 8. Verify that Git operations work

```bash
cd /srv/inventory-api
git remote -v
```

Expected output:

```text
origin  git@github-inventory-api:example-org/inventory-api.git (fetch)
origin  git@github-inventory-api:example-org/inventory-api.git (push)
```

Then test fetching:

```bash
git fetch origin
```

If that succeeds without an authentication error, the deploy key works. From then on, update the deployment with:

```bash
git pull --ff-only
```

`--ff-only` refuses to create a merge commit on the server. A deployment directory should only ever fast-forward to what is already on GitHub; if it cannot, something has been changed on the server by hand and needs looking at.

## Security considerations

| Rule | Why |
|---|---|
| **One key per repository** | GitHub does not allow the same deploy key to be registered on more than one repository. |
| **Read-only by default** | Do not tick *Allow write access* unless the server genuinely must push. |
| **Protect the private key** | Restrict it to the deployment user: `chmod 600`, and no copies elsewhere. |
| **Rotate keys** | Replace deploy keys when a server is compromised or decommissioned, and delete the key on GitHub at the same time. |
| **No unnecessary privileges** | The deployment user should not need root to run Git operations, hence the `chown` in Step 7. |

For several repositories on the same server, generate a **separate key pair and a distinct host alias for each one**. If a single server needs many repositories, a dedicated machine user with read access is usually easier to manage than a dozen deploy keys.

::::challenge{id=deploy_key_diagnosis title="Why does the clone fail?"}
A colleague followed this lesson on a new server. The deploy key is registered on `example-org/inventory-api` with write access unchecked, and `ssh -T git@github-inventory-api` prints the expected "Hi example-org/inventory-api!" message.

Their clone still fails:

```bash
$ git clone git@github.com:example-org/inventory-api.git /srv/inventory-api
Cloning into '/srv/inventory-api'...
ERROR: Repository not found.
fatal: Could not read from remote repository.
```

1. Why does the clone fail when the SSH test succeeded?
2. What is the fix?
3. Later, the same server needs to deploy a second private repository, `example-org/stock-reports`. Can the existing deploy key be reused?

:::solution

**1.** The clone URL uses `github.com`, not the `github-inventory-api` alias. The alias is what points SSH at `~/.ssh/inventory_api_deploy`; with `github.com` the deploy key is never offered, GitHub sees an unauthenticated client, and a private repository reports `Repository not found` rather than revealing that it exists.

**2.** Clone through the alias:

```bash
git clone git@github-inventory-api:example-org/inventory-api.git /srv/inventory-api
```

If the repository was already cloned with the wrong URL, repoint it instead of re-cloning:

```bash
git remote set-url origin git@github-inventory-api:example-org/inventory-api.git
git fetch origin
```

**3.** No. A deploy key can be registered on only one repository across GitHub, so adding this public key to `stock-reports` is rejected. Generate a second key pair (`~/.ssh/stock_reports_deploy`), register it on `stock-reports`, and add a second block to `~/.ssh/config` with its own alias, for example `Host github-stock-reports`.

:::
::::

## Reference

- [Managing deploy keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys) — GitHub documentation; confirms that deploy keys are repository-specific and read-only unless write access is granted.
- [GitHub's SSH key fingerprints](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)

Once configured, the server retrieves code from the private `example-org/inventory-api` repository without storing a GitHub password or a personal access token anywhere on disk.
