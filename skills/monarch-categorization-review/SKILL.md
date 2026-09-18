---
name: monarch-categorization-review
description: Review how recent Monarch Money transactions are categorized. Finds uncategorized transactions, flags likely miscategorizations, and proposes auto-categorization rules to stop them recurring. Use when the user asks to review categories, clean up categorization, find uncategorized or miscategorized transactions, or asks why spending in a category looks wrong.
license: MIT
allowed-tools: mcp__monarch-money__get_transaction_categories mcp__monarch-money__get_transaction_rules mcp__monarch-money__get_transactions mcp__monarch-money__get_transactions_needing_review mcp__monarch-money__get_transaction_details mcp__monarch-money__search_transactions
---

# Categorization review

Reports first. Recategorizes and creates rules only when the user asks, and only with
writes enabled. See `## Apply`.

## Gather

1. `get_transaction_categories` gives the full category list with ids. Note the id of
   `Uncategorized`, and keep the list to hand so every suggestion names a category
   that actually exists.
2. `get_transaction_rules` gives the existing auto-categorization rules. A rule you propose
   that already exists is noise, and an existing rule may itself be the cause of a
   miscategorization.
3. Uncategorized transactions: `get_transactions(category_ids=["<Uncategorized id>"],
   start_date=..., end_date=...)`.
4. Recent transactions for the review window: `get_transactions(start_date=...,
   end_date=...)`, default the last 30 days unless the user says otherwise.

Default to the last 30 days. Ask only if the user's phrasing is genuinely ambiguous
about the period.

## Gotchas

- **`get_transactions` needs both dates.** Passing only `start_date` returns
  `{"error": true, "message": "You must specify both a startDate and endDate..."}`.
- **Do not use `get_transactions_needing_review(uncategorized_only=True)` to find
  uncategorized transactions.** That filter is applied locally to one fetched page,
  so it returns `count: 0, truncated: true` even when plenty exist. Filter by the
  `Uncategorized` category id instead. `get_transactions_needing_review` is still
  the right tool for Monarch's own review queue.
- **Amounts are signed**: expenses negative, income positive. A positive amount in an
  expense category is usually a refund, not a miscategorization, so check before flagging.
- Paging is by `limit`/`offset`, default limit 100. If a result looks truncated, page
  until exhausted rather than reporting the first page as the whole answer.

## Judge

For each uncategorized transaction, propose a category from the real list, based on
merchant, description, amount, and how similar past transactions were categorized.
Say when you are guessing.

For miscategorization, the signal is inconsistency, not your opinion of where a
merchant belongs:

- The same merchant categorized differently across transactions. One of them is wrong,
  and the majority treatment is usually right.
- A transaction that does not fit its category's pattern: a $900 charge in Coffee Shops,
  a grocery-store charge in Home Improvement.
- Transfers and credit card payments categorized as spending, which double-counts them.
- Anything in a catch-all (`Miscellaneous`, `Other`, `Check`, `Cash & ATM`) that has an
  obvious real category.
- Business vs personal categories on the same merchant, where the split looks accidental.

Do not flag a merchant that legitimately spans categories (Amazon, Costco, Target)
just for spanning them. Flag it only when a specific transaction is clearly wrong.

## Propose rules

Only where a rule would have prevented a real finding above and no existing rule covers
it. A good rule is a merchant pattern that maps to one category with no exceptions in
the data you just read.

State each as: merchant/description match → category, plus how many transactions in the
window it would have fixed. A rule applies going forward, so existing transactions still
need recategorizing separately, either in the same pass or with `apply_to_existing`.

Skip rules for merchants that legitimately vary by transaction. A wrong rule is worse
than no rule: it miscategorizes silently, forever.

## Report

Lead with counts: how many transactions reviewed, how many uncategorized, how many
flagged. Then:

1. Uncategorized: date, merchant, amount, account, suggested category.
2. Likely miscategorized: same fields plus current category and why it looks wrong.
3. Suggested rules: pattern to category, and transactions fixed.

Order each list by dollar amount, largest first, which is the order in which fixing
them changes the user's reporting. Tables are fine; keep them scannable. If a section
is empty, say so in one line rather than padding it.

## Apply

Only when the user asks, and only for the findings they confirm. Recategorizing and
creating rules need `MONARCH_MCP_READ_ONLY=0`. If the tools below are absent, that is
why. Say so and stop rather than working around it.

Recategorize with `bulk_categorize_transactions(transaction_ids=[...],
category_id=...)`, which takes one category across many transactions and marks them
reviewed by default. Use `categorize_transaction` for a single one. Both fan out to one
request per transaction rather than a bulk API call, so a large group is slow rather
than atomic, and a partial failure leaves the rest applied. The result reports
`successful` and `failed` counts; read them rather than assuming the whole batch landed.

Create a rule with `create_transaction_rule`, then verify what it will do before moving
on. Verified against a live account:

- **A new rule is created last and therefore wins.** Rules run in order and a later one
  overwrites an earlier one, and creation appends to the end. So a new rule silently
  overrides every existing rule that matches the same transaction. Check
  `get_transaction_rules` for an existing rule on that merchant first, and prefer fixing
  it to stacking another on top.
- **`eq` matches one exact spelling.** A merchant recorded two ways, or renamed by the
  provider later, stops matching without any error. Prefer `contains` on the distinctive
  part of the name unless an exact match is genuinely wanted.
- **`update_transaction_rule` is non-destructive.** It re-reads the rule and merges the
  change over its current state, so passing only `merchant_criteria` keeps the existing
  category and tag actions. Use it rather than delete-and-recreate, which loses them.
- `apply_to_existing=True` runs the rule over past transactions as well as future ones.
  Say which of the two the user is getting, because "create a rule" usually sounds like
  both and defaults to neither.

Report per finding: what was recategorized, from what to what, and how many transactions
each rule now covers. Never report a rule as created without its id from the response.
