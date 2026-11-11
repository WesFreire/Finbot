# FinBot Constitution

Rules for every feature. A spec may narrow them. It may not break them.

## Language

Every user-facing bot message, button label, command example, error, and notification is Brazilian Portuguese (pt-BR).

Code, specs, commit messages, and logs are English.

## Spec before code

A feature is implemented only from a confirmed spec whose acceptance criteria carry traceable requirement IDs.

If the spec and the code disagree, the spec wins until someone changes the spec.

## Dependency order

Build in this order: identity and data isolation, then accounts, then categories, then transactions, then queries.

A feature does not land before the feature it depends on.

## Incremental delivery

Each user story is independently demoable through Telegram.

Ship the smallest story that a person can try, then the next.

## Tests

Every acceptance criterion has an automated test that asserts the outcome written in the spec.

The test runner is Jest.

A criterion without a failing-then-passing test is not done.

## Clean architecture

Separate domain, application, and infrastructure. Do not add a fourth layer.

Domain code does not import NestJS, TypeORM, the Telegram client, Redis, BullMQ, or HTTP clients.

Infrastructure implements interfaces the application needs. The domain does not know which database or chat app is attached.

## Security

One Telegram user reads and changes only that user's accounts, categories, drafts, and transactions.

Every query filters by the resolved user. There is no admin path in the MVP that skips that filter.

Money is stored as integer centavos. Floating-point money is not allowed.

Secrets (bot token, database URL) stay out of the repo and out of logs.

Logs include the Telegram update id and the resolved intent name. Logs omit the raw message text and the amount.

## Idempotency

Processing the same Telegram update twice has the same stored result as processing it once.

Confirming a draft that is already confirmed does not insert a second ledger row.

## Stack

| Layer | Technology |
| --- | --- |
| Backend | NestJS + TypeScript |
| Database | PostgreSQL |
| ORM | TypeORM |
| Interface | Telegram Bot API |
| Infrastructure | Docker + Docker Compose |
| Documentation | Swagger / OpenAPI |
| Testing | Jest |
| CI/CD | GitHub Actions |
| Background processing | BullMQ + Redis |

The MVP does not add a language model or an Open Finance provider.

BullMQ, Redis, Swagger, and Docker Compose are commitments for the product. A feature spec adds them when that feature needs a queue, an HTTP API, or a compose file. The conversational MVP does not require them to be accepted.

## MVP boundary

The first feature spec is the conversational MVP: Telegram identity, accounts, categories, income and expenses confirmed in chat, balances, history, and a fixed set of Portuguese questions.

These capabilities need their own specs later. This constitution does not specify them.

- Budgets
- Spending insights beyond the MVP question set
- Open Finance and bank connection
- Statement import and automatic categorization
- Reminders and notifications that are not the reply to the current message
- Any interface other than Telegram
