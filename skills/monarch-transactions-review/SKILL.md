---
name: monarch-transactions-review
description: Work through the Monarch Money transaction review queue, including backlogs of thousands. Finds what is distorting the numbers (income that is not income, spending hidden as transfers, possible duplicates, stuck pending items, accounts that exist twice) and sorts every flagged transaction into safe to mark reviewed or worth a look. Use when the user mentions transactions needing review, a review queue or backlog, asks whether anything looks strange in unreviewed transactions, or wants to bulk mark transactions reviewed, even if they only give a count.
license: MIT
allowed-tools: mcp__monarch-money__get_transactions_needing_review mcp__monarch-money__get_transactions mcp__monarch-money__get_transaction_categories mcp__monarch-money__get_transaction_rules mcp__monarch-money__get_accounts mcp__monarch-money__get_account_sync_health mcp__monarch-money__get_cashflow_by_month
---

# Transactions review

Reports first. Changes categories, review flags, or deletes only when the user asks, and
only with writes enabled. See `## Apply`.

The point is not to read every transaction. It is to find the few that distort the
user's numbers, then shrink the queue to what a person actually needs to look at.

## Gather

1. `get_transactions_needing_review(limit=<more than the queue>)` for the whole queue.
   Call it once with `limit=1` first: `total_count` gives the size. Compare it with any
   number the user quoted and report a gap.
2. `get_transactions` for the same dates plus a month either side, in date chunks. Its
   `limit` defaults to 100, so page each chunk with `limit` and `offset` until
   `truncated` is false. This is what the matching below runs against, because the other
   side of a transfer may already be reviewed and so absent from the queue.
3. `get_accounts` and `get_account_sync_health`, so a stale connection is not mistaken
   for missing activity, and duplicate accounts can be spotted.
4. `get_transaction_rules`, since a rule may be causing a repeated miscategorization.
   Check it against the merchants behind your findings. The result can be large.
5. `get_transaction_categories`, to tell which categories are spending, income, or
   transfer, and `get_cashflow_by_month` for a period the queue covers, to confirm which
   of them count.

Do all of this even for a narrow question. A stale connection or a rule can change the
answer, and the matching needs the wider set. If the user asks about one thing only,
answer it, then say the full sort was not done and offer it.

Large results get written to a file instead of returned. Analyze them with a short
script, rather than reading thousands of rows into the conversation.

## Gotchas

- **Keep date chunks to a few thousand transactions.** A single `get_transactions` call
  over about 26 months timed out; chunks of up to 3,370 transactions returned fine.
  It needs both `start_date` and `end_date`.
- **Field names differ.** The queue tool returns the statement text as `original_name`.
  `get_transactions` calls it `original_statement`, and also adds `review_status`.
- **`category_group` comes back empty on transactions.** Map a category to its group
  from `get_transaction_categories`.
- **Hidden transactions do not come back from either tool**, even when they exist. That
  can make the count lower than the one in Monarch's app. Say it may explain a gap, not
  that it does.
- **Match merchants case-insensitively, on merchant and statement text.** The same
  business can appear in capitals or under a longer name, and an exact match undercounts.
- **Do not assume a category's effect on cash flow. Check it** with
  `get_cashflow_by_month`. `Uncategorized` counts toward cash flow, while transfer-group
  categories come out as zero. Custom investment categories can go either way.
- **The queue can stop at a sharp date.** No rule sets review status on its own, and the
  cause may not be visible from the API. Report the boundary as a question.

## Judge

Look for these, in this order, because this is the order they mislead the user.

**Income that is not income.** In income categories, look for card payments,
overdraft or loan advances, refunds, and transfers between the household's own accounts.
Test: an equal and opposite amount on another account within 7 days, whatever that
transaction's category is. A credit card autopay in a paycheck category is the classic.

**Real income or spending in the wrong place.** The same employer in a non-income
category for a month or two, or a payment side in a loan category.

**Interest counted twice.** Two accounts posting identical interest to the cent, month
after month, despite different balances. Compare with their earlier months: amounts that
differed and then became identical point to a bad feed. Ask the user to check a statement.

**Spending hidden as transfers.** Anything in a transfer or card payment category with
no opposite side on another account: person-to-person payments and gift card or payment
processor charges on a credit card. Matching only against the queue overstates this,
because the other side may be reviewed already. Match against all transactions.

**Possible duplicates.** Same account, date, amount, and statement text. Then check
before calling one a duplicate:
- Two installments of equal size, each with the same "N of M", that both begin in the
  same month are usually two plans, not a duplicate. Say so in one line, leave them out
  of the possible duplicates, and sort them like any other transaction.
- Equal transfers that appear on both accounts the same number of times are probably real.
- Sync duplicates do happen, so "probably fine" still means ask the user to check a
  statement. Say "possible".

**Stuck pending.** `is_pending` true and weeks old. Look for a posted copy with the same
amount nearby. Absent one, the user has to decide whether it is real by checking at the
institution.

**The same account twice.** Two accounts with the same name, or one that stops the day
another starts, typically after a re-link or an issuer migration. Check how each one's
transactions are categorized. A linked replacement can put purchases in a payment
category, which removes them from spending without any error.

**Patterns worth a sentence.** Fees, a large spike at the same time each year, or a
recurring payment to something with no linked account. Ask about these rather than
assuming: a large insurance-category payment can be correct and still leave an
asset untracked.

## Sort into buckets

Sort by whether a transaction is risky, not by whether its merchant is consistent. A
store selling groceries, fuel, and electronics legitimately spans categories, and a
merchant seen twice is not suspicious for that alone. A consistency rule sends all of
these to the user for no reason, and that is how a queue stays unmanageably large.

Use these definitions, so the same queue sorts the same way each time. Measure merchant
history over everything fetched, not just the queue, matching names case-insensitively:

- **Small line:** $100. **Large line:** $500. Scale both to the household and say what
  you used.
- **Normal spending:** a negative amount in a spending category that is not a catch-all.
- **Catch-all:** `Uncategorized`, `Miscellaneous`, `Check`, `Cash & ATM`, `Other Income`.
- **Established category:** the merchant has 5 or more transactions, and at least 10% of
  them are in this category.
- **Usual amount:** in absolute terms, at most 3 times the merchant's median amount.

Take the first row that matches:

| Bucket | Rule | Why |
|---|---|---|
| Review | Pending | May never post |
| Review | Part of a finding above, possible duplicates included | Unresolved |
| Safe | $0 | No effect on totals |
| Safe | Transfer or card payment with a matched opposite side | Both sides accounted for |
| Review | Any other transfer or card payment | May be hidden spending |
| Safe | Interest or dividend under the small line | Routine, once checked for double counting above |
| Review | Income or catch-all category | Decides reported income, or has no real category |
| Review | Person-to-person or generic merchant (Zelle, Venmo, PayPal, "Deposit") | The merchant says nothing about the purpose |
| Safe | Any other activity on an investment account | High volume, and rarely in cash flow. Confirm with the cash flow check |
| Review | Any other positive amount | May be a refund or a miscategorized credit |
| Review | At or above the large line | Worth a glance whatever it is |
| Safe | Normal spending under the small line | Low stakes, whatever the merchant |
| Safe | Normal spending, established category, usual amount | Nothing stands out |
| Review | Everything else | Unusual for this merchant |

"Safe" means nothing stood out, not that it was verified: a wrong but plausible category
passes. Say so.

A recurring paycheck can move to safe once the user has looked at the series, so offer
that as a next step.

## Report

Lead with the size of the queue and how much of it looks safe. Then:

1. What distorts the numbers: a table of the issue, the evidence, and the effect in
   dollars. Largest effect first.
2. Questions only the user can answer.
3. Possible duplicates and other oddities, with dates and amounts so they can be checked.
4. The buckets: a table with the count and total for each, with the largest items in the
   safe buckets named, so the user can see what they would be approving.
5. Stale connections: which transactions are missing because of them.

If a section is empty, say so in a line. End by asking which changes to make.

## Apply

Only when the user asks, and only for what they confirm. These writes need
`MONARCH_MCP_READ_ONLY=0`. If the tools below are absent, that is why. Say so and stop
rather than working around it.

Recompute from a fresh queue read first, in case it has changed.

Fix categories before clearing, so nothing marked reviewed is still wrong:
`bulk_update_transactions(transaction_ids=[...], category_id=..., needs_review=False)`.
To clear a bucket, leave out `category_id`. Pass `needs_review=False` deliberately, because
this tool leaves the review flag alone unless told. Its `dry_run` only echoes the planned
update without asking Monarch anything.

Send batches of up to 500 ids: this size went through every time, with `requested` equal
to `affected`. If they differ, stop and report rather than retrying. A timeout means
unknown, not failed, so re-read before retrying.

Keep the ids and original categories of each change somewhere the user can find them, since
a bulk update reports only a count. Re-flagging undoes a clear. Restoring categories takes
that record.

`delete_transaction` cannot be undone. Use it only on transactions the user names. It has
worked on a stuck $0 pending item. There is no write for making a pending item post, so
deleting is the only fix on offer, and a real item is better left and marked reviewed.
Whether a deleted item returns on a later sync has not been verified, so say so.

Afterward, re-read the queue to confirm the count, and re-run `get_cashflow_by_month` for
an affected period to confirm that the totals moved as predicted. Report both.
