# FinBot

Personal finance over Telegram, in Brazilian Portuguese.

FinBot lets a user record income and expenses from short Portuguese sentences, keep more than one account, and ask a small set of balance and spending questions. Entries go through a confirmation draft before anything is stored.

## Status

Specs-first. The MVP is specified under [`.specs/features/finbot-mvp/`](.specs/features/finbot-mvp/); application code lands next.

## Stack

| Layer | Technology |
| --- | --- |
| Backend | NestJS + TypeScript |
| Database | PostgreSQL + TypeORM |
| Interface | Telegram Bot API (grammY long polling) |
| Testing | Jest |
| Runtime | Node.js `>=22.22.3` |

Money is stored as integer centavos (BRL only). User-facing replies are pt-BR; code, specs, and commits are English.

## MVP scope

- Telegram identity (one Telegram user → one FinBot user)
- Accounts and categories
- Income and expense drafts with confirm / edit / cancel
- Balance, history, and a fixed set of Portuguese questions
- Per-user data isolation

Out of scope for the MVP: budgets, Open Finance, statement import, language models, web/mobile UI, multi-currency, shared households.

## Repository layout

```
.specs/     Project constitution, decisions, and feature specs
src/        Application source (to be implemented)
```

## Local development

Setup will land with the NestJS scaffold. Expect at least:

1. Copy env vars (bot token, database URL) from an example file — never commit secrets.
2. Start PostgreSQL.
3. Install dependencies and run the Nest process (grammY long polling; no HTTP server in the conversational MVP).

## License

MIT. See [LICENSE](LICENSE).
