# FinBot MVP Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/finbot-mvp/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Generated from codebase, project guidelines, and spec - confirm before Execute. Guidelines found: `.specs/constitution.md` (Jest; every acceptance criterion has an automated test that asserts the spec outcome). No test files exist yet. Commands below are the scripts T1 registers. Strong defaults applied for depth.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| ---------- | ------------------ | -------------------- | ---------------- | ----------- |
| Process bootstrap | unit | Nest application object is created without listening on a port | `src/main.spec.ts` | `npm test` |
| Tooling config | none | build gate only | `package.json`, `eslint.config.mjs`, `tsconfig.json` | build gate only |
| Domain | unit | All branches; 1:1 to the domain rules in the design; every listed edge case that is a pure function | `src/domain/**/*.spec.ts` | `npm test` |
| Application | unit | 1:1 to the requirement IDs on that task; happy path, each refusal reply, and the edge cases those IDs repeat | `src/application/**/*.spec.ts` | `npm test` |
| Application ports | none | types only, build gate | `src/application/ports.ts` | build gate only |
| Persistence schema | none | build gate only; behavior is asserted through the repository | `src/infrastructure/persistence/schema.sql` | build gate only |
| Persistence repository | integration | Owner filter, case-insensitive unique names, one pending draft, one ledger row per draft, no balance column, processed-update insert-once | `src/infrastructure/persistence/**/*.integration.spec.ts` | `npm run test:integration` |
| Telegram adapter | unit | Button labels and callback data, reply-failure log omits the amount, text updates and callbacks map to `IncomingUpdate` | `src/infrastructure/telegram/**/*.spec.ts` | `npm test` |
| Nest module wiring | integration | Module boots against Testcontainers and one confirmed expense is readable back through the repository with a fake transport | `src/app.module.integration.spec.ts` | `npm run test:integration` |
| Test harness | integration | Container starts and `SELECT 1` returns 1 | `test/postgres.integration.spec.ts` | `npm run test:integration` |

## Gate Check Commands

> Generated from the constitution (Jest) and the scripts T1 adds. Confirm before Execute.

| Gate Level | When to Use | Command |
| ---------- | ----------- | ------- |
| Quick | After tasks with unit tests only | `npm test` |
| Full | After tasks with integration tests | `npm test && npm run test:integration` |
| Build | After phase completion or config/entity-only tasks | `npm run lint && npm run build && npm test && npm run test:integration` |

`npm test` runs Jest on `src/**/*.spec.ts` and ignores `*.integration.spec.ts`. `npm run test:integration` runs `*.integration.spec.ts` only. The integration config sets a 120 second hook timeout for the container startup.

---

## Execution Plan

Phases are ordered and run sequentially. Tasks inside a phase run in order. 24 tasks pack into four batches of whole phases (about eight, six, five, and five tasks). Execute offers batch sub-agents before the first batch; it does not start them unless you accept.

### Phase 1: Foundation

```
T1 → T2
```

### Phase 2: Domain

```
T3 → T4 → T5 → T6 → T7 → T8
```

### Phase 3: Parser

```
T9
```

### Phase 4: Identity and reference data

```
T10 → T11 → T12 → T13 → T14
```

### Phase 5: Drafts and questions

```
T15 → T16 → T17 → T18 → T19
```

### Phase 6: Persistence

```
T20 → T21
```

### Phase 7: Telegram wiring

```
T22 → T23 → T24
```

---

## Task Breakdown

### Phase 1: Foundation

### T1: Scaffold the NestJS 12 process

**What**: Add a compiling NestJS 12 app that builds a Nest application and does not call `listen`. Register `npm test`, `npm run test:integration` (with `--passWithNoTests` until T2), `npm run lint`, and `npm run build`. ESLint forbids `@nestjs/*`, `typeorm`, `grammy`, and `src/infrastructure/**` imports from `src/domain/**` and `src/application/**`.
**Where**: `src/main.ts`
**Depends on**: None
**Reuses**: none (empty repo)
**Requirement**: foundation

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] `package.json` engines field is `>=22.22.3` and depends on NestJS 12.4.x, `@nestjs/typeorm` 12.0.x, `typeorm` 1.1.x, `grammy`, `pg`, Jest, and TypeScript 6
- [ ] `src/main.spec.ts` asserts a Nest application object is created
- [ ] `npm test` passes
- [ ] `npm run lint` passes
- [ ] Test count: 1 test passes

**Tests**: unit
**Gate**: quick

**Commit**: `feat(app): scaffold the NestJS process`

---

### T2: Start PostgreSQL from the integration suite

**What**: Add a Jest integration project that starts `postgres:18-alpine` with `PostgreSqlContainer`, exports a helper that returns the connection URI, and runs `SELECT 1`.
**Where**: `test/postgres.integration.spec.ts`
**Depends on**: T1
**Reuses**: `package.json` scripts from T1
**Requirement**: foundation (AD-002)

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] The test reads `getConnectionUri()`, runs `SELECT 1`, and stops the container
- [ ] Gate check passes: `npm test && npm run test:integration`
- [ ] Test count: 1 integration test passes, and the T1 unit test still passes

**Tests**: integration
**Gate**: full

**Commit**: `test(db): run integration tests against disposable Postgres`

---

### Phase 2: Domain

### T3: Parse and format centavos

**What**: Implement `parseAmount`, `formatCentavos`, and `formatDigits` exactly as the design's Money section.
**Where**: `src/domain/money.ts`
**Depends on**: T2
**Reuses**: none
**Requirement**: TX-16

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] Accepted forms `R$ 42`, `R$ 42,00`, `R$ 1.234,56`, `R$ 3.500`, `R$ 1.000.000,00`, and ungrouped `R$ 3500` become the centavos in the spec table
- [ ] `R$42`, `R$ 42.50`, `R$ 42,5`, `R$ 0`, `R$ 0,00`, and `R$ 1.000.000,01` return `'invalid'`
- [ ] `formatCentavos(-4200)` is `-R$ 42,00` and `formatCentavos(350000)` is `R$ 3.500,00`
- [ ] Gate check passes: `npm test`
- [ ] Test count: 12 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(domain): parse and format BRL centavos`

---

### T4: Resolve São Paulo calendar dates

**What**: Implement `todayInSaoPaulo`, `addDays`, `resolveDate`, `formatDate`, and `isInMonth`.
**Where**: `src/domain/calendar-date.ts`
**Depends on**: T3
**Reuses**: none
**Requirement**: TX-17

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] A frozen instant at 02:30 UTC resolves to the previous civil date in America/Sao_Paulo
- [ ] `hoje`, `ontem`, a valid `D/M`, `31/02`, 29/02 in a non-leap year, and a future day in the current São Paulo year match the design
- [ ] `formatDate` zero-pads day and month
- [ ] Gate check passes: `npm test`
- [ ] Test count: 8 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(domain): resolve dates in America/Sao_Paulo`

---

### T5: Suggest at most one category

**What**: Implement `suggestCategory` with the Uber short-circuit and whole-word tie rule.
**Where**: `src/domain/suggest-category.ts`
**Depends on**: T4
**Reuses**: none
**Requirement**: TX-18, TX-19, TX-20

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] Exact `Uber` / `uber` returns the stored Transporte spelling when that name is owned
- [ ] One whole-word owned name matches; two owned names return null; a match inside `na conta` is not this function's job because the caller passes description only
- [ ] A name that is only a prefix of a longer word does not match
- [ ] Gate check passes: `npm test`
- [ ] Test count: 6 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(domain): suggest a single owned category`

---

### T6: Compute balances from ledger rows

**What**: Implement `accountBalance` and `totalBalance` as income centavos minus expense centavos.
**Where**: `src/domain/balance.ts`
**Depends on**: T5
**Reuses**: none
**Requirement**: TX-23

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] Mixed rows on two accounts match the Nubank `-4200` and Inter `350000` example, and the total is `345800`
- [ ] An empty list is 0
- [ ] Gate check passes: `npm test`
- [ ] Test count: 3 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(domain): compute account balances from confirmed rows`

---

### T7: Guard draft status transitions

**What**: Implement `confirm`, `cancel`, and `validateDescription`.
**Where**: `src/domain/draft.ts`
**Depends on**: T6
**Reuses**: none
**Requirement**: TX-03, TX-04, TX-05, TX-06, TX-25

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] Pending confirms and pending cancels; a second confirm returns `already_confirmed`; confirm or cancel of a cancelled draft returns the cancelled result
- [ ] Description trim accepts 1 to 80 characters and rejects empty or 81
- [ ] Gate check passes: `npm test`
- [ ] Test count: 6 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(domain): guard draft confirmation and cancellation`

---

### T8: Keep a single default account

**What**: Implement `validateName`, `markCreated`, `markDefault`, and `removeAccount` including the earliest-`createdAt` promotion and the id tie-break.
**Where**: `src/domain/account.ts`
**Depends on**: T7
**Reuses**: none
**Requirement**: ACCT-02, ACCT-03, ACCT-11, ACCT-12, ACCT-15

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] The first account is the only default, a second create stays non-default, and set-default moves the flag
- [ ] Deleting the default promotes the earliest remaining account; equal timestamps use the smaller id; deleting the only account returns `only_account`; `hasConfirmedTransactions` returns `has_transactions`
- [ ] Names trim to 1–40 characters
- [ ] Gate check passes: `npm test`
- [ ] Test count: 7 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(domain): keep exactly one default account`

---

### Phase 3: Parser

### T9: Parse the fixed Portuguese phrase set

**What**: Implement `parseMessage` in the order and shapes in the design's Parser section. Date clauses stay unvalidated. Amount lexemes that fail `parseAmount` still return the expense or income command with an invalid amount marker so the application can reply `Valor inválido` instead of `Não entendi`.
**Where**: `src/domain/parser.ts`
**Depends on**: T8
**Reuses**: `src/domain/money.ts`
**Requirement**: TX-26, QRY-15

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] Each account command, category command, `Saldo`, `Histórico`, the four questions (with and without `?`), the four edit phrases, and one expense and one income with `hoje` / `ontem` / `em D/M` / `na conta` parse to the matching `type`
- [ ] Text `Confirmar`, `Cancelar`, `Editar`, and `historico` return `unparsed`
- [ ] Captured account, category, and description text keep the user's casing
- [ ] Gate check passes: `npm test`
- [ ] Test count: 24 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(domain): parse the MVP Portuguese phrase set`

---

### Phase 4: Identity and reference data

### T10: Declare application ports

**What**: Add the `Repositories`, `UnitOfWork`, `Clock`, `Ids`, `AuditLog`, `Notifier`, `OutgoingMessage`, and `IncomingUpdate` types from the design. No runtime code.
**Where**: `src/application/ports.ts`
**Depends on**: T9
**Reuses**: domain types
**Requirement**: foundation

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] The file exports the types the later application tasks import
- [ ] Gate check passes: `npm run lint && npm run build && npm test && npm run test:integration`
- [ ] Test count: unchanged (no new test file)

**Tests**: none
**Gate**: build

**Commit**: `feat(app): declare ledger ports`

---

### T11: Implement the in-memory ledger

**What**: Add an in-memory `UnitOfWork` that commits only when the callback resolves, filters by `userId`, allows one pending draft per user, and treats `tryBegin` as insert-once.
**Where**: `src/application/in-memory-ledger.ts`
**Depends on**: T10
**Reuses**: `src/application/ports.ts`
**Requirement**: USER-04, TX-13

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] A rollback path (callback throws) leaves the maps unchanged
- [ ] User A's list does not return user B's account
- [ ] A second `tryBegin` for the same update id returns false
- [ ] Gate check passes: `npm test`
- [ ] Test count: 3 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(app): add an in-memory ledger for unit tests`

---

### T12: Resolve a Telegram user and seed categories

**What**: Implement `resolveUser`. First sight inserts the user and exactly two categories, Transporte then Alimentação. The same telegram id returns the same user and does not seed again.
**Where**: `src/application/resolve-user.ts`
**Depends on**: T11
**Reuses**: `src/application/in-memory-ledger.ts`
**Requirement**: USER-01, USER-02, CAT-01

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] A new id creates one user and the two seed categories in that order
- [ ] A second call inserts nothing
- [ ] Gate check passes: `npm test`
- [ ] Test count: 2 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(app): resolve a Telegram user and seed categories`

---

### T13: Execute account commands

**What**: Implement `executeAccount` for create, list, rename, set default, and delete, using the domain account rules and `accountBalance` for the figures.
**Where**: `src/application/account-commands.ts`
**Depends on**: T12
**Reuses**: `src/domain/account.ts`, `src/domain/balance.ts`
**Requirement**: ACCT-01, ACCT-02, ACCT-03, ACCT-04, ACCT-05, ACCT-06, ACCT-07, ACCT-08, ACCT-09, ACCT-10, ACCT-11, ACCT-12, ACCT-13, ACCT-14, ACCT-15

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] Each ACCT reply in the spec is asserted, including duplicate create, duplicate rename, delete with transactions, delete of the only account, missing name, and the 40-character bound
- [ ] After deleting the default, the earliest remaining account is the only default
- [ ] User B's account is invisible to user A (USER-04)
- [ ] Gate check passes: `npm test`
- [ ] Test count: 15 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(app): create, list, rename, and delete accounts`

---

### T14: Execute category commands

**What**: Implement `executeCategory` for create and list. Uniqueness ignores case. The list includes the seed plus later creates, oldest first.
**Where**: `src/application/category-commands.ts`
**Depends on**: T13
**Reuses**: `validateName` from `src/domain/account.ts`
**Requirement**: CAT-02, CAT-03, CAT-04, CAT-05

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] `Criar categoria Lazer` replies `Categoria Lazer criada.`
- [ ] The list after seed plus Lazer is Transporte, Alimentação, Lazer
- [ ] `transporte` is rejected with `Você já tem uma categoria com esse nome.`
- [ ] Empty and 41-character names get `Use um nome de categoria entre 1 e 40 caracteres.`
- [ ] Gate check passes: `npm test`
- [ ] Test count: 4 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(app): create and list categories`

---

### Phase 5: Drafts and questions

### T15: Open an expense or income draft

**What**: Implement `openDraft` and `renderDraft`. Valid sentences insert one pending draft and cancel a previous pending draft. Invalid amount, date, description, missing account, or unknown `na conta` insert nothing.
**Where**: `src/application/open-draft.ts`
**Depends on**: T14
**Reuses**: `src/domain/parser.ts`, `src/domain/suggest-category.ts`, `src/domain/calendar-date.ts`
**Requirement**: TX-01, TX-02, TX-13, TX-15, TX-16, TX-17, TX-18, TX-19, TX-20, TX-21, TX-22, TX-25

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] `Gastei R$ 42 no Uber hoje` with default Nubank and Transporte matches the spec card and inserts no ledger row
- [ ] `Recebi R$ 3.500 de salário` stores 350000 centavos, empty category, today, Nubank
- [ ] Uber, a single whole-word category, and a two-category tie follow TX-18, TX-19, and TX-20
- [ ] `na conta Inter` selects Inter; an unknown account does not create a draft
- [ ] No account, bad amount, bad date, and bad description return the spec replies and leave drafts unchanged
- [ ] A second expense cancels the previous pending draft
- [ ] Gate check passes: `npm test`
- [ ] Test count: 14 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(app): open a pending expense or income draft`

---

### T16: Edit a pending draft

**What**: Implement `editDraft` and `editHelp`. Unknown category or account keeps the previous value.
**Where**: `src/application/edit-draft.ts`
**Depends on**: T15
**Reuses**: `renderDraft` from `src/application/open-draft.ts`
**Requirement**: TX-07, TX-08, TX-09, TX-10, TX-11, TX-12

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] `valor R$ 50,00`, `categoria Alimentação`, `descrição Corrida`, and `conta Inter` each update that field and return the card with buttons
- [ ] Editar help returns the instruction sentence and leaves status pending
- [ ] A missing category or account returns the spec not-found reply and keeps the previous id
- [ ] Gate check passes: `npm test`
- [ ] Test count: 6 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(app): edit a pending draft from chat`

---

### T17: Confirm or cancel a draft

**What**: Implement `confirmDraft` and `cancelDraft`. Confirm inserts one ledger row. A second confirm does not insert another. Cancel inserts none.
**Where**: `src/application/confirm-draft.ts`
**Depends on**: T16
**Reuses**: `src/domain/draft.ts`
**Requirement**: TX-03, TX-04, TX-05, TX-06, TX-23

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] Confirm of an expense replies `Despesa salva.` and confirm of an income replies `Receita salva.`
- [ ] A second confirm replies `Esse lançamento já foi salvo.` and the ledger length stays 1
- [ ] Cancel replies `Lançamento cancelado.` and stores no ledger row
- [ ] Confirm or cancel of a cancelled draft, a missing id, or another user's draft replies `Esse lançamento foi cancelado.` and writes nothing
- [ ] The account balance after confirm equals income minus expense from transactions, and the account record has no balance field
- [ ] Gate check passes: `npm test`
- [ ] Test count: 6 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(app): confirm or cancel a draft once`

---

### T18: Answer saldo, histórico, and the four questions

**What**: Implement `executeQuery` for the history story and the four questions, using `occurredOn` for the month and `confirmedAt` for history order and the largest-expense tie.
**Where**: `src/application/queries.ts`
**Depends on**: T17
**Reuses**: `src/domain/balance.ts`, `src/domain/calendar-date.ts`, `src/domain/money.ts`
**Requirement**: QRY-01, QRY-02, QRY-03, QRY-04, QRY-05, QRY-06, QRY-07, QRY-08, QRY-09, QRY-10, QRY-11, QRY-12, QRY-13, QRY-14

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] `Saldo` matches the Nubank `-R$ 42,00` / Inter `R$ 3.500,00` / `Total: R$ 3.458,00` reply, and the empty-account reply
- [ ] `Histórico` returns at most 10 confirmed rows, newest `confirmedAt` first, with the U+2014 separator, and omits pending and cancelled drafts
- [ ] Each question reply matches the spec, including zero spend, no accounts, no expenses, category stored spelling, unknown category, and income ignored
- [ ] Two expenses with the same amount return the one with the later `confirmedAt`
- [ ] Gate check passes: `npm test`
- [ ] Test count: 14 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(app): answer balance, history, and the four questions`

---

### T19: Route one Telegram update

**What**: Implement `handleUpdate`. Persist the update id inside the transaction, route text and callbacks, leave the pending draft in place for non-draft commands, and call `Notifier.send` only after commit.
**Where**: `src/application/handle-update.ts`
**Depends on**: T18
**Reuses**: the application command modules and `src/domain/parser.ts`
**Requirement**: USER-03, USER-04, USER-05, USER-06, TX-14, TX-24, TX-26, QRY-15

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] The same update id applied twice inserts one user and one ledger row
- [ ] A missing telegram user id inserts no user and sends nothing
- [ ] The audit spy receives the update id and the intent, and the recorded line does not contain the raw text or the amount digits
- [ ] `Saldo` while a draft is pending answers the balance and leaves the draft pending
- [ ] Text `Confirmar` gets the unparsed reply and does not confirm
- [ ] A notifier that throws after confirm leaves the ledger row in place (TX-24)
- [ ] `valor R$ 10,00` with no pending draft gets the unparsed reply
- [ ] Gate check passes: `npm test`
- [ ] Test count: 7 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(app): route a Telegram update once`

---

### Phase 6: Persistence

### T20: Write the PostgreSQL schema

**What**: Add `schema.sql` for the six tables and the indexes in the design. No `balance` column. `synchronize` is not enabled.
**Where**: `src/infrastructure/persistence/schema.sql`
**Depends on**: T19
**Reuses**: the column list in `design.md`
**Requirement**: TX-23

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] The SQL creates users, accounts, categories, drafts, transactions, and processed_updates with the unique and check constraints from the design
- [ ] Gate check passes: `npm run lint && npm run build && npm test && npm run test:integration`
- [ ] Test count: unchanged (no new test file)

**Tests**: none
**Gate**: build

**Commit**: `feat(db): add the ledger schema`

---

### T21: Implement the TypeORM ledger repository

**What**: Implement `UnitOfWork` and `Repositories` on a TypeORM 1.1 `DataSource`. Apply `schema.sql` in the integration setup. Import the container helper exported from `test/postgres.integration.spec.ts`. Map a unique-violation on account or category name to the duplicate result the application already turns into the spec reply.
**Where**: `src/infrastructure/persistence/ledger.repository.ts`
**Depends on**: T20
**Reuses**: `src/application/ports.ts`, `test/postgres.integration.spec.ts` container helper (extract the helper into this task if it is still inline)
**Requirement**: USER-04, TX-04, TX-23

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] An account inserted for user B is absent from user A's list
- [ ] A second account named `nubank` for the same user fails the unique index
- [ ] A second pending draft for the same user fails the partial unique index
- [ ] Two confirms of the same draft id cannot insert two transactions
- [ ] `accounts` has no balance column; the balance query returns income minus expense
- [ ] A second `tryBegin` for the same update id returns false
- [ ] Gate check passes: `npm test && npm run test:integration`
- [ ] Test count: 6 integration tests pass in this file, plus the earlier tests still pass

**Tests**: integration
**Gate**: full

**Commit**: `feat(db): persist the ledger in PostgreSQL`

---

### Phase 7: Telegram wiring

### T22: Map replies to grammY keyboards

**What**: Implement `toReplyOptions`. `buttons: true` adds Confirmar, Editar, and Cancelar with callback data `confirm:<uuid>`, `edit:<uuid>`, and `cancel:<uuid>` for the draft id the caller passes.
**Where**: `src/infrastructure/telegram/outgoing.ts`
**Depends on**: T21
**Reuses**: `OutgoingMessage` from `src/application/ports.ts`
**Requirement**: TX-01, TX-26

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] A card message carries the three labels in that order and the three callback payloads
- [ ] A message with `buttons` omitted has no reply markup
- [ ] Gate check passes: `npm test`
- [ ] Test count: 2 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(telegram): render draft buttons`

---

### T23: Dispatch a grammY update into handleUpdate

**What**: Implement `dispatchUpdate` against a `TelegramTransport` interface. Map text and callback queries to `IncomingUpdate`. After `handleUpdate` resolves, send the reply. If send throws, record `reply_failed` without the error text. Install `bot.catch` so a polling error does not use grammY's default stop-the-process handler.
**Where**: `src/infrastructure/telegram/receive-update.ts`
**Depends on**: T22
**Reuses**: `src/application/handle-update.ts`, `src/infrastructure/telegram/outgoing.ts`
**Requirement**: USER-05, TX-24

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] A text update and a `confirm:<uuid>` callback call `handleUpdate` with the mapped input
- [ ] A transport rejection whose message contains `R$ 42,00` does not appear in the audit sink; the sink does contain `reply_failed` and the update id
- [ ] Gate check passes: `npm test`
- [ ] Test count: 3 tests pass in this file, plus the earlier tests still pass

**Tests**: unit
**Gate**: quick

**Commit**: `feat(telegram): dispatch updates without logging the message`

---

### T24: Wire the module and boot against Postgres

**What**: Register the DataSource, repository, clock, and polling provider on the root module. Start polling from `OnApplicationBootstrap` via `bot.start()` and stop it on shutdown. The integration test uses Testcontainers and a fake transport, not the live Bot API.
**Where**: `src/app.module.ts`
**Depends on**: T23
**Reuses**: `src/infrastructure/persistence/ledger.repository.ts`, `src/infrastructure/telegram/receive-update.ts`
**Requirement**: USER-01, TX-03, QRY-06

**Tools**:

- MCP: NONE
- Skill: tlc-spec-driven

**Done when**:

- [ ] Missing `BOT_TOKEN` or `DATABASE_URL` refuses startup and the log line does not contain the secret
- [ ] The integration test creates a user, account Nubank, and a confirmed 4200-centavo Uber expense, then reads that expense back
- [ ] `Quanto gastei este mês` through `handleUpdate` replies `Você gastou R$ 42,00 este mês.`
- [ ] Gate check passes: `npm run lint && npm run build && npm test && npm run test:integration`
- [ ] Test count: 2 integration tests pass in `src/app.module.integration.spec.ts`, plus the earlier tests still pass

**Tests**: integration
**Gate**: full

**Commit**: `feat(app): wire polling and PostgreSQL`

---

## Phase Execution Map

```
Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6 → Phase 7

Phase 1: T1 → T2
Phase 2: T3 → T4 → T5 → T6 → T7 → T8
Phase 3: T9
Phase 4: T10 → T11 → T12 → T13 → T14
Phase 5: T15 → T16 → T17 → T18 → T19
Phase 6: T20 → T21
Phase 7: T22 → T23 → T24
```

Execution is strictly sequential. There is no intra-phase parallelism.

**How phase-based execution works:**

At Execute, the agent counts total tasks and packs phases into task-budgeted batches (about 7 tasks per worker, whole phases). A phase is never split across workers. This feature is 24 tasks, so the packing is:

| Batch | Phases | Tasks |
| --- | --- | --- |
| 1 | Phase 1 and Phase 2 | T1–T8 (8) |
| 2 | Phase 3 and Phase 4 | T9–T14 (6) |
| 3 | Phase 5 | T15–T19 (5) |
| 4 | Phase 6 and Phase 7 | T20–T24 (5) |

When packing yields more than one batch, the agent offers batch sub-agents and waits for a yes before dispatching any. Batches run one after another. After T24 is committed, a fresh Verifier runs automatically.

---

## Task Granularity Check

| Task | Scope | Status |
| ---- | ----- | ------ |
| T1: Scaffold the NestJS 12 process | 1 bootstrap, primary file `src/main.ts` | ✅ Granular |
| T2: Start PostgreSQL from the integration suite | 1 test file | ✅ Granular |
| T3: Parse and format centavos | 1 module | ✅ Granular |
| T4: Resolve São Paulo calendar dates | 1 module | ✅ Granular |
| T5: Suggest at most one category | 1 function | ✅ Granular |
| T6: Compute balances from ledger rows | 1 module | ✅ Granular |
| T7: Guard draft status transitions | 1 module | ✅ Granular |
| T8: Keep a single default account | 1 module | ✅ Granular |
| T9: Parse the fixed Portuguese phrase set | 1 module | ✅ Granular |
| T10: Declare application ports | 1 type module | ✅ Granular |
| T11: Implement the in-memory ledger | 1 module | ✅ Granular |
| T12: Resolve a Telegram user and seed categories | 1 module | ✅ Granular |
| T13: Execute account commands | 1 module | ✅ Granular |
| T14: Execute category commands | 1 module | ✅ Granular |
| T15: Open an expense or income draft | 1 module | ✅ Granular |
| T16: Edit a pending draft | 1 module | ✅ Granular |
| T17: Confirm or cancel a draft | 1 module | ✅ Granular |
| T18: Answer saldo, histórico, and the four questions | 1 module | ✅ Granular |
| T19: Route one Telegram update | 1 module | ✅ Granular |
| T20: Write the PostgreSQL schema | 1 SQL file | ✅ Granular |
| T21: Implement the TypeORM ledger repository | 1 module | ✅ Granular |
| T22: Map replies to grammY keyboards | 1 module | ✅ Granular |
| T23: Dispatch a grammY update into handleUpdate | 1 module | ✅ Granular |
| T24: Wire the module and boot against Postgres | 1 module | ✅ Granular |

**Granularity check**: each task is one module or one file. T1 also writes `package.json`, TypeScript config, and ESLint config because a Nest process cannot compile from `src/main.ts` alone. Those files are the bootstrap, not a second feature.

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| ---- | ---------------------- | ------------- | ------ |
| T1 | None | no incoming arrow | ✅ Match |
| T2 | T1 | T1 → T2 | ✅ Match |
| T3 | T2 | cross-phase, no intra-phase arrow | ✅ Match |
| T4 | T3 | T3 → T4 | ✅ Match |
| T5 | T4 | T4 → T5 | ✅ Match |
| T6 | T5 | T5 → T6 | ✅ Match |
| T7 | T6 | T6 → T7 | ✅ Match |
| T8 | T7 | T7 → T8 | ✅ Match |
| T9 | T8 | cross-phase, no intra-phase arrow | ✅ Match |
| T10 | T9 | cross-phase, no intra-phase arrow | ✅ Match |
| T11 | T10 | T10 → T11 | ✅ Match |
| T12 | T11 | T11 → T12 | ✅ Match |
| T13 | T12 | T12 → T13 | ✅ Match |
| T14 | T13 | T13 → T14 | ✅ Match |
| T15 | T14 | cross-phase, no intra-phase arrow | ✅ Match |
| T16 | T15 | T15 → T16 | ✅ Match |
| T17 | T16 | T16 → T17 | ✅ Match |
| T18 | T17 | T17 → T18 | ✅ Match |
| T19 | T18 | T18 → T19 | ✅ Match |
| T20 | T19 | cross-phase, no intra-phase arrow | ✅ Match |
| T21 | T20 | T20 → T21 | ✅ Match |
| T22 | T21 | cross-phase, no intra-phase arrow | ✅ Match |
| T23 | T22 | T22 → T23 | ✅ Match |
| T24 | T23 | T23 → T24 | ✅ Match |

---

## Test Co-location Validation

| Task | Code Layer Created/Modified | Matrix Requires | Task Says | Status |
| ---- | --------------------------- | --------------- | --------- | ------ |
| T1: Scaffold the NestJS 12 process | Process bootstrap | unit | unit | ✅ OK |
| T2: Start PostgreSQL from the integration suite | Test harness | integration | integration | ✅ OK |
| T3: Parse and format centavos | Domain | unit | unit | ✅ OK |
| T4: Resolve São Paulo calendar dates | Domain | unit | unit | ✅ OK |
| T5: Suggest at most one category | Domain | unit | unit | ✅ OK |
| T6: Compute balances from ledger rows | Domain | unit | unit | ✅ OK |
| T7: Guard draft status transitions | Domain | unit | unit | ✅ OK |
| T8: Keep a single default account | Domain | unit | unit | ✅ OK |
| T9: Parse the fixed Portuguese phrase set | Domain | unit | unit | ✅ OK |
| T10: Declare application ports | Application ports | none | none | ✅ OK |
| T11: Implement the in-memory ledger | Application | unit | unit | ✅ OK |
| T12: Resolve a Telegram user and seed categories | Application | unit | unit | ✅ OK |
| T13: Execute account commands | Application | unit | unit | ✅ OK |
| T14: Execute category commands | Application | unit | unit | ✅ OK |
| T15: Open an expense or income draft | Application | unit | unit | ✅ OK |
| T16: Edit a pending draft | Application | unit | unit | ✅ OK |
| T17: Confirm or cancel a draft | Application | unit | unit | ✅ OK |
| T18: Answer saldo, histórico, and the four questions | Application | unit | unit | ✅ OK |
| T19: Route one Telegram update | Application | unit | unit | ✅ OK |
| T20: Write the PostgreSQL schema | Persistence schema | none | none | ✅ OK |
| T21: Implement the TypeORM ledger repository | Persistence repository | integration | integration | ✅ OK |
| T22: Map replies to grammY keyboards | Telegram adapter | unit | unit | ✅ OK |
| T23: Dispatch a grammY update into handleUpdate | Telegram adapter | unit | unit | ✅ OK |
| T24: Wire the module and boot against Postgres | Nest module wiring | integration | integration | ✅ OK |

---

## Requirement Map

| Requirement IDs | Task |
| --- | --- |
| USER-01, USER-02, CAT-01 | T12 |
| USER-03, USER-04, USER-05, USER-06 | T19 |
| USER-04 (SQL) | T21 |
| ACCT-01 through ACCT-15 | T13 |
| ACCT-02, ACCT-03, ACCT-11, ACCT-12, ACCT-15 (domain rules) | T8 |
| CAT-02, CAT-03, CAT-04, CAT-05 | T14 |
| TX-01, TX-02, TX-13, TX-15, TX-16, TX-17, TX-18, TX-19, TX-20, TX-21, TX-22, TX-25 | T15 |
| TX-03, TX-04, TX-05, TX-06 | T17 |
| TX-03, TX-04, TX-05, TX-06, TX-25 (domain rules) | T7 |
| TX-07, TX-08, TX-09, TX-10, TX-11, TX-12 | T16 |
| TX-14, TX-24, TX-26 | T19 |
| TX-16 (amount lexeme) | T3 |
| TX-17 (civil date) | T4 |
| TX-18, TX-19, TX-20 | T5 |
| TX-23 | T6, T17, T20, T21 |
| TX-26 (phrase set) | T9 |
| QRY-01 through QRY-14 | T18 |
| QRY-15 | T9, T19 |
