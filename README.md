# Monarch Skills

An open-source collection of Claude Code skills for working with
[Monarch Money](https://www.monarchmoney.com/) data.

The skills do not talk to Monarch directly. They call an MCP server, and detect which one you
have connected. The plugin bundles neither, so you only run the one you choose.

## Install

Clone it and work inside it. That is the recommended setup:

```bash
git clone git@github.com:rlorenzo/Monarch-Skills.git
cd Monarch-Skills
claude
```

`.claude/skills` symlinks the skills into place. Everything stays scoped to this directory:
the skills load here, and neither they nor a server you connect here follow you into unrelated work.

That scoping is the point. These skills are useless outside a Monarch session and they
cost tokens in every session that loads them (~700 always-on for the seven), so installing
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

Installed skills are namespaced, so they appear as `/monarch-skills:monarch-doctor`.

The slash-command form defaults to **user** scope, which is almost never what you want
here. A plugin install also serves skills from a cached clone, so local edits to this repo
do not show up in it.

> **Upgrading from 1.x:** the plugin no longer brings a server with it, so connect one below.
> Your existing community server login stays in your OS keyring.

</details>

## Connect a Monarch server

Connect either one. If both are connected, skills use the community server.

| | Community server | Official connector |
|---|---|---|
| Source | [robcerda/monarch-mcp-server](https://github.com/robcerda/monarch-mcp-server), third party, private API | Monarch, hosted |
| Sign-in | Session cookies in your OS keyring | OAuth |
| Connection health (`monarch-doctor`) | Full | Reduced: infers staleness only |
| Writing budgets | Yes | No |
| Review queue matching | Statement text | Merchant, statement text for 20 rows at a time |
| Finicity-linked accounts | Included | Hidden, and their review queue items cannot be read |
| Limits | None known | Weekly usage cap, 100 transactions per page |

### Official connector

```bash
claude mcp add --transport http monarch https://api.monarch.com/mcp
```

Then run `/mcp`, select `monarch`, and sign in. If you already have Monarch as a claude.ai
connector, sign in to that instead. Choose read-only unless you want skills to be able to make
changes. If you use the command above, keep the server name `monarch`. Any other name means a permission prompt on every read, and skills only detect a name containing "monarch".

### Community server

From a clone, in the directory where you want it available:

```bash
./scripts/setup-community
```

Without a clone, register it yourself from the project directory:

```bash
claude mcp add-json monarch-money '{"type":"stdio","command":"uvx","args":["--with","mcp[cli]<3","--with","monarchmoneycommunity==1.5.2","--from","git+https://github.com/robcerda/monarch-mcp-server@5cc005783086ef232cdb967a5259b4cfa78d05f4","monarch-mcp-server"],"env":{"MONARCH_MCP_READ_ONLY":"${MONARCH_MCP_READ_ONLY:-1}"}}'
```

Then log in with `scripts/login` from a clone. Choose **session cookies from browser**. It is
long-lived, works with SSO, and avoids the Cloudflare CAPTCHA that blocks programmatic password
login. The session lands in your OS keyring, which is what the server reads.

`scripts/login` runs upstream's `login_setup.py` at whatever commit `scripts/community-server.json`
pins, with the same dependency bounds. The script is fetched from the pinned sha rather than
vendored here, so bumping the pin updates login too. Nothing is written to this repo: no password,
no MFA secret, no config to edit.

> The server also exposes `monarch_login` as an in-client tool, but read-only mode withholds it,
> because writing a session counts as durable-state mutation (`read_only.py:57-62`). Logging in through the
> client would mean starting the server with `MONARCH_MCP_READ_ONLY=0`, which is why the script
> above is the recommended path.

`uvx` builds the pinned commit on first run (~8s) and starts from cache afterwards (~1.5s).
Requires [`uv`](https://docs.astral.sh/uv/).

**Your registered pin does not update itself.** The skills' gotchas are verified against the commit
above. To move to a newer pin, pull and re-run `./scripts/setup-community`, or re-run the command
above after removing the old entry with `claude mcp remove monarch-money`.

## Read-only by default

On the official connector, write access is the scope you approve at sign-in. Review or revoke it in
Monarch under Settings → Integrations. Monarch cannot undo changes made through it.

The rest of this section covers the community server. Upstream ships with write access **on**.
This repo flips that:

```json
"env": { "MONARCH_MCP_READ_ONLY": "${MONARCH_MCP_READ_ONLY:-1}" }
```

Read-only is enforced by never registering the 29 mutating tools, so they are absent from
the tool list rather than refused at call time. A model talked into a write by a merchant
name it read back has nothing to call. Verified: 29 tools withheld at startup.

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

Claude Code puts that in the session environment and the registered server reads it through
`${MONARCH_MCP_READ_ONLY:-1}`. Restart, then check `/mcp`: 30 tools becomes 59.

Don't edit the committed default. It is what protects anyone who clones this and starts
running skills before reading anything. Leave the approval prompts on the write tools too.

Writes change what a skill can do, so turn them on deliberately. A skill's `## Apply`
section is inert while the tools are unregistered, so the read path is identical either
way. See [Skills](#skills).

## Version pinning

`scripts/community-server.json` pins three things, because pinning only the first is not enough:

| Pin | Why |
|---|---|
| `@5cc0057…` (full commit sha) | uv skips the git fetch entirely; a bare `main` re-resolves HEAD on every launch |
| `mcp[cli]<3` | upstream declares `mcp>=1.10.0` unbounded; mcp 2.x already broke older commits at startup |
| `monarchmoneycommunity==1.5.2` | upstream declares it unbounded; this matches their lockfile |

`uvx` ignores the upstream `uv.lock`, so dependency bounds have to live here. Bump all
three together, update the README command, and re-review the server's source at the new sha when you do. Upstream
publishes no tags or releases, so the sha is pinning a snapshot of a moving branch.

## Lint

```bash
./scripts/lint
```

Runs `claude plugin validate` on the manifests and skills when the CLI is available,
then the checks it cannot know about:

- every skill's frontmatter `name` matches its directory, and it carries a license
- no skill's `allowed-tools` pre-approves a write tool
- no skill or official reference names a write tool outside its `## Apply` section. Those
  tools are never registered under the default, so a gather or judge step that calls one dies at runtime
- every skill has the same `## Pick the server` block, a `references/official.md`, and
  pre-approvals for each server
- `scripts/community-server.json` still pins a full commit sha with an `mcp` upper bound, the
  README command uses it, and no root `.mcp.json` has come back

CI also fails a PR that changes `skills/` or `scripts/community-server.json` without changing `version` in
`.claude-plugin/plugin.json`, because plugin users get nothing until that field moves. For a doc-only
edit to a skill, add the `no-version-bump` label.

## Evals

```bash
claude plugin eval . --model sonnet --ablation none --no-publish
```

The cases in `evals/` run against the invented account in `evals/mocks/monarch-money/`, so they need no Monarch login and the real server never starts. Each run calls a model on your Claude credential.

Keep the mocks small. Larger results are saved to a folder the eval sandbox cannot read, which is also why these cases don't use a live account. Don't pass `--mocks off`: it runs against your real account, and the results can hold your financial data.

## Skills

Every skill reports before it changes anything. On the read-only default, reporting is
all it can do, and you take the findings to Monarch yourself. With writes enabled, the
`## Apply` section of a skill can make the fixes instead, but only the ones
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
| `monarch-transactions-review` | The review queue, including large backlogs: what distorts the numbers, possible duplicates, and what is safe to mark reviewed. |
| `monarch-budget-analyzer` | 6-12 months of budget vs actual: chronically over, chronically under, unbudgeted spending, and recommended amounts. |
| `monarch-cashflow-analyzer` | Spending trends plus anomalies worth investigating: spikes, duplicates, silent price hikes, possible fraud. |
| `monarch-subscription-manager` | Every recurring charge, normalized to monthly and annual cost, with cut and downgrade candidates. |
| `monarch-merchant-review` | One business recorded under two or more spellings, which splits its totals and can make a single subscription look like two. |

Each skill declares `allowed-tools` listing only the read tools it uses, so running one
does not stop for a permission prompt per call. Note that `allowed-tools` pre-approves
rather than restricts. It is an ergonomic setting, not a safety control, which is why
the lists name read tools explicitly instead of wildcarding the server.

Each list names community tools as `mcp__monarch-money__*` and official tools twice, as
`mcp__claude_ai_Monarch__*` and `mcp__monarch__*`, since the connector's prefix depends on how it
was installed. Lint checks that both are there.

Each skill encodes the quirks of its server's tools, which is most of their value.
On the community server, for instance, `uncategorized_only` on `get_transactions_needing_review` filters one fetched
page locally, so it reports `count: 0, truncated: true` on an account that has plenty of
uncategorized transactions; `get_budgets`'s `remaining` is rollover-inflated and is not
`planned - actual`; and `get_transactions` errors unless given both dates.

## License

MIT
