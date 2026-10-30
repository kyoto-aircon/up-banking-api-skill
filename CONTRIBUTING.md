# Contributing

Thanks for taking an interest. This repo is a Hermes Agent skill for the Up Banking API — small scope, small surface.

## How to contribute

1. Open an issue describing the change (bug, missing endpoint, docs fix).
2. Fork the repo, make your change on a descriptive branch.
3. Open a pull request linking the issue.

Keep changes scoped to the skill and its helper script. If you're adding a new API endpoint, make sure it exists in the live API first (check https://developer.up.com.au).

## Standards

- **SKILL.md** — follows the [Hermes Agent skill authoring standard](https://github.com/NousResearch/hermes-agent). Description ≤ 60 chars, frontmatter complete, no personal data.
- **scripts/up_client.py** — Python stdlib only (no pip deps). argparse-style flags, stderr for errors, stdout for data.
- **README.md** — keep the For AI Agents section accurate. It's what agents read when deciding how to invoke the skill.
- **No personal data** — account IDs, balances, or tokens must never appear in committed files.

## Upstream PR

Status: **not yet submitted** — the skill still lives only in this repo. The target is
`NousResearch/hermes-agent/optional-skills/finance/up-banking-api/` (optional tier, `finance`
category). An upstream PR also needs:

- `tests/skills/test_up_banking_api_skill.py` — stdlib + pytest, no live network
- a regenerated docs page (`python website/scripts/generate-skill-docs.py`)

Main-repo review standards apply there: description ≤ 60 chars, `author` crediting the
human first, no machine-local paths. This repo remains the development staging ground.

## Questions

Open an issue. If it's about the Up API itself (endpoint behaviour, missing features), direct it to https://github.com/up-banking/api instead.