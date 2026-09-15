# Security review — robcerda/monarch-mcp-server

**Commit reviewed:** `609d790bfafc6957ee824caa77b3be129c72ea4a` (2026-09-14)
**Date:** 2026-09-15
**Scope:** static review of this repo's source (51 Python files under `src/`). No runtime
execution, no dependency audit.
**Question asked:** is it safe to run with real financial credentials and to recommend to others?

## Verdict: safe with caveats

No exfiltration, no obfuscation, no shell-out, no plaintext password storage. The credential
at rest is a long-lived session token that sits in plaintext on disk (mode 0600) whenever the
OS keyring is unavailable, and the out-of-box posture is full write access to the account.

## Findings

### Medium — session token in plaintext on the file fallback (non-Windows)

`secure_session.py:317-326`. When `_keyring_available()` (`:116-151`) fails, the whole session
blob — long-lived token, `device_uuid`, and cookies including `session_id`/`csrftoken` — is
written as raw JSON to `~/.monarch-mcp-server/token`. DPAPI encryption is applied *only* on
Windows (`_dpapi_available()`, `:79-87`); on macOS/Linux `data = token` verbatim (`:322`).

Mechanics are correct: `_write_secret_file` (`:185-225`) opens with `os.open(..., 0o600)`
rather than write-then-chmod, so there is no permission window; it writes to a temp file and
`os.replace`s it; the directory is created and chmod'd to 0700 (`:309-315`). Impact is limited
to any process running as your user, and to any backup or sync of `$HOME`. The token grants
full read/write and does not expire — `monarch_auth.py:211-217` explicitly refuses to save
anything with a non-null `tokenExpiration`.

Worst in Docker: `Dockerfile:24-27` creates `/home/app/.monarch-mcp-server` and the image has
no keyring daemon, so the container path is *always* plaintext. Upstream documents this.

Quirk: `_keyring_available()` is evaluated once at import (`:300`, module singleton at `:732`),
so a keyring that becomes available later in the process lifetime is never picked up.

### Medium — read-only gate is a hand-maintained denylist

`read_only.py:27-69`, installed at `app.py:27-32`.

The enforcement is the strong part: `read_only.install()` monkeypatches `mcp.tool` *before* the
tools package is imported, so a gated tool is never registered and cannot be listed or called.
All 58 tools register through `@mcp.tool()` — no bare `@mcp.tool`, no `mcp.add_tool()` anywhere
in `src/`. The decorator handles the positional-name form (`read_only.py:99-100`), so
`@mcp.tool("delete_transaction")` cannot slip past.

The weakness is that `MUTATING_TOOLS` is an explicit 28-name set rather than derived. All 58
registered tools were cross-checked against it: **zero bypasses at this commit**. Residual risk
is future drift, partly covered by `tests/test_read_only.py:200-209`, which derives the writing
set from the AST and fails if a registered tool writes but is unlisted — a CI guard, not a
runtime one.

Read-only is **off by default** (`read_only.py:74-76`).

### Medium — destructive tools have no server-side confirmation

`tools/transactions.py:905-925` (`delete_transaction`) and `tools/rules.py:841-858`
(`delete_transaction_rule`) take an ID and execute immediately. No `dry_run`, no confirm flag.

`dry_run` exists on exactly four tools: `update_account` (`accounts.py:116`),
`upload_account_balance_history` (`accounts.py:332`), `update_category` (`categories.py:240`),
`bulk_categorize_transactions` (`transactions.py:825`). Upstream's README names those same four
and does not overclaim.

Mitigating: no `delete_account` or `delete_transaction_category` tool is exposed, even though
the upstream client has those methods. `update_category` requires an explicit
`confirm_rollover_reset` before a rollover-destroying change (`categories.py:327-337`).

### Low — `_cleanup_old_session_files` deletes files under CWD it did not create

`secure_session.py:692-728`. On every save and logout it unlinks `$HOME/.mm/mm_session.pickle`,
`$HOME/monarch_session.json`, and the same two paths under `Path.cwd()` (`:702-716`), plus
`rmdir`s `.mm` if empty. Intent is legitimate — the upstream library pickles a session to a
*relative* `.mm/` path, so a server started from a checkout leaves a plaintext token next to the
source. Narrowly scoped to three fixed filenames, all failures swallowed (`:727-728`), but it is
deletion outside the server's own directory, driven by wherever the MCP host set CWD.

### Low — `check_auth_status` echoes `MONARCH_EMAIL` into model context

`tools/auth.py:96-98`. Not a secret. `monarch_whoami` deliberately returns more (name, email,
`hasPassword`, external auth providers); comments at `tools/identity.py:20-35` show birthday and
profile picture were deliberately excluded. Reasonable tradeoff — but these land in transcripts.

### Low — `monarchmoneycommunity` unbounded in `pyproject.toml`

`pyproject.toml:31` declares `monarchmoneycommunity>=1.5.2` with no upper bound. The lockfiles do
pin it (`requirements-lock.txt:296` is `==1.5.2` with hashes), so `uv sync --locked` is safe — a
bare `pip install -e .` is not. This repo's `.mcp.json` pins it explicitly because `uvx` ignores
the upstream lockfile.

### Low — `python-dotenv` declared but never imported

`pyproject.toml:33`. Nothing in `src/` imports it. Dead dependency.

## Clean categories

**Network egress.** The only hosts in `src/` are `https://api.monarch.com`
(`monarch_auth.py:38`), `https://app.monarch.com` as an `Origin` header (`monarch_auth.py:72`),
and loopback literals for HTTP transport binding (`app.py:117-128`). No telemetry, analytics,
crash reporting, update check, webhook, or beacon. The only outbound request constructed in this
repo is the login POST at `monarch_auth.py:174-177`; everything else goes through the upstream
client. Exfiltration is ruled out for this repo's code.

**Dependency tree.** `uv.lock` resolves all 78 packages from `registry = "https://pypi.org/simple"`
— zero git, URL, or local-path sources. `requirements-lock.txt` carries 469 sha256 hashes. The one
non-obvious entry is `oathtool==2.4.0`, pulled in by `monarchmoneycommunity` for TOTP.

**Prompt-injection surface.** No `eval`, `exec`, `compile`, `__import__`, `pickle`, `marshal`,
`subprocess`, `os.system`, or `os.popen` anywhere in `src/`. Zero f-string or concatenated
GraphQL — every query is a module-level `gql()` constant invoked with `variables={...}`. The tools
package performs no filesystem I/O at all; the single hit across all 19 modules is
`os.getenv("MONARCH_EMAIL")` at `tools/auth.py:96`. No tool description instructs the model to act
on field contents (`tools/sync_health.py:76` refers to the server's own computed `needs_attention`
field, not bank-supplied text).

**Nothing alarming.** No obfuscation, no base64 blobs (the only `base64` use is the DPAPI blob at
`secure_session.py:102,109`). `pyproject.toml:1-4` is stock `setuptools.build_meta` — no
`setup.py`, no `cmdclass`, no postinstall. No reads of `~/.ssh`, browser profiles, other keychains,
or env dumps; the only paths touched are `~/.monarch-mcp-server/` (`secure_session.py:54-55`), the
cookie file (`login_setup.py:56-75`), and the three cleanup filenames. Dockerfile runs non-root as
uid 10001, uses `uv sync --locked --no-dev` and `UV_PYTHON_DOWNLOADS=0`, and ships no curl or wget.

## Not verified

- `monarchmoneycommunity==1.5.2` — not vendored, not installed, not read. This third-party fork
  makes every actual API call and is the largest unaudited surface.
- The other 77 dependencies. 35 advisories sit accepted in `.github/audit-baseline.txt`.
- No runtime execution as part of the review — no tests run, no live API call. Static reading only.
- Published GitHub/PyPI artifacts vs this source; no tag signatures checked.
- Intermediate history (203 commits) not audited for introduce-then-revert.
