<div align="center">

# YNAB Claude Skills

**Run a real budget with Claude. Two skills, one for the household and one for the company,
built on Jesse Mecham's Four Rules and Mike Michalowicz's Profit First.**

[![License: MIT](https://img.shields.io/badge/License-MIT-informational.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-skills-8A63D2.svg)](https://claude.ai/code)
[![YNAB API](https://img.shields.io/badge/YNAB-API%20v1-1CA9E8.svg)](https://api.ynab.com)

</div>

These are not prompt templates. They are a complete working setup for running a household
budget and a company's Profit First allocation against the live YNAB API, built and tested
on real money. That is also why the [lessons file](docs/lessons-learned.md) exists.

## What it actually does

Ask "how are we doing?" and get a report in this shape back in about a minute, from
live data:

```
YNAB DIAGNOSIS - [Month] [Year]

Rule 1 (Every Dollar a Job)
  - Ready to Assign: $X (target: $0)
  - Uncategorized transactions: N from last 30 days
  - Verdict: GREEN

Rule 2 (True Expenses)
  - Funded this month: [categories]
  - Underfunded: [category] ($X of $Y)
  - Verdict: YELLOW

Rule 3 (Roll with the Punches)
  - Overspent this month: [category] (-$X)
  - Chronically overspent: [category] (N recent months)
  - Verdict: YELLOW

Rule 4 (Age Your Money)
  - Approximate Age of Money: N days (target: 30+)
  - Verdict: YELLOW

Top 3 actions, in priority order:
  1. [Category] has been short N months running. The target is the fiction,
     not the spending. Raise it to $X and cut [category] to match.
  2. Top up [category] $X (Rule 2 / Ch 3).
  3. Categorize the N stragglers so next month's numbers are real.
```

Then it waits. It does not write to your budget without you saying so.

## Two skills, deliberately separate

|  | Personal | Business |
|---|---|---|
| **Skill** | [`ynab-coach`](personal/skills/ynab-coach/SKILL.md) | [`ynab-profit-first`](business/skills/ynab-profit-first/SKILL.md) |
| **Command** | [`/ynab-daily`](personal/commands/ynab-daily.md) | [`/ynab-reconcile`](business/commands/ynab-reconcile.md) |
| **Framework** | Mecham, the Four Rules | Michalowicz, Profit First, on top of YNAB |
| **Answers** | "Is every dollar doing a job?" | "What gets allocated, and is profit funded?" |
| **Cadence** | Daily categorize, monthly budget date | Weekly pass, allocation twice a month |
| **Modes** | 8, from a 60-second diagnose to an autonomous build | 2, a fast LIGHT status and a DEEP assessment |

**Do not merge them.** A household budget and a company allocation have different
invariants, different approval rhythms, and different failure modes. One shared skill
means every trigger loads the wrong half, and the two drift apart the first time you
edit one.

## How the pieces fit

```
          YNAB (your real money)
                   |
                   |  200 requests/hour
                   v
        +----------------------+
        |   MCP server         |   13 tools, dollars not milliunits
        |   ynab_* tools       |   explicit account: personal | business
        +----------------------+
                   |
        +----------+----------+
        |                     |
        v                     v
   ynab-coach           ynab-profit-first        <- the brains: read, diagnose, propose
        |                     |
   /ynab-daily          /ynab-reconcile          <- the hands: the actual run procedure
        |                     |
        +----------+----------+
                   |
                   v
            propose -> you approve -> write
```

The frameworks in [`docs/`](docs/) are what the skills cite. The
[runbook](business/reconciliation-runbook.md) is what the business skill treats as law.

## Quickstart

**1. Wire up a connector.** Any MCP server that exposes the YNAB API works. Match the
tool names in [`docs/tool-reference.md`](docs/tool-reference.md) and the skills work
unchanged. Full walkthrough with a working server sketch:
[`docs/connector-setup.md`](docs/connector-setup.md).

**2. Install the skills.**

```bash
git clone https://github.com/thejetmansam/ynab-claude-skills
cd your-project
mkdir -p .claude/skills .claude/commands

# personal
cp -r ../ynab-claude-skills/personal/skills/ynab-coach .claude/skills/
cp ../ynab-claude-skills/personal/commands/ynab-daily.md .claude/commands/

# business
cp -r ../ynab-claude-skills/business/skills/ynab-profit-first .claude/skills/
cp ../ynab-claude-skills/business/commands/ynab-reconcile.md .claude/commands/
```

**3. Fill in the config block** at the top of whichever skill you installed. Nothing is
hardcoded, and nothing should be.

**4. Run a diagnose.** Ask Claude "how does the budget look?" and read what comes back
before you let it write anything.

## Documentation

| File | What it is |
|---|---|
| [connector-setup.md](docs/connector-setup.md) | Wiring an MCP server to the YNAB API, with a working sketch |
| [tool-reference.md](docs/tool-reference.md) | All 13 tools, real signatures, and the traps |
| [lessons-learned.md](docs/lessons-learned.md) | **Start here.** What cost real money to learn |
| [four-rules.md](docs/four-rules.md) | The Mecham framework, distilled with chapter citations |
| [four-rules-index.md](docs/four-rules-index.md) | One-page quick reference |
| [profit-first.md](docs/profit-first.md) | The Michalowicz framework, distilled |
| [profit-first-index.md](docs/profit-first-index.md) | One-page quick reference |
| [chapter-map.md](docs/chapter-map.md) | What each cited chapter covers, for both books |
| [reconciliation-runbook.md](business/reconciliation-runbook.md) | The business system of record, as a fillable template |
| [books/](books/README.md) | The source books, for checking a citation against the original |

## Three rules that are not optional

1. **Reads are free, writes are gated.** Never let an assistant mutate a budget without
   explicit per-batch approval in the same session. The skills are written to propose in
   plain text and wait. Keep that property if you modify them; it is the only thing
   standing between a language model and your register.

2. **Never guess a category id.** Call `ynab_list_categories()` every run and build ids
   from that read. A stale id does not throw an error, it writes money somewhere wrong
   and silently. The bulk endpoint will even report success.

3. **Mind the rate limit.** 200 requests per hour, per token. Use
   `ynab_update_transactions_bulk` instead of looping single updates, or a real backlog
   will lock you out at request 200 with the job half done.

## Traps worth knowing before you start

- A refund inherits the **payee's** last category, not the category of the purchase being
  refunded. Refunds can hide for months in a category with almost no history.
- Never diff a statement on `(date, amount)`. Issuers report transaction date, YNAB
  imports posting date, so a strict match **turns a couple of real differences into
  hundreds of false ones.** Match on amount with a plus or minus 7 day window.
- The bulk write endpoint **silently accepts a mistyped category id** and reports success.
- `to_be_budgeted` is the only true Ready to Assign. The "Inflow: Ready to Assign"
  category balance carries a lifetime artifact that can grow enormous. It is not cash.

The rest are in [lessons-learned.md](docs/lessons-learned.md).

## On the books

The two frameworks belong to their authors. The files in `docs/` are **summary and
commentary written for operational use**, with chapter pointers so the citations in the
skills resolve. They skip the stories and worked examples that make the ideas land the
first time.

- Jesse Mecham, *You Need a Budget* (HarperBusiness, 2017).
  [Buy on Amazon](https://www.amazon.com/dp/0062567586)
- Mike Michalowicz, *Profit First* (Obsidian Press, 2014).
  [Buy on Amazon](https://www.amazon.com/dp/073521414X)

The source text for both is in [`books/`](books/README.md), for checking a citation
against the original. The Profit First citations follow the 2014 edition; the in-print
2017 revision renumbers some chapters.

## Contributing

Issues and pull requests welcome, particularly:

- Connector implementations for other MCP hosts
- Traps you have hit that are not in `lessons-learned.md`
- Modes that turned out to be useful in practice

Please do not open a PR containing real budget data, account identifiers, or balances.

## Disclaimer

This is not financial, tax, or accounting advice. These skills automate bookkeeping
mechanics against your own data. You are responsible for every number that reaches your
accountant.

## License

MIT. See [LICENSE](LICENSE).
