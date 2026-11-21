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

## One-time setup per app

1. Open the app's settings page (`https://github.com/settings/apps/<app-slug>`) and
   select **Generate a private key**. GitHub downloads a `.pem` file.
2. Store the `.pem` somewhere secret: a password manager or vault.
   **Never commit it, never paste it into chat.**
3. Note the app's **Client ID** (top of the same settings page) and the
   **installation ID** (open `https://github.com/settings/installations`, click
   *Configure* on the app; the ID is the last number in the URL).

### Apps in the darrina account

| Agent            | App             | Client ID             | Installation ID | Installed on          |
| ---------------- | --------------- | --------------------- | --------------- | --------------------- |
| Sluggo (Instinct) | `instinct-app`  | `Iv23li59qRQP3gREuVaL` | `184348145`     | `darrina/santi-systems` |
| Marco (Muse)     | `marco-muse-app` | `Iv23liandSy4nz2MQhbw` | `184348020`     | `darrina/santi-systems` |

These identifiers are not secrets, but they are useless without each app's private key.

## Usage

```sh
gh app-login --client-id <id> --installation-id <id> --key <path-to-pem> [--mode <mode>]
```

Modes:

- `--mode token` (default): print the raw installation token.
- `--mode export`: print an `export GH_TOKEN=...` line. Use with `eval` to scope the
  app token to the current shell:

  ```sh
  eval "$(gh app-login --client-id Iv23liandSy4nz2MQhbw --installation-id 184348020 \
      --key ~/secrets/marco-muse-app.pem --mode export)"
  ```

- `--mode auth`: persist the token with `gh auth login --with-token`, so every `gh`
  command runs as the app. This **replaces** gh's stored github.com credential; use
  `export` mode if you want to keep your normal login and scope the app token to one
  shell.

The installation token expires after **1 hour**. Re-run `gh app-login` to mint a
fresh one.

### Attributing commits to the app bot

With `GH_TOKEN` set (or `--mode auth` done), git pushes over HTTPS authenticate as the
app. Set the commit author to match:

```sh
git config user.name "marco-muse-app[bot]"
git remote set-url origin https://x-access-token:${GH_TOKEN}@github.com/darrina/santi-systems.git
```

## How it works

- JWT claims: `iat` = now - 60s (clock-skew allowance), `exp` = now + 600s,
  `iss` = Client ID. Signed RS256 with the private key via `openssl dgst -sha256 -sign`.
- Token exchange: `gh api --method POST /app/installations/<id>/access_tokens`.
- The private key never leaves your machine: it is only read locally by `openssl`.

## House rules for darrina repos

- Work on a branch, open a PR. Never commit directly to `main`.
- No secrets, keys or account data in any repo. `.gitignore` here already excludes `*.pem`.
