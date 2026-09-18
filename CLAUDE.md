# Monarch Skills

Claude Code skills for analyzing Monarch Money data, distributed as a plugin that bundles
a pinned build of [robcerda/monarch-mcp-server](https://github.com/robcerda/monarch-mcp-server).

## Before committing

```bash
./scripts/lint
```

Runs `claude plugin validate` on the manifests and skills, then the repo-specific checks.
It must exit 0.

## Constraints

- **Skills report first, apply on request.** The read path must work under the read-only
  default, so a skill may only name a write tool under an `## Apply` section — read-only
  mode *unregisters* the 28 mutating tools rather than refusing them, and a skill that
  names one in its gather or judge steps sends Claude at a tool that does not exist.
  The apply section says the write needs `MONARCH_MCP_READ_ONLY=0` and stops if it is
  missing. Lint enforces the placement; do not work around it.
- **`allowed-tools` pre-approves, it does not restrict.** List the specific read tools a
  skill calls. Never wildcard the server, or a user running with writes enabled gets
  mutations pre-approved without a prompt.
- **The MCP server is pinned to a full commit sha** in `.mcp.json`, with upper bounds on
  `mcp` and `monarchmoneycommunity` because `uvx` ignores the upstream lockfile and
  resolves dependencies fresh on every launch. `scripts/login` reads that same pin.
  Bump all of it together, re-review the server's source at the new sha, and update the
  Security review section of `README.md` when you do.

## Writing skills

The value of a skill here is the judgment and the tool quirks it encodes, not a
restatement of the tool list. Every gotcha in a SKILL.md was verified against a live
account — if you add one, verify it the same way rather than reasoning about what the API
probably does.

**Never commit personal financial data.** No balances, account ids, institution names,
transaction details, or family members' names — including as an illustrative example in a
skill or a commit message. Write the gotcha generically.

## Layout

| Path | |
|---|---|
| `skills/*/SKILL.md` | the skills; plugin layout, so they only load once installed |
| `.claude/skills` | symlink to `skills/`, so they also load while working in the repo |
| `.mcp.json` | pinned server, read-only by default |
| `.claude-plugin/` | plugin and marketplace manifests |
| `scripts/login` | one-time auth; reads the pin from `.mcp.json` |
| `scripts/lint` | validator plus repo-specific checks |
