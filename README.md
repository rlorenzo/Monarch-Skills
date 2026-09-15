# Monarch Skills

An open-source collection of Claude Code skills for working with
[Monarch Money](https://www.monarchmoney.com/) data, plus a pinned, read-only-by-default
configuration of the MCP server they talk to.

The skills do not talk to Monarch directly. They call tools exposed by
[robcerda/monarch-mcp-server](https://github.com/robcerda/monarch-mcp-server), which this
repo pins to an exact commit.

## Install

Clone it and work inside it. That is the recommended setup:

```bash
git clone git@github.com:rlorenzo/Monarch-Skills.git
cd Monarch-Skills
claude
```

Claude Code picks up `.mcp.json` and prompts once to approve the server, and
`.claude/skills` symlinks the skills into place. Everything stays scoped to this
directory: the skills load here, the Monarch connection lives here, and neither follows
you into unrelated work.

That scoping is the point. These skills are useless outside a Monarch session and they
cost tokens in every session that loads them (~600 always-on for the five), so installing
them user-wide taxes every project you open in exchange for nothing. Editing a `SKILL.md`
here also takes effect immediately, which an install does not.

<details>
<summary>Installing as a plugin instead</summary>

Use this to get the skills in a *different* project without cloning this one — a personal
finance notes repo, say. Scope it to that project rather than to your user:

```
/plugin marketplace add rlorenzo/Monarch-Skills
/plugin install monarch-skills@monarch-skills
```

Or from a shell, where the scope is explicit:

```bash
claude plugin marketplace add rlorenzo/Monarch-Skills
claude plugin install monarch-skills@monarch-skills -s project
```

Installed skills are namespaced, so they appear as `/monarch-skills:monarch-doctor`. The
plugin brings the pinned MCP server with it, registered as `monarch-money`.

The slash-command form defaults to **user** scope, which is almost never what you want
here. A plugin install also serves skills from a cached clone, so local edits to this repo
do not show up in it.

</details>

## Authenticate

One-time:

```bash
./scripts/login
```

Choose **session cookies from browser** — long-lived, works with SSO, and avoids the Cloudflare
CAPTCHA that blocks programmatic password login. The session lands in your OS keyring, which is
what the server reads.

`scripts/login` runs upstream's `login_setup.py` at whatever commit `.mcp.json` pins, with the
same dependency bounds. The script is fetched from the pinned sha rather than vendored here, so
bumping the pin updates login too. Nothing is written to this repo: no password, no MFA secret,
no config to edit.

> The server also exposes `monarch_login` as an in-client tool, but read-only mode withholds it —
> writing a session counts as durable-state mutation (`read_only.py:57-62`). Logging in through the
> client would mean starting the server with `MONARCH_MCP_READ_ONLY=0`, which is why the script
> above is the recommended path.

`uvx` builds the pinned commit on first run (~8s) and starts from cache afterwards (~1.5s).
Requires [`uv`](https://docs.astral.sh/uv/).

## Read-only by default

Upstream ships with write access **on**. This repo flips that:

```json
"env": { "MONARCH_MCP_READ_ONLY": "${MONARCH_MCP_READ_ONLY:-1}" }
```

Read-only is enforced by never registering the 28 mutating tools, so they are absent from
the tool list rather than refused at call time — a model talked into a write by a merchant
name it read back has nothing to call. Verified: 28 tools withheld at startup.

To allow writes, set `MONARCH_MCP_READ_ONLY=0` in your own environment. Don't edit the
committed default, and keep client-side approval prompts on mutating tools.

## Version pinning

`.mcp.json` pins three things, because pinning only the first is not enough:

| Pin | Why |
|---|---|
| `@609d790…` (full commit sha) | uv skips the git fetch entirely; a bare `main` re-resolves HEAD on every launch |
| `mcp[cli]<3` | upstream declares `mcp>=1.10.0` unbounded; mcp 2.x already broke older commits at startup |
| `monarchmoneycommunity==1.5.2` | upstream declares it unbounded; this matches their lockfile |

`uvx` ignores the upstream `uv.lock`, so dependency bounds have to live here. Bump all
three together and re-run the security review when you do. Upstream publishes no tags or
releases, so the sha is pinning a snapshot of a moving branch.

## Lint

```bash
./scripts/lint
```

Runs `claude plugin validate` on the manifests and skills when the CLI is available,
then the checks it cannot know about: every skill's frontmatter `name` matches its
directory, no skill references a tool that read-only mode withholds (those tools are
never registered, so calling one fails at runtime rather than being refused), and
`.mcp.json` still pins a full commit sha with an `mcp` upper bound, which `scripts/login`
depends on, every skill carries a license, and no skill's `allowed-tools` pre-approves a
write tool.

## Security review

`609d790` was reviewed before pinning. Verdict: **safe with caveats**. Full notes in
[SECURITY-REVIEW.md](SECURITY-REVIEW.md). What matters if you install this:

- **Your session token is stored in plaintext when no OS keyring is available** (Docker,
  WSL, headless). It's `0600` in `~/.monarch-mcp-server/token`, but it's a long-lived
  token with full account read/write that never expires, and `monarch_logout` doesn't
  revoke it server-side.
- **The server deletes files under the current working directory** — three fixed session
  filenames it did not create, on every save and logout. Legitimate intent (the upstream
  client leaves a plaintext token in a relative `.mm/`), but worth knowing.
- **`delete_transaction` and `delete_transaction_rule` execute immediately** — no
  `dry_run`, no confirmation. Read-only mode withholds both.
- Identity tools echo your email and name into the transcript.

Clean on the things that would have been dealbreakers: no exfiltration (only
`api.monarch.com`), no telemetry, no obfuscation, no install hooks, no `eval`/`exec`/
`subprocess`, no string-built GraphQL, and no reads of unrelated files.

The review covered this repo's source only. `monarchmoneycommunity`, which makes every
actual API call, is a third-party fork and is the largest unaudited surface.

## Skills

All read-only: they report findings you act on in Monarch. None of them can recategorize
a transaction, change a budget, or cancel anything.

| Skill | What it does |
|---|---|
| `monarch-doctor` | Connections needing re-auth, stale or disconnected accounts, and what data they invalidate. Run this first — every other analysis is wrong in proportion to how long a connection has been dead. |
| `categorization-review` | Uncategorized transactions, likely miscategorizations, and auto-categorization rules to propose. |
| `budget-analyzer` | 6-12 months of budget vs actual: chronically over, chronically under, unbudgeted spending, and recommended amounts. |
| `cashflow-analyzer` | Spending trends plus anomalies worth investigating — spikes, duplicates, silent price hikes, possible fraud. |
| `subscription-manager` | Every recurring charge, normalized to monthly and annual cost, with cut and downgrade candidates. |

Each skill declares `allowed-tools` listing only the read tools it uses, so running one
does not stop for a permission prompt per call. Note that `allowed-tools` pre-approves
rather than restricts — it is an ergonomic setting, not a safety control, which is why
the lists name read tools explicitly instead of wildcarding the server.

Each skill encodes the quirks of this MCP server's tools, which is most of their value.
For instance `uncategorized_only` on `get_transactions_needing_review` filters one fetched
page locally, so it reports `count: 0, truncated: true` on an account that has plenty of
uncategorized transactions; `get_budgets`'s `remaining` is rollover-inflated and is not
`planned - actual`; and `get_transactions` errors unless given both dates.

## License

MIT
