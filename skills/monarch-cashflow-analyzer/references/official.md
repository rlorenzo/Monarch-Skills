# Official connector

These replace Gather, Gotchas, and Apply in the main file.

## Gather

1. `GetCashFlow(base_query='{"group_by_time":"month","group_by_entity":"category"}', ...)` is the
   trend backbone. Rows have `entity_name`, `entity_id`, `month`, and `amount`.
2. `GetCashFlow` with `{"group_by_entity":"merchant"}` for merchant totals. An unfiltered grouped
   response also returns `total_income`, `total_expense`, and `net_savings_rate`. With a
   `category_type` filter it returns only `total_amount`.
3. `GetTransactions` for the periods or categories that need explaining. Follow `next_offset`.
4. `GetTransactions` with `filters='{"transaction_type":"All","merchants":["<merchant>"]}'` for
   one merchant's history. The match is fuzzy, so check the `merchant` on the rows returned. Do
   this over 12 months for the top recurring merchants only, chosen from the merchant totals. Skip
   usage-based billing. Say which ones you checked for price rises, and which had their history on
   a hidden account, so their price could not be checked.

Use the default windows from the main file.

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
- **Do not use `GetSpendingByCategory` for totals.** It returns only the top 5 categories.
- **`post_aggregation.limit` did not cut merchant results** in testing. Expect the full list.
- **A `GetCashFlow` total can include transactions on hidden accounts.** If a drill-down returns
  fewer rows, say the rest cannot be viewed. The total is not wrong.
- Exclude transfers before judging spending, using `"category_type"` in `filters` or the
  categories named in the main file's guidance.
- A partial current month looks like a decline. Exclude it or label it.
