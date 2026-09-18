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

Use this to get the skills in a *different* project without cloning this one: a personal
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

Choose **session cookies from browser**. It is long-lived, works with SSO, and avoids the
Cloudflare CAPTCHA that blocks programmatic password login. The session lands in your OS keyring, which is
what the server reads.

`scripts/login` runs upstream's `login_setup.py` at whatever commit `.mcp.json` pins, with the
same dependency bounds. The script is fetched from the pinned sha rather than vendored here, so
bumping the pin updates login too. Nothing is written to this repo: no password, no MFA secret,
no config to edit.

> The server also exposes `monarch_login` as an in-client tool, but read-only mode withholds it,
> because writing a session counts as durable-state mutation (`read_only.py:57-62`). Logging in through the
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
the tool list rather than refused at call time. A model talked into a write by a merchant
name it read back has nothing to call. Verified: 28 tools withheld at startup.

To allow writes, set `MONARCH_MCP_READ_ONLY=0` yourself. Per launch:

```bash
MONARCH_MCP_READ_ONLY=0 claude
```

Or for every session in this directory, in `.claude/settings.local.json`, which git does
not track:

```json
{
  "env": { "MONARCH_MCP_READ_ONLY": "0" }
}
```

Claude Code puts that in the session environment and `.mcp.json` reads it through
`${MONARCH_MCP_READ_ONLY:-1}`. Restart, then check `/mcp`: 30 tools becomes 58.

Don't edit the committed default. It is what protects anyone who clones this and starts
running skills before reading anything. Leave the approval prompts on the write tools too.

Writes change what a skill can do, so turn them on deliberately. A skill's `## Apply`
section is inert while the tools are unregistered, so the read path is identical either
way. See [Skills](#skills).

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
then the checks it cannot know about:

- every skill's frontmatter `name` matches its directory, and it carries a license
- no skill's `allowed-tools` pre-approves a write tool
- no skill names a write tool outside its `## Apply` section. Those tools are never
  registered under the default, so a gather or judge step that calls one dies at runtime
- `.mcp.json` still pins a full commit sha with an `mcp` upper bound, which
  `scripts/login` depends on

## Security review

`609d790` was reviewed before pinning. Verdict: **safe with caveats**. Full notes in
[SECURITY-REVIEW.md](SECURITY-REVIEW.md). What matters if you install this:

- **Your session token is stored in plaintext when no OS keyring is available** (Docker,
  WSL, headless). It's `0600` in `~/.monarch-mcp-server/token`, but it's a long-lived
  token with full account read/write that never expires, and `monarch_logout` doesn't
  revoke it server-side.
- **The server deletes files under the current working directory.** It removes three fixed
  session filenames it did not create, on every save and logout. Legitimate intent (the upstream
  client leaves a plaintext token in a relative `.mm/`), but worth knowing.
- **`delete_transaction` and `delete_transaction_rule` execute immediately.** No
  `dry_run`, no confirmation. Read-only mode withholds both.
- Identity tools echo your email and name into the transcript.

Clean on the things that would have been dealbreakers: no exfiltration (only
`api.monarch.com`), no telemetry, no obfuscation, no install hooks, no `eval`/`exec`/
`subprocess`, no string-built GraphQL, and no reads of unrelated files.

The review covered this repo's source only. `monarchmoneycommunity`, which makes every
actual API call, is a third-party fork and is the largest unaudited surface.

## Skills

Every skill reports before it changes anything. On the read-only default, reporting is
all it can do, and you take the findings to Monarch yourself. With writes enabled, the
`## Apply` section at the end of a skill can make the fixes instead, but only the ones
you ask for, and every call still stops for approval.

Three skills have no `## Apply` section, because their findings are not things this API
can fix. Re-authenticating an institution is a browser flow, `monarch-doctor` can only
request a sync. Nothing cancels a subscription, so `monarch-subscription-manager` can at
most dismiss a stale recurring stream. And `monarch-cashflow-analyzer` reports anomalies
to investigate rather than changes to make.

| Skill | What it does |
|---|---|
| `monarch-doctor` | Connections needing re-auth, stale or disconnected accounts, and what data they invalidate. Run this first, because every other analysis is wrong in proportion to how long a connection has been dead. |
| `monarch-categorization-review` | Uncategorized transactions, likely miscategorizations, and auto-categorization rules to propose. |
| `monarch-budget-analyzer` | 6-12 months of budget vs actual: chronically over, chronically under, unbudgeted spending, and recommended amounts. |
| `monarch-cashflow-analyzer` | Spending trends plus anomalies worth investigating: spikes, duplicates, silent price hikes, possible fraud. |
| `monarch-subscription-manager` | Every recurring charge, normalized to monthly and annual cost, with cut and downgrade candidates. |
| `monarch-merchant-review` | One business recorded under two or more spellings, which splits its totals and can make a single subscription look like two. |

Each skill declares `allowed-tools` listing only the read tools it uses, so running one
does not stop for a permission prompt per call. Note that `allowed-tools` pre-approves
rather than restricts. It is an ergonomic setting, not a safety control, which is why
the lists name read tools explicitly instead of wildcarding the server.

Each skill encodes the quirks of this MCP server's tools, which is most of their value.
For instance `uncategorized_only` on `get_transactions_needing_review` filters one fetched
page locally, so it reports `count: 0, truncated: true` on an account that has plenty of
uncategorized transactions; `get_budgets`'s `remaining` is rollover-inflated and is not
`planned - actual`; and `get_transactions` errors unless given both dates.

## License

MIT
