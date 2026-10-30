---
name: up-banking-api
description: "Query Up bank: accounts, transactions, balances, spending."
version: 1.2.1
author: "kyoto-aircon, Hermes Agent"
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [finance, up-bank, banking, transactions, open-banking]
    category: finance
    related_skills: [stocks]
---

# Up Banking API Skill

Read-only access to Up (Australian neobank) account data through their public REST API.
Requires a Personal Access Token scoped to the user's own account.

## Links

| What | URL |
|------|-----|
| Up website | https://up.com.au |
| Features product tree | https://up.com.au/tree/ |
| Support | https://up.com.au/support/ |
| API documentation | https://developer.up.com.au |
| API reference for agents (llms.txt) | https://developer.up.com.au/llms.txt |
| Token portal | https://api.up.com.au |
| OpenAPI spec (GitHub) | https://github.com/up-banking/api |
| Community projects | https://github.com/up-banking/api/blob/master/community/EXAMPLES.md |
| API changelog | https://github.com/up-banking/api/issues/31 |

## When to Use

- User asks for their bank balance, recent transactions, or spending breakdown
- User wants to categorise a transaction or add/remove tags
- User mentions Up bank, their Up account, or asks about money in/out
- User wants to set up webhook notifications for new transactions
- Don't use for: other Australian banks' CDR/Open Banking data (separate API per bank)

## Prerequisites

1. An Up bank account (get it at https://up.com.au)
2. A Personal Access Token from the Up app: swipe right -> Data sharing -> Personal Access Token -> Generate a token. Tokens can be issued with an expiry, so note the date and reissue before it lapses
3. Store the token with `hermes config set UP_BANKING_PAT <token>` — the full string including the `up:yeah:` prefix. Hermes injects it into new sessions as `$UP_BANKING_PAT`
4. Python 3.8+ and `curl`

## How to Run

Resolve this skill's script directory once (works wherever the skill is installed):

```bash
SCRIPT_DIR=$(dirname "$(find "${HERMES_HOME:-$HOME/.hermes}/skills" -path '*up-banking-api*/scripts/up_client.py' | head -1)")
UP="$SCRIPT_DIR/up_client.py"
ANALYSIS="$SCRIPT_DIR/analysis.py"
```

Then:

```bash
python "$UP" ping
python "$UP" accounts
python "$UP" transactions
python "$UP" transactions --page-size 5
python "$UP" transactions --account-id <uuid>
python "$UP" account --account-id <uuid>
python "$UP" categories
python "$UP" notes
python "$UP" webhooks
```

Spending analysis (requires Hermes for the `::preview` widget):

```bash
python "$ANALYSIS" summary --days 30
python "$ANALYSIS" widget --days 14    # outputs ::preview directive
```

Or directly with curl (the `$UP_BANKING_PAT` env var is available inside Hermes sessions):

```bash
curl -s --globoff -H "Authorization: Bearer $UP_BANKING_PAT" \
  'https://api.up.com.au/api/v1/util/ping'

curl -s --globoff -H "Authorization: Bearer $UP_BANKING_PAT" \
  'https://api.up.com.au/api/v1/accounts'

curl -s --globoff -H "Authorization: Bearer $UP_BANKING_PAT" \
  'https://api.up.com.au/api/v1/transactions?page%5Bsize%5D=20'
```

## Transaction notes (`message`)

Every transaction carries three separate text fields. They are easy to conflate, and only one of
them is written by the account holder:

| Field | Written by | Typical example | What it is |
|-------|-----------|-----------------|------------|
| `rawText` | The bank / card processor | `SQ *EXAMPLE CAFE, SYDNEY` | The raw descriptor. The `SQ *` prefix is the processor and `SYDNEY` is where it cleared — neither names the merchant. |
| `description` | Up | `Example Cafe` | Up's cleaned merchant name. Often still a cipher. |
| `message` | **The account holder** | `team lunch, split 4 ways` | **The user's own note.** Usually names the real vendor, the purpose, the recipient, or a claim reference. |

`message` is empty on most rows and is populated only where the account holder annotated the
transaction in the Up app. Where it exists it is the highest-value field on the row:

- **Read it before asking the user what a charge was.** A transaction whose description is an
  opaque processor string is frequently already explained in `message`. Asking anyway wastes a
  round trip and makes the agent look like it never read the data it fetched.
- **Use it for reconciliation.** Notes commonly carry claim, invoice, or reference numbers that
  tie a credit back to a statement.
- **Let it decide what the transaction was, not how to book it.** A note reading `rent share`
  means the outflow was a shared rent payment even if Up left the row `uncategorised`; booking
  it is still a judgement call, but the fact of what happened is settled by the note.
- **A note can contradict the category.** Treat the note as more reliable about intent, then
  categorise deliberately rather than inheriting Up's label.

```bash
python "$UP" notes     # every annotated transaction in one call
```

The formatted `transactions` table does **not** print `message`. Use `notes`, or
`transactions --json`, to read notes.

## Commands

### `ping`
Connection health check. Returns a UUID and a lightning bolt if the token is valid.

### `accounts`
List all accounts with ID, display name, balance, and type (TRANSACTIONAL or SAVER).

### `account --account-id <id>`
Get a single account by its UUID. Shows the same fields as `accounts` but for one account.

### `transactions [--account-id <id>] [--page-size N] [--json]`
Recent transactions, newest first. The script requests 100 per page by default (the API's own default varies and is often 20); scope to one account with `--account-id`, adjust with `--page-size` (the API caps pages at 100). Pass `--json` for structured output the agent can pipe through `jq` or Python for spending analysis — see the For AI Agents section in the README.

`--page-size` sets the page size **per request**, not a cap on results — the helper then follows
`links.next` up to `--limit` (default 1000). Use `--limit` to bound a query.

Rows are ordered by `createdAt`, not `settledAt`. A HELD row that settles days later appears out
of order against the settled date, so sort explicitly before any date-bounded reconciliation.

### `notes`
Every transaction carrying a non-empty `message`, with date, amount, status, category and
description, followed by the note itself. This is the fast way to see the account holder's own
annotations without dumping JSON.

### `categories`
List all transaction categories with IDs.

### `webhooks`
List registered webhook endpoints with their active status.

### `analysis summary --days N`
Spending breakdown by category — total in/out, net, top categories with percentages. Uses `--json` output internally.

### `analysis widget --days N`
Generates a themed HTML bar chart as a `::preview` widget for the Hermes desktop app. Renders spending by description with proportional bars.

### Write operations (curl only)
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

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | /util/ping | Health check |
| GET | /accounts | List all accounts |
| GET | /accounts/{id} | Single account |
| GET | /accounts/{id}/transactions | Account-scoped transactions |
| GET | /transactions | All transactions |
| GET | /transactions/{id} | Single transaction |
| PATCH | /transactions/{id}/relationships/category | Set category |
| POST | /transactions/{id}/relationships/tags | Attach tags |
| DELETE | /transactions/{id}/relationships/tags | Detach tags |
| GET | /categories | List all categories |
| GET | /categories/{id} | Single category |
| GET | /tags | List all tags |
| GET | /attachments | List attachments |
| GET | /attachments/{id} | Single attachment (temporary `fileURL`) |
| POST | /webhooks | Create webhook |
| GET | /webhooks | List webhooks |
| GET | /webhooks/{id} | Single webhook |
| DELETE | /webhooks/{id} | Delete webhook |
| POST | /webhooks/{webhookId}/ping | Test webhook |
| GET | /webhooks/{webhookId}/logs | Webhook delivery logs |

## Data Model

Responses follow JSON:API spec (`data` -> array of resource objects, `links` for pagination).

**Account:** `id`, `displayName`, `accountType` (TRANSACTIONAL or SAVER), `ownershipType`, `balance` (currencyCode + value + valueInBaseUnits), `createdAt`.

**Transaction:** `id`, `status` (HELD or SETTLED), `rawText`, `description`, **`message`** (the account holder's own note — see Transaction notes above), `amount` (currencyCode + value + valueInBaseUnits), `settledAt`, `createdAt`, category/tags relationships.

**Attachment:** `id`, `createdAt`, `fileURL` (temporary download link), `fileURLExpiresAt`, `fileExtension`, `fileContentType`. Fetch a fresh `fileURL` when it expires.

## Pitfalls

- **The formatted transaction table does not print `message`.** Notes are visible only via the `notes` command or `--json`. Treating the bare table as the whole record silently drops the account holder's annotations, and then asks them to repeat what they already wrote.
- **`rawText`, `description` and `message` are three different things.** Conflating them turns a processor string into a merchant name, or a user's note into bank data.
- **Square brackets in query params** need the `--globoff` flag on curl, or URL-encode as `%5B`/`%5D` to avoid bash glob expansion.
- **Token includes the `up:yeah:` prefix** — store the full string as returned by the app. The API requires this prefix; stripping it causes a 401 error.
- **Rate limits** are undocumented but generous. Pages cap at `page[size]=100`; the helper follows `links.next` up to `--limit` (default 1000), so prefer its pagination over hand-rolled loops.
- **The PAT is personal** — it grants full read access to your accounts. Store it with `hermes config set UP_BANKING_PAT`; Hermes redacts it from logs and injects it as an env var in new sessions. Never echo it.
- **Webhooks** require a public callback URL. This skill lists and manages webhooks but does not include a receiver.
- **MSYS shell** on Windows: use `C:/...` paths for native tools; avoid `/tmp` for persistent files.

## Verification

```bash
SCRIPT_DIR=$(dirname "$(find "${HERMES_HOME:-$HOME/.hermes}/skills" -path '*up-banking-api*/scripts/up_client.py' | head -1)")
UP="$SCRIPT_DIR/up_client.py"
python "$UP" ping          # -> pong - <uuid>
python "$UP" accounts      # -> at least one account row
python "$UP" transactions  # -> at least one transaction row
python "$UP" notes         # -> annotated rows, or a message saying there are none
```

## Limitations

- **Read-only with exceptions:** categorisation and tagging are write endpoints; everything else is read-only. No funds transfer or payment initiation.
- **Beta API:** Endpoints may change. Features like scheduled payments and statements are not yet available via the API.
- **Personal scope only:** the PAT is scoped to the user's own accounts. No organisation/business access through this token type.
