# Official connector

These replace Gather, Gotchas, and Apply in the main file.

## Gather

This is a reduced check. **Say so in the first line:** the official connector cannot see
re-authentication, disconnects, sync settings, or providers, so this can only infer staleness,
and it cannot confirm a connection is healthy. For a real check, point to the community server.

1. `GetAccounts` for the account list and `finicity_excluded`.
2. `GetNetWorthHistory(start_date=<90 days ago>, interval="week", include_account_breakdown=true)`.
   Unfiltered, `account_deltas` lists only the 10 biggest movers (`account_deltas_truncated`), so
   a flat account never appears. To check for flat balances, repeat the call with `accounts` set
   to about 10 with a nonzero balance whose type should change regularly, such as checking, cards,
   and loans. There, a `change_amount` of zero marks a balance that has not moved. Movement is weak
   evidence the other way: prices, interest, and amortization change balances without a sync.
3. For accounts that should be active, `GetTransactions` with
   `filters='{"transaction_type":"All","accounts":["<name>"]}'` and `limit=1` gives the latest
   transaction date, which is the stronger signal. Do this for 2 or 3 of the most active accounts,
   since each one is a call.

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
- **Net worth totals include hidden accounts, but the breakdown does not.** Listed balances can
  add up to far less than `net_worth`. Use the breakdown per account, and never sum it as a total.
- **A flat balance is only a hint.** It can be a dormant or zero-balance account. Report it as
  "looks stale", never "broken".
- **Finicity accounts are a separate finding.** They are hidden from the connector, not broken
  in Monarch. Do not report them as disconnected.
- Never recommend deleting an account to silence a warning.
