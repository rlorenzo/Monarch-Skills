---
name: monarch-merchant-review
description: Find merchants split across several spellings in Monarch Money: the same business recorded under two or more names, which fragments merchant-level reporting and recurring-subscription detection. Use when the user asks about duplicate or split merchants, why a merchant's totals look low, why one subscription appears twice, or wants to tidy up merchant names.
license: MIT
allowed-tools: mcp__monarch-money__get_transactions mcp__monarch-money__get_transaction_details mcp__monarch-money__get_merchant mcp__monarch-money__get_recurring_transactions
---

# Merchant review

Reports first. Merges only when the user asks, and only with writes enabled. See
`## Apply`.

Monarch creates a separate merchant record per distinct name its providers send. The
same business arrives hyphenated and unhyphenated, in title case and in caps, with and
without a leading `The`, and every merchant-level view treats each spelling as its own
business.

## What this does and does not fix

Categories live on the transaction, not the merchant, so splits do **not** corrupt
category spending, budgets, or cash flow. Say this plainly, because the user may assume the
problem is bigger than it is.

What splits do break:

- Merchant-level totals and "top merchants". A split merchant is understated by
  however much sits under its other name.
- Recurring detection. Each merchant record carries its own recurring stream, so one
  subscription can appear twice, and `monarch-subscription-manager` will double-count it.
- Rules keyed to a merchant with the `eq` operator, which silently stop matching when a
  new spelling appears.

## Gather

There is no tool that lists merchants. Build the list from transactions.

1. `get_transactions(start_date=..., end_date=..., limit=2000, offset=N)`, paging on
   `offset` until you have `total_count`. Default to the full history unless the user
   names a window. Splits accumulate over years, and a 30-day window finds almost none.
2. Group by the `merchant` field and count transactions per name.

## Gotchas

- **`get_transactions` needs both dates.** Passing only `start_date` returns
  `{"error": true, "message": "You must specify both a startDate and endDate..."}`.
- **The response is an envelope, not a list.** Transactions are under `data`, with
  `count`, `total_count` and `truncated` alongside. Treat `truncated: true` as "keep
  paging", never as the whole answer.
- **A full page will exceed the tool-output limit and be written to a file instead.**
  At `limit=2000` a page is roughly 2 MB. Do not try to read those files into context.
  Extract `.result`, parse it, and do the grouping in a script.
- **Neither `get_transactions` nor `search_transactions` returns a merchant id**, only
  the display name. Grouping by name is enough to find splits, and the merge below works
  by name too, so you rarely need ids. `get_transaction_details` has the id under
  `merchant.id` when you do.
- **`get_merchant` takes an id, not a name.** Use it to confirm a merge landed:
  `transaction_count`, and `can_be_deleted: true` once a record is empty.

## Judge

Normalize names before comparing (lowercase, strip a leading `The`, drop punctuation
and whitespace, drop a trailing `Inc`/`LLC`/`Ltd`/`Corp`), then group names that collide.
Then throw most of the groups away. On a real account the raw collision list is mostly
noise, and reporting it unfiltered buries the few that matter.

Discard:

- **Statement lines that are not merchants.** Transfers, ACH deposits, ATM withdrawals,
  autopay and payment lines, dividends and reinvestments, buy/sell lines, and
  installment counters like `Monthly Installments (12 of 24)`. These collide constantly
  on capitalization alone and merging them changes nothing.
- **Groups whose transactions all sit in transfer-ish categories** (`Transfer`,
  `Credit Card Payment`, `Balance Adjustments`, `Buy`, `Sell`, `Investments`).
- **Store-number variants**, unless the user asks for them. A name carrying a branch
  number against the same name without one is a real split, but collapsing per-location
  records is a bigger decision than fixing a spelling, and some people want the
  locations kept apart.

Keep a group when the names are plainly one business and at least one variant has enough
transactions to move a total. Pick the canonical name: the spelling with the most
transactions, preferring the one that is properly capitalized and carries no store
number or provider prefix.

Flag separately, because it is the case with a real cost: a split where more than one
variant has a recurring stream. Cross-check against `get_recurring_transactions`.

## Report

Lead with the shape of it: transactions scanned, distinct merchant names, how many
groups survived filtering, and how many transactions they cover. Then one table, ordered
by total transactions descending, which is the order in which merging changes reporting:

| Canonical | Variants (count) | Transactions | Recurring? |

Name the noise you filtered out in one line, with its count, so the user can ask for it
if they disagree with the cut. Do not paste the unfiltered list.

If nothing survives filtering, say so in one line. A clean merchant list is a normal
result, not a failure to look hard enough.

## Apply

Only when the user asks, and only for groups they name or confirm. Merging is a write,
so it needs `MONARCH_MCP_READ_ONLY=0`. If the tools below are absent, that is why.
Say so and stop rather than working around it.

There is no merge endpoint. Reassign each transaction to the canonical name:

`update_transaction(transaction_id=..., merchant_name="<canonical>")`

Passing a name that already exists attaches the transaction to that existing merchant
record rather than creating another one. Verified: the response comes back with the
canonical merchant's own id. Passing a name that does not exist creates a merchant, so
copy the canonical spelling exactly, including punctuation.

Work one group at a time:

1. Collect the transaction ids for the non-canonical variants.
2. Update the first one, and check the returned `merchant.id` matches the canonical
   record before doing the rest. If it came back with a new id, the spelling was off.
   Stop and fix it rather than creating a third merchant.
3. Confirm with `get_merchant(canonical_id)`: `transaction_count` should have grown by
   exactly the number moved.

Then tell the user what is left behind, because the merge does not clean it up:

- **The emptied merchant record survives** with `transaction_count: 0` and
  `can_be_deleted: true`. This server exposes no merchant-delete tool; it is removed in
  Monarch's UI, or left alone, where it is harmless but visible in merchant pickers.
- **Its recurring stream survives and stays `is_active: true`**, even though the
  merchant's own `has_active_recurring_streams` flips to `false`. A stale stream keeps
  the subscription showing twice, which was likely the point of merging. Offer
  `review_recurring_stream(stream_id=..., review_status="ignored")` to dismiss it, and
  do not call it without asking, since it is a second write on a different object.
- **Rules keyed to the old spelling still exist** and will re-split future charges. A
  merchant rule using `eq` only matches that exact name; switching it to `contains` on
  the distinctive part of the name covers every variant. `update_transaction_rule` is
  non-destructive: it re-reads the rule and merges your change over its current state,
  so passing only `merchant_criteria` keeps the existing category and tag actions.

Report what changed, per group, with the counts you verified. Never report a merge you
did not confirm with `get_merchant`.
