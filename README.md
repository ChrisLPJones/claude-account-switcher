# claude-acct

Hot-swap the Claude Code login **while Claude Code is running** — no restart, no re-login.

On Linux, Claude Code stores the active OAuth login in `<config>/.credentials.json`
and re-reads it whenever the file's mtime changes. `claude-acct` keeps a small store
of saved accounts and swaps that file (plus the `oauthAccount` identity block in
`~/.claude.json`) in place, so switching accounts takes effect on the next message.

It can also watch your usage and automatically rotate to another account when a
5-hour or 7-day limit gets close, and print a one-line status for a tab bar.

## Why it's careful

Refresh tokens **rotate**: while an account is active, Claude Code refreshes its
token and the previous refresh token dies. So before swapping an account out,
`claude-acct` always copies the live credentials back into its slot — otherwise the
slot would hold a dead token and the next switch to it would fail.

Swaps are done under Claude Code's own `proper-lockfile` locks (the mkdir-based
`.oauth_refresh.lock` and config lock), so a switch can never land in the middle of
a token refresh. Writes are atomic (`tempfile` + `os.replace`) and files are chmod
`0600`.

## Install

```sh
# single-file script, Python 3 standard library only — no dependencies
install -m 755 claude-acct ~/.local/bin/claude-acct   # or anywhere on $PATH
```

## Usage

```
claude-acct save <name>        store the currently logged-in account as <name>
claude-acct switch [name]      switch to <name> (no name: rotate to the next)
claude-acct list               show stored accounts, mark the active one
claude-acct status             show the active account
claude-acct remove <name>      delete a stored account
claude-acct auto [--threshold 90] [--interval 120] [--once]
                               watch usage; switch when 5h or 7d hits threshold
claude-acct bar                one-line status (active account + cached usage)
```

### Adding accounts

1. Log in to Claude Code with your first account, then:
   ```sh
   claude-acct save personal
   ```
2. In Claude Code run **`/login`** (not `/logout`) and sign in with the second
   account, then:
   ```sh
   claude-acct save work
   ```

Now switch any time:

```sh
claude-acct switch work     # or just `claude-acct switch` to rotate
```

The switch takes effect on Claude Code's next message (the first one may rebuild
the prompt cache).

### Auto-rotate on usage

```sh
claude-acct auto --threshold 90 --interval 120
```

Checks every account each round, caches the readings (used by `bar`), and when the
active account crosses the threshold on its 5-hour *or* 7-day window, switches to
another account that still has headroom. It only reads usage from an unexpired
access token and never refreshes one itself, so it can't rotate a token out from
under Claude Code.

### Status bar

```sh
claude-acct bar
# ● Work  5h ▰▰▱▱▱ 34%  7d ▰▰▰▰▱ 71% ┃ ○ Personal  5h ▰▱▱▱▱ 12%  7d ▰▱▱▱▱ 8%
```

Reads the cache only (no network), marks the active account with `●`.

## Where things live

| Path | What |
|------|------|
| `~/.claude-accounts/<name>/` | a saved account (`credentials.json` + `oauthAccount.json`) |
| `~/.claude-accounts/.usage.json` | cached usage readings |
| `<config>/.credentials.json` | the live login Claude Code reads |
| `~/.claude.json` | holds the active `oauthAccount` identity block |

`<config>` is `$CLAUDE_CONFIG_DIR` when set, otherwise `~/.claude`.

## Notes

- Linux only (paths and the mtime-reload behaviour).
- Credentials never leave your machine; the only network call is the usage API
  (`api.anthropic.com/api/oauth/usage`) using an already-issued access token.

## License

MIT — see [LICENSE](LICENSE).
