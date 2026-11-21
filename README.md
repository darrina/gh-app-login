# gh-app-login

A [gh CLI](https://cli.github.com/) extension that logs in as a **GitHub App installation**.

`gh` has no built-in GitHub App auth. This extension does the two-step dance for you:

1. Mint a JWT (RS256, `iss` = the app's Client ID) signed with the app's private key.
2. Exchange it for an installation access token (`POST /app/installations/<id>/access_tokens`).

The token can then drive `gh` (and `git` over HTTPS) as the app bot, so commits and API
calls are attributed to `<app-name>[bot]` instead of a person.

## Requirements

- `bash`
- `openssl`
- `gh` CLI

No other dependencies.

## Install

```sh
gh extension install darrina/gh-app-login
```

Upgrade or remove:

```sh
gh extension upgrade app-login
gh extension remove app-login
```

## Setting up an app

`gh app-login` needs three things per app:

- the app's **Client ID**
- the app's **installation ID** for the account it is installed on
- the app's **private key** (a `.pem` file)

### Creating a new GitHub App

1. Open <https://github.com/settings/apps/new> (Settings, then Developer settings,
   then GitHub Apps, then **New GitHub App**).
2. Name the app and set a homepage URL (any URL works; the target repo is fine).
   Under **Webhook**, uncheck **Active** unless the app really needs events.
3. Under **Repository permissions**, grant only what the agent needs. For commit and
   PR work: **Contents** = *Read and write* and **Pull requests** = *Read and write*.
4. Set **Where can this GitHub App be installed?** to *Only on this account* and
   click **Create GitHub App**.
5. From the app's settings page, choose **Install App**, pick the account, and
   select only the repositories the agent should touch.

### Retrieving the credentials

- **Client ID**: shown at the top of the app's settings page
  (`https://github.com/settings/apps/<app-slug>`).
- **Private key**: on the app's settings page, go to **Credentials**, then
  **Key pairs**, and click **New key**. GitHub downloads a `.pem` file; this is the
  only time it is available. Store it somewhere secret, like a password manager or
  vault. **Never commit it, never paste it into chat.**
- **Installation ID**: open `https://github.com/settings/installations`, click
  *Configure* on the app; the ID is the last number in the URL.

## Usage

```sh
gh app-login --client-id <id> --installation-id <id> --key <path-to-pem> [--mode <mode>]
```

Modes:

- `--mode token` (default): print the raw installation token.
- `--mode export`: print an `export GH_TOKEN=...` line. Use with `eval` to scope the
  app token to the current shell:

  ```sh
  eval "$(gh app-login --client-id <client-id> --installation-id <installation-id> \
      --key ~/secrets/<app-slug>.pem --mode export)"
  ```

- `--mode auth`: persist the token with `gh auth login --with-token`, so every `gh`
  command runs as the app. This **replaces** gh's stored github.com credential; use
  `export` mode if you want to keep your normal login and scope the app token to one
  shell.

The installation token expires after **1 hour**. Re-run `gh app-login` to mint a
fresh one.

### Attributing commits to the app bot

Keep the installation token out of the remote URL. Putting a token in a URL stores it in plaintext in `.git/config` and can expose it in remote listings. `GH_TOKEN` is used by `gh` commands in the current shell. For Git over HTTPS, use the GitHub CLI credential helper so Git gets the token without embedding it in the remote.

The setup below scopes the helper configuration to this repository's `.git/config`, leaving your global Git configuration and normal `gh` login alone. The exported `GH_TOKEN` is available to `gh` and its Git credential helper only in this shell.

```sh
eval "$(gh app-login --client-id <client-id> --installation-id <installation-id> \
    --key ~/secrets/<app-slug>.pem --mode export)"
GIT_CONFIG_GLOBAL="$(git rev-parse --git-path config)" gh auth setup-git --hostname github.com --force
git remote set-url origin https://github.com/<owner>/<repo>.git
git config user.name "<app-slug>[bot]"
git push
git pull
```

Do not use `gh auth login --with-token` for this flow. That would persist the app token as a regular GitHub CLI login instead of keeping it in `GH_TOKEN` for the current shell.

## How it works

- JWT claims: `iat` = now - 60s (clock-skew allowance), `exp` = now + 600s,
  `iss` = Client ID. Signed RS256 with the private key via `openssl dgst -sha256 -sign`.
- Token exchange: `gh api --method POST /app/installations/<id>/access_tokens`.
- The private key never leaves your machine: it is only read locally by `openssl`.

## House rules for darrina repos

- Work on a branch, open a PR. Never commit directly to `main`.
- No secrets, keys or account data in any repo. `.gitignore` here already excludes `*.pem`.
