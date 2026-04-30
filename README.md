# CodexAccountSwitcher

`CodexAccountSwitcher` provides the `codex-multi-auth` command: a small local wrapper around [`codex-auth`](https://github.com/Loongphy/codex-auth) for managing multiple ChatGPT OAuth accounts with the Codex Mac App.

It is designed for people who use the Codex Mac App as the main UI but want lower-friction account switching from the terminal.

## What it does

- Captures the current Codex ChatGPT OAuth session into stable slot aliases such as `slot-01`.
- Switches the active `~/.codex/auth.json` through `codex-auth`.
- Pauses `codex-auth` auto-switch before manual switches so manual selection stays sticky.
- Fully restarts the Codex Mac App after switching and waits for the local app-server to stay connected.
- Rotates expired or near-expired ChatGPT OAuth access tokens using the stored local refresh token.
- Shows slot usage with a best-effort live refresh from the local account snapshots.

## What it does not do

- It is not an official OpenAI product.
- It does not create extra quota or bypass plan limits.
- It does not make the Codex Mac App hot-swap accounts without a full restart.
- It does not upload or sync your auth snapshots anywhere.

## Install

Prerequisites:

- macOS with the Codex Mac App installed at `/Applications/Codex.app`
- [Bun](https://bun.sh/)
- `codex-auth` installed and working

Install `codex-auth` first:

```sh
npm i -g @loongphy/codex-auth
```

Install this tool from the GitHub checkout:

```sh
git clone https://github.com/bozliu/CodexAccountSwitcher.git
cd CodexAccountSwitcher
chmod +x bin/codex-multi-auth
ln -sf "$PWD/bin/codex-multi-auth" ~/.local/bin/codex-multi-auth
```

Make sure `~/.local/bin` is on your `PATH`, or run the command directly with:

```sh
bun bin/codex-multi-auth help
```

## Usage

Capture the account currently logged in to Codex:

```sh
codex-multi-auth setup slot-01
```

Log in and capture additional slots:

```sh
codex-multi-auth login slot-02
codex-multi-auth login slot-03
```

List stored slots:

```sh
codex-multi-auth slots
```

`slots` first makes expiring OAuth tokens fresh, then refreshes usage. Use cached mode when you only want saved values and no network or file writes:

```sh
codex-multi-auth slots --cached
```

Refresh tokens and usage without switching accounts:

```sh
codex-multi-auth refresh --all
codex-multi-auth refresh slot-04
```

Inspect token freshness without writing anything:

```sh
codex-multi-auth doctor auth
```

Switch to a slot:

```sh
codex-multi-auth switch slot-02
```

The switch command restarts the Codex Mac App because the desktop app does not reliably pick up replaced credentials without a full relaunch.

## Auto-switching

Manual switching always pauses `codex-auth` auto-switch first. This prevents a background watcher from racing against your explicit choice.

If you want to enable automation again, do it explicitly:

```sh
codex-multi-auth auto on --local
```

API-backed auto-switching is intentionally gated:

```sh
codex-multi-auth auto on --api --yes-risk
```

Use this only if you understand the policy and stability risks of the upstream `codex-auth` experimental mode.

## Usage freshness

By default, `codex-multi-auth slots` makes the local OAuth token usable before it tries direct usage refresh:

- It decodes the access token expiry in each local snapshot.
- It refreshes only when the token is expired or within five minutes of expiry.
- It writes the rotated token back to that slot snapshot.
- If the refreshed slot is the active account, it also updates `~/.codex/auth.json`.
- It uses per-slot lock files so two processes do not consume the same rotating refresh token at the same time.

This is designed to fix the common failure mode where usage refresh returns `401` because a stored access token expired.

This is not an official public API contract. If refresh fails, the command falls back to cached registry data and reports that clearly. Use cached mode when you only want the saved snapshot:

```sh
codex-multi-auth slots --cached
```

If the server rejects a refresh token as expired, reused, invalidated, or otherwise unauthorized, OAuth security rules still require a fresh login for that slot:

```sh
codex-multi-auth login slot-04
```

## Security

Treat every file under `~/.codex/accounts/` as password-equivalent.

Do not commit, sync, paste, upload, or share:

- `~/.codex/auth.json`
- `~/.codex/accounts/`
- `*.auth.json`
- `registry.json`
- backups of any of the above

This repository's `.gitignore` excludes those paths by default, but you should still run your own secret scan before publishing forks or patches.

## Configuration

The tool uses `CODEX_HOME` when set, otherwise it defaults to `~/.codex`.

```sh
CODEX_HOME=/path/to/codex-home codex-multi-auth status
```

## License

MIT
