# Tool reference

The 13 tools the skills call. Every tool takes `account` ("personal" or "business")
as a required first argument and `budget_id` (defaulting to `"last-used"`).

**All amounts are dollars**, converted from YNAB's milliunits at the server boundary.
Pass `600.00`, not `600000`.

## Reads

### `ynab_list_budgets(account)`
Every budget the token can see. Returns id, name, last_modified_on, first and last
month. The `id` is what you pass as `budget_id` elsewhere, or use the literal
`"last-used"` for the most recently opened budget.

### `ynab_list_accounts(account, budget_id, include_closed=False)`
Bank and card accounts. Returns id, name, type, `on_budget` flag, and balances in
dollars. Closed accounts are filtered out unless you ask for them.

### `ynab_list_categories(account, budget_id, include_hidden=False)`
Category groups, each holding its categories. Each category carries id, name,
current-month budgeted, activity, balance, and goal info if set.

**Amounts here are current month only.** For any other month use `ynab_get_month`.

**Call this every run.** Category ids and names drift as the budget gets restructured.
A hardcoded id does not error when it goes stale, it writes money into the wrong place.

### `ynab_list_payees(account, budget_id)`
Payees with id, name, and `transfer_account_id`. That last field is set when the payee
represents an account transfer, which is how you identify transfer legs and how you
create a transfer.

### `ynab_list_transactions(account, budget_id, account_id, category_id, payee_id, since_date, transaction_type, limit)`
Transactions with filters. Pick **exactly one** of `account_id`, `category_id`, or
`payee_id` to scope by entity, or omit all three for budget-wide.

- `since_date`: ISO date. Strongly recommended. Without it you pull years of history.
- `transaction_type`: `"uncategorized"` or `"unapproved"`.
- `limit`: newest first.

Returns date, amount in dollars, payee_name, category_name, account_name, memo, and
cleared/approved status.

### `ynab_get_month(account, budget_id, month="current")`
The canonical "how am I doing this month" call. Month totals (income, budgeted,
activity, **to_be_budgeted**) plus a per-category breakdown for that specific month.
`month` is an ISO month start (`YYYY-MM-01`) or the literal `"current"`.

## Writes

### `ynab_update_transaction(account, transaction_id, budget_id, category_id, payee_id, payee_name, memo, approved, cleared, flag_color)`
Updates a single transaction. Only the fields you pass change; omitted fields are left
alone. Does not move money and does not delete.

`cleared` is `"cleared"`, `"uncleared"`, or `"reconciled"`. If both `payee_id` and
`payee_name` are passed, the id wins. A `payee_name` matching nothing **creates a new
payee**, which is an easy way to litter a budget with near-duplicate payees.

### `ynab_update_transactions_bulk(account, updates, budget_id)`
**The most valuable tool in the set.** Many transactions in one API call. Each item in
`updates` is a dict with `transaction_id` plus any of `category_id`, `payee_id`,
`payee_name`, `memo`, `approved`, `cleared`, `flag_color`.

Roughly 100 transactions per call, against a 200-per-hour limit. Use this instead of
looping `ynab_update_transaction`. Returns counts and the saved ids; check the count
against what was approved.

### `ynab_create_transaction(account, account_id, date, amount, budget_id, payee_id, payee_name, category_id, memo, cleared, approved, flag_color)`
Adds a transaction YNAB never imported, typically one found during a reconcile.

**Sign convention: negative is outflow, positive is inflow.** Dollars.

To create a **transfer**, set `payee_id` to the destination account's transfer payee:
the payee from `ynab_list_payees` whose `transfer_account_id` equals the destination
account id, named like `Transfer : Checking`. YNAB creates the matching leg
automatically. Leave `category_id` unset for transfers between on-budget accounts.

`approved` defaults to True so the transaction does not land in the unapproved queue.

### `ynab_delete_transaction(account, transaction_id, budget_id)`
Permanent. If the transaction is a transfer, **the matching leg on the other account
goes too.** Used for stale or duplicate entries found during a reconcile. Returns the
deleted transaction as confirmation, which is the only record you will have.

### `ynab_update_month_category(account, category_id, budgeted, budget_id, month="current")`
The "assign $X to this category this month" knob. Only the assigned amount is settable;
activity and balance are derived from transactions and cannot be written.

Pass dollars as a float. **Only ever write the current month.** Past months are
read-only history, and rewriting them destroys the record of what was actually decided.

### `ynab_create_category(account, name, category_group_id, budget_id, note, goal_target, goal_target_date)`
New category inside an existing group. Get `category_group_id` from
`ynab_list_categories`. `goal_target` is dollars, `goal_target_date` is an ISO date.

### `ynab_create_category_group(account, name, budget_id)`
The parent that holds categories. Name is capped at 50 characters.

## What is not exposed

The connector has no rename, no hide, and no delete for categories, and no way to
perform YNAB's formal **reconcile lock** (that is UI-only, by design, in YNAB itself).

The skills handle this by listing those as explicit manual steps at the end of a run
rather than pretending the work is done.

## Traps

**`to_be_budgeted` is the only true Ready to Assign.** Do not read the balance of the
"Inflow: Ready to Assign" *category*. It accumulates a lifetime artifact that can grow
enormous and is not spendable cash.

**Transfer legs look uncategorized and are supposed to.** Payees starting `Transfer :`
never take a category. Counting them as a dirty-books backlog produces a permanent
phantom queue that never clears no matter how much you categorize.

**Payee memory lies on refunds.** YNAB assigns a refund the category the payee last
used, not the category of the purchase being refunded. Match refunds to the original
purchase by amount and date.

**Everything-store payees lie by default.** A single delivery or marketplace payee can
legitimately span several unrelated categories in the same week. Never trust payee
history alone for those.

**Categorized inflows are invisible to an uncategorized-only pull.** Money earmarked
for something specific gets categorized on arrival and then never appears in your
cleanup queue, so it quietly diffuses into general funds. Query
`Inflow: Ready to Assign` separately.

**YNAB renamed budgets to "plans" in API v1.83.** Normalize it at the server boundary
and keep saying "budget" to the user; the UI and the API disagree and it is not worth
propagating.

**200 requests per hour, per token.** Exceeding it returns 429 and locks you out for
the rest of the hour, typically halfway through a categorization pass.
