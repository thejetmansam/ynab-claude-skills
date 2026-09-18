# Connector setup

The skills read and write a live budget through an MCP server that wraps the YNAB API.
Without it they have nothing to work on. This describes a shape that holds up in daily
use, and what matters regardless of how you build yours.

## What you need

1. **A YNAB personal access token.** YNAB account settings, Developer Settings, New
   Token. It is scoped to your whole account and there is no read-only variant, so
   treat it as a write credential from day one.
2. **An MCP server exposing YNAB as tools.** Any implementation works. The tool surface
   in `tool-reference.md` is what the skills call; match those names and signatures and
   the skills work unchanged.
3. **A place to keep the token that is not this repository.** Environment variable,
   1Password, your platform's secret store. Never a file in the repo, never pasted into
   a chat, never a value printed to a log.

## The multi-account pattern

If you run a household budget and a company budget, you have two YNAB accounts and two
tokens. The single most useful design decision is making **every tool take an explicit
`account` argument** with no default:

```
ynab_get_month(account="personal", month="current")
ynab_get_month(account="business", month="current")
```

There is no ambient default to get wrong. A skill that forgets to pass it gets an error
instead of silently reading the wrong budget, and an allocation proposed against the
wrong entity's numbers is the kind of mistake you only want to make in theory.

Map each value to its own token:

```
YNAB_TOKEN_PERSONAL=...
YNAB_TOKEN_BUSINESS=...
```

## Server shape

A minimal FastMCP server, sketched:

```python
import os
import httpx
from fastmcp import FastMCP

mcp = FastMCP("ynab")

TOKENS = {
    "personal": os.environ["YNAB_TOKEN_PERSONAL"],
    "business": os.environ["YNAB_TOKEN_BUSINESS"],
}
BASE = "https://api.ynab.com/v1"


def _client(account: str) -> httpx.Client:
    if account not in TOKENS:
        raise ValueError(f"unknown account: {account}")
    return httpx.Client(
        base_url=BASE,
        headers={"Authorization": f"Bearer {TOKENS[account]}"},
        timeout=30.0,
    )


def _dollars(milliunits: int | None) -> float | None:
    """YNAB speaks milliunits. Everything above this layer speaks dollars."""
    return None if milliunits is None else milliunits / 1000.0


@mcp.tool()
def ynab_get_month(account: str, budget_id: str = "last-used", month: str = "current"):
    """Budget versus actual for a month. Returns dollars, not milliunits."""
    with _client(account) as c:
        r = c.get(f"/budgets/{budget_id}/months/{month}")
        r.raise_for_status()
        m = r.json()["data"]["month"]
    return {
        "month": m["month"],
        "income": _dollars(m["income"]),
        "budgeted": _dollars(m["budgeted"]),
        "activity": _dollars(m["activity"]),
        "to_be_budgeted": _dollars(m["to_be_budgeted"]),
        "categories": [
            {
                "id": c_["id"],
                "name": c_["name"],
                "budgeted": _dollars(c_["budgeted"]),
                "activity": _dollars(c_["activity"]),
                "balance": _dollars(c_["balance"]),
            }
            for c_ in m["categories"]
            if not c_["hidden"] and not c_["deleted"]
        ],
    }
```

**Convert milliunits at the server boundary, once.** If dollars and milliunits both
reach the model, it will eventually mix them, and a factor-of-1000 error in a write is
not a subtle bug. Every tool in the reference returns dollars for this reason.

## Deployment

Anywhere that holds a secret and speaks HTTP; a small hosted container is plenty.
Remote deployment matters more than it sounds: it means a Claude session on any
machine, including headless and cloud sessions, can reach the budget without a local
install.

If your MCP host needs a session handshake (FastMCP does: `initialize`, then
`notifications/initialized`, then `tools/call`), a bare curl will fail with a missing
session id. Use a real MCP client rather than fighting it.

## Rate limit

**200 requests per hour per token.** This is the constraint that shapes everything:

- A daily pass pulling transactions, categories, payees, accounts, and the month costs
  5 requests. Fine.
- Categorizing 80 transactions with `ynab_update_transaction` in a loop costs 80. Also
  technically fine, until you do it twice and then need to reconcile.
- Categorizing 80 transactions with `ynab_update_transactions_bulk` costs **1**.

Use the bulk tool. The skills are written to. When you exceed the limit YNAB returns
429 and you are locked out for the remainder of the hour, mid-pass, with half the
backlog categorized and no clean way to tell where you stopped.

## Verify the wiring

```
ynab_list_budgets(account="personal")
```

You should get back your budgets with ids. If that works, point the skills at it and
run Diagnose. If it returns 401, the token is wrong. If it returns 404 on a budget id
that exists, you are probably passing a budget id from the other account.

## Security notes

- The token is full read and write on your real money. There is no read-only mode.
- Nothing in this repo should ever contain a token, a budget id, an account id, or a
  balance. The `.gitignore` covers the obvious cases; it cannot cover a paste.
- The skills are written to propose before writing. Keep that property if you modify
  them. It is the only thing standing between an LLM and your register.
