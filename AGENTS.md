# Up Banking API — Agent Instructions

This file is consumed by AI agents (Hermes, Claude Code, etc.) that need to operate this skill correctly. Humans should read [README.md](README.md) instead.

## When to load this skill

- The user asks about their bank balance, spending, or recent transactions
- The user wants to categorise a transaction or manage tags
- The user mentions their Up bank account

## Credential resolution

The helper script reads the token from `$UP_BANKING_PAT` in the environment and
nowhere else. Hermes injects it into new sessions from the value stored with
`hermes config set UP_BANKING_PAT <token>`; for a standalone run, export it first.
Keep the script free of any credentials-file parsing — Hermes' skill scanner
hard-blocks skills that read its config files.

The token includes the `up:yeah:` prefix — do not strip it. Store the full string.

## Script invocation

Resolve the script directory from wherever the skill is installed:

```bash
SCRIPT_DIR=$(dirname "$(find "${HERMES_HOME:-$HOME/.hermes}/skills" -path '*up-banking-api*/scripts/up_client.py' | head -1)")
UP="$SCRIPT_DIR/up_client.py"
ANALYSIS="$SCRIPT_DIR/analysis.py"
```

Then call `python "$UP" <command> [flags]`.

All output is plain-text tabular format on stdout. Errors go to stderr and exit non-zero. `transactions` follows `links.next` and stops silently at `--limit` (default 1000) — raise it for deeper history.

## Available commands

| Command | Flags | Returns |
|---------|-------|---------|
| `ping` | — | row: `pong — <uuid>` |
| `accounts` | — | table: id, name, balance, type |
| `account` | `--account-id <id>` | labelled fields: ID, Name, Balance, Type, Created |
| `transactions` | `--account-id <id>`, `--page-size <n>`, `--json` | table or full JSON with all attributes |
| `categories` | — | table: id, name |
| `webhooks` | — | table: id, url, active |

Pass `--json` to `accounts`, `account`, `transactions`, or `categories` to get the full JSON:API resource objects instead of formatted tables. Pipe through `jq` or use the analysis script for spending breakdowns:

```bash
# Spending summary by category (excludes internal transfers)
python "$ANALYSIS" summary --days 30

# HTML spending widget (generates a ::preview for Hermes desktop)
python "$ANALYSIS" widget --days 14
```

The widget command outputs a `::preview{file=...}` directive. In the Hermes desktop app this renders as a live styled card inline in the chat. In other environments the file path is printed on stderr.

## Data model notes (learned from analysis)

- **Amount sign**: negative = money out (debit), positive = money in (credit).
- **Internal transfers** between the user's own accounts appear as transactions and **must be excluded** from spending/income/savings math. The analysis script excludes these automatically via a regex matching `Transfer from/to`, `Cover from/to`, `Quick save transfer from/to`, `Round Up`, `Final interest payment from`.
- **2Up / joint accounts**: global `transactions` (without `--account-id`) includes both individual AND joint (2Up) accounts. When breaking down expenses be aware joint household bills will appear. Use `--account-id <id>` to isolate.
- **Categories are hierarchical**: parent (e.g. `home`, `good-life`) → children (e.g. `groceries`, `eating-out`). Transactions carry a `category` slug; parent relationship is available in the full JSON.

## Transaction filters (CLI)

```bash
python scripts/up_client.py transactions --since 2026-09-01 --until 2026-09-30
python scripts/up_client.py transactions --category groceries
python scripts/up_client.py transactions --status HELD
python scripts/up_client.py transactions --account-id <uuid> --since 2026-09-01
python scripts/up_client.py transactions --limit 500      # follow pagination up to N
python scripts/up_client.py transactions --no-cache
```

Responses are cached 15 minutes to avoid rate limits. Pass `--no-cache` to force fresh data.

## Validation framework (MANDATORY for financial analysis)

Every financial analysis MUST present a validation section. Include the checks below that apply:

1. **Show your working** — report the date range, transaction count before and after filtering, and what exclusions were applied. E.g. "Based on 20 transactions from 2026-09-01 to 2026-09-30 (45 raw, 25 excluded: internal transfers)."
2. **Income = Expenses + Net** — verify the computed `income - expenses = net` balances. A mismatch means a filtering error.
3. **Category exhaustiveness** — sum the category totals and compare to the overall total. Report if >1% of spending is uncategorised.
4. **Spot-check samples** — show 2-3 sample transactions from the largest categories so the user can verify correct categorisation.
5. **2Up awareness** — flag whether joint account spending is included. If the user has a 2Up account and you didn't isolate it, say so.

## Hermes-exclusive features

### Spending widget (::preview)
Run `python scripts/analysis.py widget --days N` to generate a horizontal bar chart of your top spending categories. The HTML uses the app's theme variables (`--foreground`, `--muted-foreground`, `--border`) so it matches the current skin. Best at 14-30 day range.

### Cron — recurring reports
Use `cronjob_manage` to schedule daily balance snapshots or weekly spending digests:

```python
from hermes_tools import cronjob_manage

cronjob_manage(
    action="create",
    name="up-daily-balance",
    schedule="0 9 * * *",       # 9am daily
    goal="Run the Up Banking skill's scripts/up_client.py accounts and summarize the balances. Use ::preview to render an HTML table if the desktop app is available.",
)

cronjob_manage(
    action="create",
    name="up-weekly-spending",
    schedule="0 10 * * 1",      # 10am Monday
    goal="Run the Up Banking skill's scripts/analysis.py summary --days 7 and present the weekly spending breakdown.",
)
```

The cron goal text tells the agent what to do — it reads the skill, runs the script, and delivers the result to the user's home channel on the configured messaging platform.

### Large transaction alerts
Use `cronjob_manage` with a higher frequency and a custom curl command:

```python
cronjob_manage(
    action="create",
    name="up-large-tx-check",
    schedule="*/30 * * * *",    # every 30 min
    goal="Check the last 5 transactions from the Up Banking API. If any have an absolute amount over $100 and status HELD, alert the user with the transaction details.",
)
```

## Write operations (curl only)

The helper script does not wrap write operations. Use curl directly:

```bash
# Categorise a transaction
curl -s --globoff -X PATCH -H "Authorization: Bearer $UP_BANKING_PAT" \
  -H "Content-Type: application/json" \
  -d '{"data":{"type":"categories","id":"<category-id>"}}' \
  'https://api.up.com.au/api/v1/transactions/<tx-id>/relationships/category'

# Add tags
curl -s --globoff -X POST -H "Authorization: Bearer $UP_BANKING_PAT" \
  -H "Content-Type: application/json" \
  -d '{"data":[{"type":"tags","id":"<tag-name>"}]}' \
  'https://api.up.com.au/api/v1/transactions/<tx-id>/relationships/tags'

# Remove tags
curl -s --globoff -X DELETE -H "Authorization: Bearer $UP_BANKING_PAT" \
  'https://api.up.com.au/api/v1/transactions/<tx-id>/relationships/tags'
```

To find a category ID, run `python "$UP" categories` and grep the name.

## Pitfalls

- **The `up:yeah:` prefix is required** — never strip it. The API returns 401 without it.
- **Square brackets `[]` in query params** need URL-encoding (`%5B`/`%5D`) or `--globoff` on curl — raw brackets break bash's glob expansion.
- **The PAT grants full read access** to all accounts. Never log it, echo it, or include it in tool output meant for the user.
- **If the helper script exits with "UP_BANKING_PAT is not set"**, the env var is missing from this session. Re-run `hermes config set UP_BANKING_PAT <token>`; values are injected into new sessions, so start a new session if you just set it.
- **Paginated responses** carry a `links.next` URL in the JSON body; the helper follows it until `--limit`. Don't hand-roll pagination by fetching `links.next` yourself — raise `--limit` instead.