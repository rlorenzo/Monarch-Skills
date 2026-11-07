# Official connector

These replace Gather, Gotchas, and Apply in the main file.

## Gather

1. `GetRecurring(include_liabilities=false)`. It returns four buckets: income, expense, credit card,
   and transfer streams. Each stream has `stream_id`, `name`, `merchant`, `frequency`, `amount`,
   `last_paid_date`, `next_forecasted_date`, `account`, and `category`. Pass `include_pending=true`
   to see suspected streams as well.
2. Find charges Monarch has not flagged. Size the last 12 months with `total_count_only`, and page
   them only if that fits the usage cap. Otherwise pick candidates from `GetMerchants`, which is
   sorted by transaction count, skip transfer and investment names, and try
   `search` on words like subscription and membership. Say the sweep was not exhaustive.
3. `GetTransactions` with `filters='{"transaction_type":"All","merchants":["<merchant>"]}'` per
   candidate for its charge history, cadence, and price changes. The match is fuzzy, so drop rows from
   other businesses. Keep a spelling variant only if its amounts and cadence continue the same
   charge, because its older charges are the price history. If that is unclear, report them
   separately and say so.
4. `GetCashFlow` with no `base_query` for income. It includes transfers and card payments, so say
   the comparison is rough.

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
- **The stream list is incomplete.** Check raw transactions as well.
- **`last_paid_date` can be wrong in both directions.** A stream kept forecasting long after its
  last charge, and another showed a date a year old while charges continued monthly. Before
  calling a stream ended, look up its charges. If the lookup is empty but `transaction_count` is
  above 0, the charges are on a hidden account, so the stream may still be active. If the count is
  0, check the spellings in `suggested_merchant_names` before calling it ended.
- **`amount` is the expected amount, not what was charged.** Use real transactions for cost.
- **No date window or per-charge rows.** There is one call for the whole list, and nothing to mark
  a forecast separately from `next_forecasted_date`.
- **Not every stream is a subscription.** Mortgages, utilities, paychecks, and transfers recur too.
- Some streams have no `account` or `category`. Look them up by merchant instead of dropping them.
