# Official connector

These replace Gather, Gotchas, and Apply in the main file.

## Gather

1. `GetBudget(start_date=..., end_date=..., include_actuals=true)` returns one row per category
   per month, with `category`, `category_group`, `category_type`, `month`, `budget_amount`, and
   `actual_amount`. It accepts a multi-month range. Six months can be too large to return inline
   and land in a file, with the JSON inside a `result` string. `totals_overall` has the window's
   budgeted and actual income and expenses.
2. `GetCashFlow(base_query='{"group_by_time":"month","group_by_entity":"category"}', ...)` for the
   trend behind a variance and for spending in unbudgeted categories. It also returns the window's
   `total_income`, `total_expense`, and `net_savings_rate`.
3. `GetAccounts` once, for `finicity_excluded`. The budget and cash flow calls did not return it.

Use the default window and current-month rule from the main file.

## Gotchas

- **Every data call needs `intent`.** Pick the closest listed value. If it is `other`,
  `intent_description` names the task in a few generic words, never personal details.
- **Report `finicity_excluded` whenever it appears.** Accounts linked through Finicity are
  hidden from results. Its count and institutions change from call to call, and some calls omit
  it, so report every institution you saw and never a count. Give its reconnect link. Any total
  you compute may be missing those accounts.
- **There is no connection data.** `GetAccounts` has no sync state, last-updated time,
  provider, or active and hidden flags. Wherever the main file checks `sync.state` or
  staleness, say instead that a dead connection cannot be ruled out, and suggest
  `monarch-doctor` on the community server.
- **Usage is capped weekly** with no published number, so keep windows to what the question needs.
- **Compute `budget_amount - actual_amount` yourself, and say you did.** `difference` and
  `status` include rollover, so a category well under its monthly plan can read "Over Budget".
- **`status` is backwards for income and meaningless for transfers.** Income above plan and
  transfers both read "Over Budget". Use `category_type` to leave transfers out of verdicts.
- **Rows have no category id.** Join to `GetCashFlow` on `category` and `entity_name`.
- **Signs differ.** `actual_amount` is positive for spending. `GetCashFlow` `amount` is negative
  for expenses, and refunds can make a month positive.
- **Income totals can differ between tools.** `actual_income` in `GetBudget` and `total_income`
  in `GetCashFlow` did not match on a live account. Say which one you used.
- `budget_amount: 0` means unbudgeted, as in the main file.

## Apply

The official connector has no tool for writing budgets. Give the recommended amounts for the
user to enter in Monarch, and mention that the community server can write them.
