# Lessons learned

Traps that cost real money, real hours, or a wrong number in front of an accountant.
Each one came out of running live household and business budgets through Claude and the
YNAB API. They are here so you do not have to learn them the same way.

## The API will let you write nonsense

**The bulk endpoint silently accepts a mistyped category id.** Rows come back reported
as approved, and they are uncategorized. No error, no warning, and the success count
matches what you sent. Nothing flags the bad rows; only a later pass catches them.

Two rules came out of it, and they are not optional:

1. **Build every category id from a live read in the same run.** Never type one, never
   carry one over from an earlier session, never trust one in a note.
2. **Re-read uncategorized after every bulk write.** The response is not proof. The
   only proof is the budget's state afterward.

**The month endpoint can serve a cached income figure.** An allocation computed at the
top of a session can be stale by the time you write it: the figure refreshes mid-session
as new deposits land, and the split has to be redone. **Always re-read
`to_be_budgeted` immediately before writing an allocation**, not at the top of the
session.

## Reconciliation

**Never diff on (date, amount).** Card issuers report the transaction date; YNAB
imports the posting date. A strict two-field match **turns a couple of real differences
into hundreds of false ones.** Fuzzy-match on **amount with a plus or minus 7 day
window** and the noise disappears.

**Watch the sign flip.** Card exports typically list charges as positive. YNAB stores
them negative. A diff that ignores this finds nothing in common between two identical
lists.

**Hunt, do not adjust.** On business books especially, a cash-account adjustment dumps
into Ready to Assign and pollutes the income and spend reports that back the tax
return. A plug entry does not fix a discrepancy, it hides one that will come back
bigger. Two exceptions, both narrow:
- **The first-ever reconcile** of an account, where a baseline adjustment beats a
  ten-month forensic hunt. Date it, memo it clearly as a baseline, and treat prior
  history as best effort.
- **A residual under $10** that survives a full hunt.

**Categorize an account completely before you reconcile it.** Reconciled rows lock.
Category edits stay possible afterward; amount and date edits do not. Reconcile first
and you have cemented whatever was wrong.

**The reconcile lock is UI-only.** No API endpoint exists for it. Automate everything
up to the penny match, then hand off to a human to press the button. Worth knowing:
pressing it also arms YNAB's 3-day anti-duplicate import window.

## Categorization

**Payee memory lies on refunds.** A refund inherits the category the *payee* last used,
not the category of the purchase being refunded. If that category sees little traffic,
the refund sits there as phantom money until someone goes looking. Always match a refund
to its original purchase by amount and date, and leave an audit memo saying which one.

**Everything-store payees lie by default.** Payee memory labels every order from a
delivery or marketplace payee with the same category, and many of those labels will be
wrong. One payee can cover several unrelated categories in the same week. Marketplace,
delivery, and big-box payees get inspected per transaction, never blanket-mapped.

**Only trust payee memory at about 90% consistency.** A payee that maps to one category
nine times out of ten can auto-categorize. Payment processors and multi-purpose
merchants stay inspect-every-time, permanently.

**Categorized inflows are invisible.** A pass that queries only uncategorized
transactions will never see money that arrived already categorized, so an earmarked
inflow quietly diffuses into general funds. Query `Inflow: Ready to Assign` separately,
and assign earmarked money to its target the day it lands.

**Transfer legs are supposed to look uncategorized.** Payees beginning `Transfer :`
never take a category. Counting them as backlog creates a phantom queue that never
clears no matter how much work you do.

## Reading the budget

**`to_be_budgeted` is the only true Ready to Assign.** The "Inflow: Ready to Assign"
*category balance* accumulates a lifetime artifact that can grow enormous. It is not
cash, and reading it as RTA produces a confidently wrong answer.

**The trust check.** Total Available across all categories, plus RTA, must equal total
on-budget cash. To the penny. If it does not, something is wrong and no amount of
categorizing will fix it. Run it every pass; it is the cheapest correctness check
available.

**Watch for double-funding.** Clearing card-charged overspending automatically feeds
the card's payment envelope, *on top of* any direct assignment you make to it, so an
allocation can overfund the envelope without anyone noticing. Budget for the
interaction, or fund the envelope last.

## Structure and strategy

**Repair in place. Do not Fresh Start.** YNAB's Fresh Start archives transaction
history out of the live plan. On a business budget the history *is* the tax record.
Plan Reset is the official middle path and it re-zeroes every assignment, which
flattens category balances you may badly want to keep, a fully funded tax envelope being
the one you will miss most.
Repair in place: one account at a time, categorize everything, then reconcile, then fix
the plan forward.

**Current month only.** Assignments in past months are read-only history. Rewriting
them destroys the record of what was actually decided at the time. Categorization fixes
to past transactions are fine and are the entire point of cleanup; allocation changes
are not.

**An existing habit beats a textbook date.** Profit First prescribes allocations on the
10th and 25th. If a bookkeeper already runs a different rhythm, keep it: that costs
nothing and preserves the one part of the system that is already working.

**Small anchored sessions, not heroic catch-ups.** The relapse pattern is always the
same: skip a week, skip a month, then face a backlog of hundreds and skip that too.
One 15-minute weekly pass prevents the thing that actually kills the system.

## Working with an assistant on money

**Propose, wait, write.** Reads are free. Every write gets explicit approval in the same
session. This is the only real protection, and it is worth the friction.

**Three gates, not a hundred.** Per-item approval on a 100-transaction backlog gets
abandoned halfway through. Batch into coherent chunks: categorize, then structure, then
amounts. Three decisions a person will actually make beats a hundred they will not.

**Confidence tiers make the batch reviewable.** HIGH gets approved in bulk. MEDIUM gets
a spot-check. LOW gets one question with two or three candidate answers. Never guess
past LOW, and never auto-approve it.

**Flag, do not classify.** Personal spending on a business account gets flagged for the
owner to tag themselves. An assistant guessing at the business/personal line is
guessing at a tax position.

**Name the slip.** When a month gets repaired by moving money, say what got sacrificed.
"Debt payoff got only part of its target this month because the refund went to making
the month honest" is a real report. "All categories funded" is not.
