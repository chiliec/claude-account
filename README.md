# claude-account

[![ci](https://github.com/chiliec/claude-account/actions/workflows/ci.yml/badge.svg)](https://github.com/chiliec/claude-account/actions/workflows/ci.yml)

Switch between multiple authorized Claude Code logins on macOS, without re-authorizing every time.

Claude Code stores exactly one login. `/login` with a second account silently overwrites the
first, and getting back means a full browser round-trip. `claude-account` keeps a copy of each
authorized account and swaps them in on demand.

```
$ claude-account
 1) * personal          me@gmail.com
 2)   work             me@acme.com
 3)   client           me@client.io
account [1-3]: 2
switched to 'work' (me@acme.com) — start a new Claude Code session
```

## Install

One file, no dependencies beyond what macOS already ships (`security`, `jq`, bash).

```bash
curl -fsSL https://raw.githubusercontent.com/chiliec/claude-account/main/claude-account \
  -o /usr/local/bin/claude-account
chmod +x /usr/local/bin/claude-account
```

Any directory on your `PATH` works — `~/.local/bin`, `~/.claude/bin`, wherever you keep things.

## Usage

```bash
claude-account              # interactive picker; * marks the account in use
claude-account list
claude-account save [name]  # snapshot whoever is logged in right now
claude-account use work
claude-account rm work
claude-account selftest     # round-trip check against throwaway state
```

Adding an account is a one-time cost:

```bash
claude /login               # authorize the new account in the browser
claude-account save work    # capture it
```

From then on, switching is offline.

## How it works

A Claude Code login is two pieces of state:

| Where | What |
|---|---|
| Keychain item `Claude Code-credentials` | the OAuth token blob |
| `~/.claude.json` → `oauthAccount`, `userID` | the identity the client reports |

`/login` overwrites both. `save` copies the token to a parallel keychain item
(`Claude Code-credentials:<name>`) and writes the non-secret identity block to
`~/.claude/accounts/<name>.json` (mode `600`). `use` puts them back.

Tokens never leave the keychain — only the identity block (email, org name, account UUID)
is written to disk.

## Things worth knowing

- **Switching applies to new sessions.** A running Claude Code process keeps the token it
  started with.
- **Tokens rotate.** Claude Code refreshes the live credential periodically, so a profile
  you saved months ago may be stale. Re-run `save` while logged in as that account to
  refresh it. If a stale profile fails to authorize, `claude /login` + `save` fixes it.
- **Nothing is stranded.** `use` auto-snapshots the currently live account first if it
  isn't stored yet, so switching can never lose a login you never explicitly saved.
- **Keychain prompts.** macOS may ask you to authorize `security` the first time it reads
  or writes these items. "Always Allow" makes it quiet.
- **The token passes through argv** on write (`security add-generic-password -w "$token"`).
  `security`'s stdin prompt truncates at 128 characters and the credential blob is ~1.4 KB,
  so there is no stdin path. Exposure is to same-user `ps` for the duration of one exec —
  the same trust boundary as reading the keychain item at all. Every write is read back and
  compared before being trusted.

## Environment overrides

Mostly for testing; the defaults are right for a normal install.

| Variable | Default |
|---|---|
| `CLAUDE_KEYCHAIN_SERVICE` | `Claude Code-credentials` |
| `CLAUDE_ACCOUNTS_DIR` | `~/.claude/accounts` |
| `CLAUDE_CONFIG_FILE` | `~/.claude.json` |

## Scope

macOS only — it talks to the login keychain via `security`. On Linux, Claude Code keeps
credentials in `~/.claude/.credentials.json`, where swapping accounts is a file copy and
this tool isn't needed.

Unofficial, not affiliated with Anthropic. It reads and writes your own local credential
store and talks to nothing over the network.

## License

MIT
