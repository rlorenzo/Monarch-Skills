---
name: monarch-cashflow-analyzer
description: Analyze Monarch Money cash flow for spending trends and anomalies — unusual charges, spikes, duplicate transactions, and patterns worth investigating for fraud, billing errors, or creeping costs. Use when the user asks about spending trends, unusual or suspicious transactions, where their money went, or wants a check for fraud and billing mistakes.
license: MIT
allowed-tools: mcp__monarch-money__get_cashflow_by_month mcp__monarch-money__get_spending_summary mcp__monarch-money__get_transactions mcp__monarch-money__search_transactions mcp__monarch-money__get_accounts
---

# Cashflow analyzer

Read-only. Surface what looks wrong; the user investigates and acts.

## Gather

1. `get_cashflow_by_month(start_date=..., end_date=...)` — per-category monthly totals.
   This is the trend backbone.
2. `get_spending_summary(start_date=..., end_date=...)` — totals by category, group, and
   merchant, plus income, expenses, savings rate.
3. `get_transactions(start_date=..., end_date=...)` for the periods or categories where a
   trend or anomaly needs explaining at the transaction level. Page with `limit`/`offset`.
4. `search_transactions(search="<merchant>", ...)` to pull one merchant's full history when
   testing whether a charge is normal for them.

Default window: last 6 months, so a month has five peers to be judged against. Use 12
months when checking for annual charges or seasonality.

## Gotchas

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
