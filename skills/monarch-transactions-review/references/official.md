# Official connector

These replace Gather, Gotchas, and Apply in the main file.

## Gather

1. `GetTransactions` with `filters='{"transaction_type":"All","needs_review":true}'` and
   `total_count_only=true` for the queue size. Compare it with any number the user quoted.
   Use a start date that predates the account.
2. The same filter with `limit=100`, following `next_offset`, for the whole queue.
3. For matching, size the same dates plus a month either side with `total_count_only`. On a very
   large account, fetch only `categories` `Transfer`, `Credit Card Payment`, `Auto Payment`, and
   `Loan Repayment`, and say you narrowed it. If you skip matching, say the matched-transfer safe
   rule could not apply.
4. `GetAccounts`, to spot duplicate accounts, and `ListRules`, to check merchants behind findings.
5. `GetCategories` for each category's type, and `GetCashFlow` with
   `base_query='{"group_by_time":"month","group_by_entity":"category"}'` for a period the queue
   covers, to confirm which categories count.

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
  `needs_review`, `search`, `merchants`, `accounts`, `categories`, `category_type`, and `visibility`. `review_status` and
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
- **Say up front that matching is by merchant.** Normal rows have no statement text or review
  status. Tests in the main file that depend on statement text need `include_details`, 20 rows at a
  time, so use it on shortlisted findings only: duplicates, installment "N of M" checks, and
  generic merchants.
- **Matches can be false.** Without statement text, duplicates and same-merchant lookalikes are
  harder to tell apart. Keep saying "possible". The other side of a transfer may be on a hidden
  account, so say that before calling an unmatched transfer spending.
- **Tally the buckets even without full history.** Where you did not fetch a merchant's history,
  the rows that need it do not apply. Those transactions fall through to Review, unless one
  `merchants` lookup for a frequent merchant settles it. Say how many went to Review for this
  reason, since it can be most of them. Tally per row and say so. Results come back inline, so keep
  running totals as you page.
- **Queue items on hidden accounts cannot be read, and they can be most of the queue.** `shown_count`
  includes them, so it is not the rows returned. Lead the report with how many you could read out
  of the queue count, and the institutions from `finicity_excluded`. Unread rows are in no bucket.
- **`needs_review_by_user` grouping in `GetCashFlow` returned no per-user split** in testing. Do not
  use it to size the queue.

## Apply

Only when the user asks, and only for what they confirm. Writes need write access on the
connection. That comes from the sign-in scope, not `MONARCH_MCP_READ_ONLY`. If a write is
refused or the tool is absent, say the connection is read-only and stop. Monarch cannot undo
these changes.

Clearing with `review_status` has been run live. Everything else below comes from the connector's
tool descriptions, so say so and check the result after each write.

Recompute from a fresh queue read first.

Fix categories before clearing:
`BulkUpdateTransactions(transaction_ids=[...], updates='{"category_id":"...","review_status":"reviewed"}')`.
To clear a bucket, leave out `category_id`. Run it first with `dry_run=true`, which previews the
affected count without writing. No batch size has been tested here, so start with a small batch.

Keep the ids and original categories of each change somewhere the user can find them. Setting
`review_status` to `needs_review` should undo a clear, but that has not been run live, so check
the result. Restoring categories takes that record.

`DeleteTransaction` only works on manual transactions, so it cannot remove a stuck synced item.
Leave those for Monarch's app, and use it only on manual transactions the user names.

Afterward, re-read each row with `include_details`, the queue count, and `GetCashFlow` for an
affected period, and report them. Clearing a normal row lowers the count by one. Clearing a split
parent removes its row without lowering the count, so check the row and do not expect that drop.
