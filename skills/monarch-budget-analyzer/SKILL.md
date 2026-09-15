---
name: monarch-budget-analyzer
description: Analyze Monarch Money budget performance over 6-12 months — which categories run over, under, or on plan — and recommend budget adjustments grounded in actual spending. Use when the user asks how they are tracking against budget, whether a budget is realistic, where they overspend, or wants help resetting budget amounts.
license: MIT
allowed-tools: mcp__monarch-money__get_budgets mcp__monarch-money__get_cashflow_by_month mcp__monarch-money__get_spending_summary mcp__monarch-money__get_accounts
---

# Budget analyzer

Read-only. You cannot change budget amounts; recommend numbers the user sets in Monarch.

## Gather

1. `get_budgets(start_date=..., end_date=...)` — one row per budgeted category per month:
   `name`, `planned`, `actual`, `remaining`, `category_group`, `month`. Pull the full
   window in one call; it accepts a multi-month range.
2. `get_cashflow_by_month(start_date=..., end_date=...)` — per-category monthly totals,
   to see the trend behind a variance and to catch spending in categories with no budget.
3. `get_spending_summary(start_date=..., end_date=...)` for window totals, income, and
   savings rate when the user wants the top-level picture.

Default window: last 6 full months. Use 12 when the user asks about a full year or when
you need to separate a seasonal pattern from a real trend. Exclude the current month from
variance verdicts — it is partial and will look under budget. Report it separately if at all.

## Gotchas

- **Do not trust `remaining`.** It reflects Monarch's rollover behavior, not
  `planned − actual`. A real row: Shopping planned 750, actual 218.78, remaining
  −14921.64. Compute `planned − actual` yourself and say that is what you did.
- **Most categories have `planned: 0`.** That means unbudgeted, not a $0 target. Never
  report a category as "over budget by $X" when planned is 0 — report it as unbudgeted
  spending, which is a different finding.
- **Income categories invert.** In `get_budgets`, Paychecks with planned 10500 and actual
  13014 means income came in *above* plan, which is good. Do not report it as overspending.
- **Transfers, Credit Card Payment, Buy/Sell are not spending.** They move money between
  the user's own accounts. Exclude them from budget verdicts or they swamp the analysis.
- Stale accounts understate actuals. If `get_accounts` shows `sync.state` of
  `needs_reauth` or `disconnected`, say which categories may be incomplete as a result.

## Analyze

Per budgeted category, across the window: months over, months under, average variance,
and the trend. Then sort into:

- **Chronically over** — over in most months. The budget is wrong, or the spending is.
  Say which, based on whether the overage is stable (budget too low) or growing (spending
  rising).
- **Chronically under** — under every month by a wide margin. Money is parked in a budget
  that does not need it.
- **Volatile** — swings wildly month to month. Often lumpy-but-real (insurance, travel,
  medical). An annual average is the honest number here, not a monthly target.
- **On track** — within roughly 10% most months. Say so briefly and move on.
- **Unbudgeted spending** — real spending where planned is 0. Often the biggest finding.

## Recommend

Only where the data supports it. For each: current planned, recommended planned, and the
actual numbers behind it (e.g. "median $612/mo over 6 months, never below $480").

Prefer the median over the mean for a target — one bad month should not reset a budget.
Call out one-offs you excluded and why.

If total recommended spending exceeds income, say so plainly and show the gap. Do not
quietly hand back a budget that does not balance.

## Report

Open with the verdict: total planned vs actual for the window, and whether they are
living within the budget overall. Then the category groups above, largest variance first.
Close with the recommended changes as a table: category, current, recommended, why.

Keep it to what the user can act on. A 70-row table of categories at 0 planned and 0
actual is not analysis.
