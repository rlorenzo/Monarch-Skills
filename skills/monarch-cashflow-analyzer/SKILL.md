---
name: monarch-cashflow-analyzer
description: Analyze Monarch Money cash flow for spending trends and anomalies — unusual charges, spikes, duplicate transactions, and patterns worth investigating for fraud, billing errors, or creeping costs. Use when the user asks about spending trends, unusual or suspicious transactions, where their money went, or wants a check for fraud and billing mistakes.
license: MIT
allowed-tools: mcp__monarch-money__check_auth_status mcp__monarch-money__get_cashflow_by_month mcp__monarch-money__get_spending_summary mcp__monarch-money__get_transactions mcp__monarch-money__search_transactions mcp__monarch-money__get_accounts mcp__claude_ai_Monarch__GetCashFlow mcp__claude_ai_Monarch__GetTransactions mcp__claude_ai_Monarch__GetAccounts mcp__monarch__GetCashFlow mcp__monarch__GetTransactions mcp__monarch__GetAccounts
---

# Cashflow analyzer

Read-only. Surface what looks wrong; the user investigates and acts.

## Pick the server

Do this first, and name the server you used in the first line of the report.

1. **Community server** (robcerda/monarch-mcp-server): if `check_auth_status` is available
   and reports a session, follow this file as written. If it reports no session, or the
   first read fails on auth, treat this server as not connected.
2. **Official connector** (Monarch's hosted MCP): otherwise, if a server with "monarch" in its
   name, in any case, has a tool ending in `__GetTransactions`, read `references/official.md` in
   this skill's directory.
   It replaces Gather, Gotchas, and Apply below, and everything else still applies using the
   field names it gives. Without sign-in the connector only exposes `authenticate` tools,
   so it is not connected.
3. **Neither:** stop. Tell the user no Monarch server is connected and that either one works:
   - Official connector: `claude mcp add --transport http monarch https://api.monarch.com/mcp`,
     or add it as a claude.ai connector, then sign in through `/mcp`. It has fewer features.
   - Community server: supports everything, and needs `uv`.

   Link both to <https://github.com/rlorenzo/Monarch-Skills#connect-a-monarch-server>.

If both are connected, use the community server. Do not switch servers once a read succeeds.

## Gather

1. `get_cashflow_by_month(start_date=..., end_date=...)` — per-category monthly totals.
   This is the trend backbone. Rows carry only `category_id`, so take the names from
   `by_category.rows` in step 2, using the same dates.
2. `get_spending_summary(start_date=..., end_date=...)` — totals by category, group, and
   merchant, plus income, expenses, savings rate.
3. `get_transactions(start_date=..., end_date=...)` for the periods or categories where a
   trend or anomaly needs explaining at the transaction level. Page with `limit`/`offset`.
4. `search_transactions(search="<merchant>", ...)` to pull one merchant's full history when
   testing whether a charge is normal for them.

Default window: last 6 months, so a month has five peers to be judged against. Use 12
months when checking for annual charges or seasonality.

## Gotchas

- **`get_spending_summary` breakdowns are objects, not lists.** `by_category`,
  `by_category_group`, and `by_merchant` each hold `rows`, `total`, `returned`, and `truncated`.
  A long merchant list is trimmed to the largest rows to fit the response, so check `truncated`.
  `limit=0` turns trimming off, but a wide window can then exceed the response limit.
- **`get_transactions` includes hidden transactions by default.** Rows carry
  `hide_from_reports`, and Monarch's reports leave those out, so sums taken from rows can run
  high. Pass `include_hidden=False` to leave them out.
- **`get_transactions` needs both dates.** Only one returns an error, not a partial result.
- **Amounts are signed**: expenses negative, income positive. A positive amount in an
  expense category is normally a refund.
- **Exclude transfers before judging spending.** `Transfer`, `Credit Card Payment`,
  `Buy`, `Sell`, `Balance Adjustments` move money between the user's own accounts. A
  $2,000 transfer is not a $2,000 spike.
- **Stale accounts fake a decline.** Check `get_accounts` for `sync.state` of
  `needs_reauth` / `disconnected` first. A category that "dropped to zero" is usually a
  dead connection, not changed behavior — say so instead of reporting a fake trend.
- A partial current month always looks like a decline. Exclude it or label it.

## Find trends

Per category, month over month: direction, size, and whether it is a step change (new
recurring cost) or a drift (gradual creep). Report the ones that are large in dollars,
not large in percent — a 300% jump on a $4 category is noise.

Separate seasonal from structural where the window lets you. Say which you cannot tell
apart with the data you have.

## Find anomalies

Judge each against that merchant's and category's own history, not a flat threshold:

- A charge several times the normal amount for that merchant.
- A first-ever charge from a new merchant that is large.
- Same merchant, same or near-same amount, same or adjacent day — a likely duplicate.
- A round-number charge where that merchant normally bills odd amounts.
- Charges in a category that is normally dormant.
- A recurring charge that silently increased — the classic subscription price hike.
- Out-of-pattern geography or channel visible in the description.
- Small unfamiliar charges, which are how card testing starts before a large one.

Rank by how much the finding would cost if it is real, not by how anomalous it looks.

## Report

Three sections, each ordered by dollars:

1. **Worth investigating now** — possible fraud, duplicates, unexpected large charges.
   Date, merchant, amount, account, and what specifically is off about it.
2. **Trends** — category, direction, monthly figures, and the likely driver.
3. **Creeping costs** — recurring charges that rose, with old and new amounts.

For each anomaly give the comparison that makes it an anomaly ("$340 at a merchant whose
last six charges averaged $28"). Without the baseline it is an accusation, not a finding.

Say plainly when something is probably fine but unusual enough to mention. Do not
manufacture findings to fill a section — "nothing anomalous this window" is a real result.
