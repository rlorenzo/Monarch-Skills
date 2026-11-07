# Official connector

These replace Gather, Gotchas, and Apply in the main file.

## Gather

1. `GetCategories` for the full list with ids and types.
2. `ListRules` for existing rules, since one may cause the miscategorization. Check
   `merchant_names`, `merchant_names_legacy`, and `original_statement_values` in `criteria`. A rule
   with `amount` or `account_ids` there only applies to those transactions.
3. `GetTransactions` for the review window, default the last 30 days, with
   `filters='{"transaction_type":"All","categories":["Uncategorized"]}'`.
4. `GetTransactions` for the same window with no category filter. Follow `next_offset`.

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
- A positive amount in an expense category is usually a refund. Check before flagging it.
- Statement text is not on normal rows. Use `include_details` on a narrow filter when the
  merchant alone does not settle a suggestion.

## Apply

Only when the user asks, and only for what they confirm. Writes need write access on the
connection. That comes from the sign-in scope, not `MONARCH_MCP_READ_ONLY`. If a write is
refused or the tool is absent, say the connection is read-only and stop. Monarch cannot undo
these changes.

The behavior below comes from the connector's tool descriptions and has not been tested on a
live account, so say so and check the result after each write.

Recategorize a group with `BulkRecategorizeTransactions(category_id=..., transaction_ids=[...])`.
Run it first with `dry_run=true`, which previews the affected count without writing, then
show that to the user. Use `UpdateTransaction` for a single one.

Create a rule with `CreateRule`, using ids from the read tools, then read it back with
`ListRules`. Check for an existing rule on that merchant first, and prefer `UpdateRule`, which
only changes the fields you pass.

- **Rules here only affect future transactions.** There is no option to apply one to past
  transactions, so recategorize those separately, or pass on the response's `user_notice`.
- `merchant_names` are literal text patterns matched against the merchant name.

Report what moved, from what to what. Never report a rule as created without its id.
