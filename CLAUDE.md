# Monarch Skills

Claude Code skills for analyzing Monarch Money data, distributed as a plugin. They work with
either [robcerda/monarch-mcp-server](https://github.com/robcerda/monarch-mcp-server) (the
community server, preferred because it supports everything) or Monarch's official MCP connector.
The plugin bundles neither: each skill detects which one is connected.

## Before committing

```bash
./scripts/lint
```

Runs `claude plugin validate` on the manifests and skills, then the repo-specific checks.
It must exit 0.

A PR that changes `skills/` or `scripts/community-server.json` must also bump `version` in `.claude-plugin/plugin.json`, and CI
fails it otherwise. Bump when the pin moves or a skill's judgment changes. A doc-only edit
gets the `no-version-bump` label instead.

## Constraints

- **Skills report first, apply on request.** The read path must work under the read-only
  default, so a skill may only name a write tool under an `## Apply` section — read-only
  mode *unregisters* the 29 mutating tools rather than refusing them, and a skill that
  names one in its gather or judge steps sends Claude at a tool that does not exist.
  The apply section says the write needs `MONARCH_MCP_READ_ONLY=0` and stops if it is
  missing. In `references/official.md`, it needs write access on the connection instead.
  Lint enforces the placement for both servers; do not work around it.
- **`allowed-tools` pre-approves, it does not restrict.** List the specific read tools a
  skill calls. Never wildcard the server, or a user running with writes enabled gets
  mutations pre-approved without a prompt.
  List official connector tools under both `mcp__claude_ai_Monarch__` and `mcp__monarch__`,
  or users get a denial or a prompt on every read. Lint checks this.
- **Keep the `## Pick the server` block identical in every skill.** Community first, then
  official, then directions for both. Lint checks this.
- **Do not add a root `.mcp.json`.** The plugin auto-loads it, which bundles the server for
  everyone, including official connector users.
- **The community server is pinned to a full commit sha** in `scripts/community-server.json`,
  with upper bounds on `mcp` and `monarchmoneycommunity` because `uvx` ignores the upstream
  lockfile and resolves dependencies fresh on every launch. `scripts/setup-community` and
  `scripts/login` read that same pin, and the README's command repeats it. Bump all of it
  together, and re-review the server's source at the new sha when you do.

## Writing skills

The value of a skill here is the judgment and the tool quirks it encodes, not a
restatement of the tool list. Every gotcha in a SKILL.md was verified against a live
account — if you add one, verify it the same way rather than reasoning about what the API
probably does. Official connector write behavior that has not been run live says so.

**Never commit personal financial data.** No balances, account ids, institution names,
transaction details, or family members' names — including as an illustrative example in a
skill or a commit message. Write the gotcha generically.

## Layout

| Path | |
|---|---|
| `skills/*/SKILL.md` | the skills; plugin layout, so they only load once installed |
| `skills/*/references/official.md` | replaces Gather, Gotchas, and Apply on the official connector |
| `.claude/skills` | symlink to `skills/`, so they also load while working in the repo |
| `scripts/community-server.json` | pinned community server, read-only by default |
| `scripts/setup-community` | registers that server for the current directory, then logs in |
| `.claude-plugin/` | plugin and marketplace manifests |
| `scripts/login` | one-time auth; reads the same pin |
| `scripts/lint` | validator plus repo-specific checks |
| `scripts/check-version-bump` | CI check that skill and pin changes bump the plugin version |
| `evals/` | eval cases and synthetic mocks; must stay outside `skills/`, or `claude plugin eval` refuses to run them |
