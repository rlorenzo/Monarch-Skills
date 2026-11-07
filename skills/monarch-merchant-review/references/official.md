# Official connector

These replace Gather, Gotchas, and Apply in the main file.

## Gather

1. `GetCashFlow(base_query='{"group_by_entity":"merchant"}', ...)` over the full window. One call
   returns every merchant with `entity_id`, `entity_name`, and `amount`, so no transaction scan is
   needed to find splits. On a large account it lands in a file, with the JSON inside a `result`
   string, so group it with a script. Drop rows with no name.
2. `GetMerchants(search=...)` for each group you keep, for `merchant_id` and `transaction_count`.
   It matches a substring, so search the shared root of the spellings, with and without punctuation.
   `entity_id` matches `merchant_id`. Paging every merchant is impractical on a large account.
3. For the groups you keep, `GetTransactions` with
   `filters='{"transaction_type":"All","search":"<variant>"}'` gives each variant's last-seen date.
   The match is broad, so follow `next_offset` until a row's `merchant_id` is the variant's, or
   results run out.
4. `GetRecurring(include_pending=true)` and `ListRules`, to spot streams and rules that name a
   variant. Rules show no match operator: check `merchant_names`, `merchant_names_legacy`,
   `original_statement_values`, and any `set_merchant_id` action.

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
- **Recurring streams cannot be checked per merchant.** `GetRecurring` has no merchant ids, and
  there is no per-merchant lookup. Match stream names by hand, and say a split could still hold a
  stream that is not listed.
- **A shared word does not make a split.** Brokerage rows can name a company you also buy from.
  Identical names under different ids are splits too, so list them separately from spellings.
- **Cash flow totals are net.** Refunds and income reduce or flip a merchant's total, so compare
  absolute size when ranking.

## Apply

Only when the user asks, and only for what they confirm. Writes need write access on the
connection. That comes from the sign-in scope, not `MONARCH_MCP_READ_ONLY`. If a write is
refused or the tool is absent, say the connection is read-only and stop. Monarch cannot undo
these changes.

Merging has been run live. Check the result after each write.

`MergeMerchants(target_merchant_id=..., source_merchant_ids=[...])` moves the transactions and
rules from the sources to the target, then **deletes the sources**. Monarch cannot undo this, so
confirm each group by name and transaction count, one group at a time.

Pick the target as the main file describes. Afterward, confirm with `GetMerchants(search=...)`
that the source is gone and the target's `transaction_count` grew by the source's earlier count.
`moved_transaction_count` can be higher than that, so do not check against it. Then re-read `GetRecurring` and say whether any subscription changed. Never report a merge you did
not confirm.
