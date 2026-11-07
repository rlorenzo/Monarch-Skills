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
- **Hidden accounts still use up result slots.** `transaction_count` includes their
  transactions, so a page can come back short or empty. Follow `next_offset` while it is present.
  If it is absent and you have fewer rows than `transaction_count`, the rest are on hidden accounts
  and cannot be read. Say so, and do not retry or treat them as absent.
- **There is no connection data.** `GetAccounts` has no sync state, last-updated time,
  provider, or active and hidden flags. Wherever the main file checks `sync.state` or
  staleness, say instead that a dead connection cannot be ruled out, and suggest
  `monarch-doctor` on the community server.
- **Unknown filter keys are silently ignored and match everything.** `filters` is a JSON
  string that must include `transaction_type`. These keys were tested and work:
  `needs_review`, `search`, `merchants`, `accounts`, and `visibility`. `review_status` and
  `merchant_ids` do nothing. Sanity-check counts with `total_count_only` when a filter matters.
- **Paging is limited.** `limit` tops out at 100, and `include_details` forces it to 20. Usage
  is capped weekly with no published number, so avoid full-history scans when `GetCashFlow`,
  `GetMerchants`, or `total_count_only` can answer the question.
- **`GetTransactions` needs both dates.** Results are under `transactions`, most recent first.
  Amounts are signed: expenses negative, income positive.
- **Field names differ from the main file.** `pending` replaces `is_pending`. There is no raw
  statement text: `cleansed_description` only comes with `include_details`, as do `needs_review`
  and `reviewed` on rows that have a review state. Other rows can omit those two.
- **Split rows are not duplicates.** A plain listing replaces the parent with its parts, which share
  `original_transaction_id`. With `include_details`, the parent appears once with a `splits` array.
  A `$0.00` part is normal. Count the split once. `transaction_count` counts parts, so it can
  exceed the rows returned.
- **The `merchants` filter can miss.** It returned nothing for a name held by two merchant records.
  If it comes back empty with a count of 0, filter with `search` on the name, check each row's
  `merchant_id`, and treat `suggested_merchant_names` as leads only.
- **Hidden transactions are left out by default.** Pass `"visibility": "all"` when a count
  looks low.
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
