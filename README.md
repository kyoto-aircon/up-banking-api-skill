# Up Banking API — Hermes Agent Skill

> **AI-generated disclaimer:** This skill was developed with the assistance of an AI agent (Hermes Agent). It has been tested against the live Up API and verified working. Review the code before use, especially the credential-handling in `scripts/up_client.py`.

Read-only access to [Up Bank](https://up.com.au) (Australian neobank) account data through their public REST API. Check balances, list transactions, categorise spending, manage tags, and set up webhooks — all from within a Hermes Agent session.

## Scope

- **Accounts** — list all accounts with balances, types, and IDs
- **Transactions** — recent transactions across all accounts or per account, with adjustable page size
- **Transaction notes** — read the notes you have attached to individual transactions (the `message` field)
- **Categories & Tags** — list, categorise transactions, attach/remove tags
- **Attachments** — list receipt/document attachments (temporary download URLs)
- **Webhooks** — create, list, and manage webhook notifications
- **Ping** — connection health check

This is a **personal-scope** integration. The API token grants access only to the user's own accounts. No funds transfer, no payment initiation, no third-party access.

## Prerequisites

- [Up Bank](https://up.com.au) account
- Personal Access Token (Up app -> swipe right -> Data sharing -> Personal Access Token -> Generate)
- [Hermes Agent](https://hermes-agent.nousresearch.com) installed
- Python 3.8+ and `curl`

## Install

```bash
# Register the repo as a skill tap
hermes skills tap add kyoto-aircon/up-banking-api-skill

# Install the skill
hermes skills install kyoto-aircon/up-banking-api-skill/up-banking-api --category finance

# Store your API token
hermes config set UP_BANKING_PAT <your-token>

# Verify it works (new session picks up the token)
find "${HERMES_HOME:-$HOME/.hermes}/skills" -name up_client.py -path '*up-banking-api*'
# then cd to that directory and run: python scripts/up_client.py ping
```

Or install directly from URL:
```bash
hermes skills install https://raw.githubusercontent.com/kyoto-aircon/up-banking-api-skill/main/SKILL.md --name up-banking-api --category finance
```

## Usage

```bash
# From inside the installed skill directory:
python scripts/up_client.py ping
python scripts/up_client.py accounts
python scripts/up_client.py transactions
python scripts/up_client.py transactions --page-size 5
python scripts/up_client.py transactions --account-id <uuid> --page-size 50
python scripts/up_client.py notes
python scripts/up_client.py categories
python scripts/up_client.py webhooks
```

## Transaction notes

Every transaction has three text fields, and only one is yours:

- `rawText` — the raw card/bank descriptor, e.g. `SQ *EXAMPLE CAFE, SYDNEY`. The processor prefix and the clearing city, neither of which names the merchant.
- `description` — Up's cleaned-up merchant name, e.g. `Example Cafe`. Often still a cipher.
- `message` — **your own note**, written in the Up app. This is where you record the actual vendor, what it was for, who it was for, or a claim reference.

If you annotate transactions in the Up app, `up_client.py notes` prints every transaction that carries a note, with the date, amount, status, category and description alongside it. The plain `transactions` table stays terse and does not include notes — use `notes`, or `transactions --json`, when you need them.

## For AI Agents

See [AGENTS.md](AGENTS.md) — token handling, command reference, output formats, and pitfalls.

## API Coverage

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /util/ping | Health check |
| GET | /accounts | List all accounts |
| GET | /accounts/{id} | Get account |
| GET | /accounts/{id}/transactions | Account transactions |
| GET | /transactions | All transactions |
| GET | /transactions/{id} | Single transaction |
| PATCH | /transactions/{id}/relationships/category | Categorise |
| POST | /transactions/{id}/relationships/tags | Add tags |
| DELETE | /transactions/{id}/relationships/tags | Remove tags |
| GET | /categories | List categories |
| GET | /tags | List tags |
| GET | /attachments | List attachments |
| GET | /attachments/{id} | Get attachment |
| POST | /webhooks | Create webhook |
| GET | /webhooks | List webhooks |
| GET | /webhooks/{id} | Get webhook |
| DELETE | /webhooks/{id} | Delete webhook |

## Links

- [Up API Documentation](https://developer.up.com.au)
- [Up API reference for agents (llms.txt)](https://developer.up.com.au/llms.txt)
- [Up API OpenAPI Spec (GitHub)](https://github.com/up-banking/api)
- [Community API projects](https://github.com/up-banking/api/blob/master/community/EXAMPLES.md)
- [Up API changelog](https://github.com/up-banking/api/issues/31)
- [Claim your Personal Access Token](https://api.up.com.au)
- [Up Website](https://up.com.au)
- [Up Features & Product Tree](https://up.com.au/tree/)

## License

MIT — see [LICENSE](LICENSE).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
