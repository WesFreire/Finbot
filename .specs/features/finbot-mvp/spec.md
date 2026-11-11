# FinBot MVP Specification

## Problem Statement

People who track money in chat lose the entry the moment the conversation moves on, and a blank finance app is more friction than they will accept. FinBot lets a Brazilian Telegram user record income and expenses, keep more than one account, and ask a small set of balance and spending questions in Portuguese. The MVP proves that loop with a deterministic parser and a confirmation step before anything is stored.

## Goals

- [ ] A user can create an account, record a confirmed expense from one Portuguese sentence, and see that expense in the balance and in the history.
- [ ] The same user can ask the four defined questions and receive amounts computed only from confirmed transactions.
- [ ] A second Telegram user cannot read or change the first user's accounts, categories, or transactions.

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
| --- | --- |
| Budgets | Needs its own spec after transactions exist |
| Spending insights beyond the four questions | The MVP answers only the question set in this spec |
| Open Finance and bank connection | No aggregator in the MVP |
| Statement import and automatic categorization | Depends on Open Finance and a later categorization spec |
| Reminders and notifications that are not the reply to the current message | No scheduler in this spec |
| Language model or other model provider | Understanding is a fixed phrase set |
| Web or mobile UI | Telegram is the only interface |
| Multi-currency | BRL only |
| Shared accounts or households | One Telegram user owns their own data |
| Deleting or editing a confirmed transaction | Drafts can be cancelled before confirm; the ledger row stays |
| Fuzzy category matching | Suggestion is the Uber alias plus an exact category-name match |

---

## Assumptions & Open Questions

Every ambiguity is resolved or recorded here - nothing is left silently unclear.

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Currency and storage | BRL only, integer centavos | Avoids float error and matches the Brazilian audience | n |
| Timezone for hoje, ontem, este mês, and em DD/MM | America/Sao_Paulo | Target users are in Brazil | n |
| Identity | One Telegram user id maps to one FinBot user; no password | Telegram is the only client | n |
| Category suggestion | Case-insensitive whole-word match on owned category names inside `<description>` only; exact description Uber (ignoring case) also suggests Transporte when that category exists; if more than one category matches, draft category stays empty (`sem categoria`) | Description is the parser field boundary; refusing ties avoids wrong categorization | y |
| Account omitted in the sentence | Use the user's default account | The overview example names Nubank as the account on the draft without the user typing the account | n |
| Category omitted | Draft category is empty, shown as sem categoria, and Confirmar stays available | A missing category should not block the entry | n |
| Edit interaction | Edit phrases work anytime while a draft is pending; Editar button only replies with instructions for those phrases; Confirmar/Editar/Cancelar are Telegram buttons only (same three on expense and income) | Avoids an edit wizard; text aliases are out of the parser contract | y |
| History length | 10 most recent confirmed transactions, newest first | Enough to check the last entries without pagination | n |
| Tie on the largest expense | The confirmed transaction with the greatest amount; if amounts match, the latest confirmedAt wins | The question asks for one expense | n |
| Pending draft lifetime | A user has at most one pending draft; a new expense/income parse replaces it; no timer; other commands and questions leave the draft pending | Keeps confirm idempotency simple without silently cancelling on Saldo/Histórico | y |
| Delete default account | Allowed when not the only account and it has no confirmed transactions; the remaining account with the earliest creation time becomes the only default | Preserves the single-default invariant without blocking delete | y |
| Rename uniqueness | Rename to a name that already exists ignoring case is rejected with the same reply as duplicate create | Create uniqueness must not be bypassed by rename | y |
| `em DD/MM` / `em D/M` | Current America/Sao_Paulo year; today or past only; future dates and impossible days rejected; both padded and unpadded day/month accepted when valid | Future dating would distort monthly totals with no MVP need | y |
| Telegram reply fails after commit | The ledger row stays; a later delivery of the same update does not insert again | The chat is not the source of truth | n |
| Queues, HTTP API, compose | Not required to accept this MVP | Constitution commits to them when a feature needs them | n |
| Rate limiting | None inside FinBot | Telegram is the only ingress | n |
| Data retention | Confirmed rows are kept; the MVP has no delete and no TTL | Deleting ledger rows is a later spec | n |
| Name limits | Account and category names 1–40 characters; description 1–80; stored trimmed | Bounds input without a settings screen | n |
| Amount bounds | Greater than 0 and at most R$ 1.000.000,00 (100000000 centavos) | Rejects zero, negative, and absurd values | n |
| Decimal syntax | Comma is the decimal mark; a dot is a thousands separator; fractional part is absent or exactly two digits; R$ 42.50 is rejected | Matches the accepted forms in the overview | n |
| Command matching | Keyword match is case-insensitive; the name the user typed keeps its casing; uniqueness ignores case | Users will not match the spec's capitalization | n |

**Open questions:** none — all resolved or logged above.

---

## Parser contract

The bot accepts only the phrases in this section. Matching is case-insensitive on keywords. Anything else gets the unparsed reply and changes no data.

Amounts, in centavos:

| Text | Centavos |
| --- | --- |
| R$ 42 | 4200 |
| R$ 42,00 | 4200 |
| R$ 1.234,56 | 123456 |
| R$ 3.500 | 350000 |
| R$ 1.000.000,00 | 100000000 |

Rejected amounts: no `R$` prefix, dot used as the decimal mark (`R$ 42.50`, `42.50`), a fractional part that is not two digits (`R$ 42,5`), zero, negative, or above 100000000 centavos.

Dates, interpreted in America/Sao_Paulo:

- Omitted or `hoje` — today's calendar date.
- `ontem` — yesterday's calendar date.
- `em DD/MM` or `em D/M` — that day in the current calendar year, only if it is today or earlier. An impossible day (31/02) or a future day in the current year is rejected.

Expense shape: `Gastei R$ <amount> no|na|em <description> [hoje|ontem|em DD/MM] [na conta <account>]`

Income shape: `Recebi R$ <amount> de <description> [hoje|ontem|em DD/MM] [na conta <account>]`

`<description>` is the text after the preposition, stopping before a date clause or `na conta`. It is required.

Account commands:

- `Criar conta <nome>`
- `Listar contas`
- `Renomear conta <nome> para <novo>`
- `Definir conta padrão <nome>`
- `Excluir conta <nome>`

Category commands:

- `Criar categoria <nome>`
- `Listar categorias`

Ledger commands:

- `Saldo`
- `Histórico`

Questions (optional trailing `?`):

- `Quanto gastei este mês?`
- `Qual é o saldo total das minhas contas?`
- `Qual foi minha maior despesa?`
- `Quanto gastei com <categoria>?`

Edit follow-ups, accepted only while a draft is pending:

- `valor R$ <amount>`
- `categoria <nome>`
- `descrição <texto>`
- `conta <nome>`

Money is displayed as pt-BR: thousands separator `.`, decimal `,`, two digits, prefix `R$ `. A negative balance uses a leading minus: `-R$ 10,00`.

---

## User Stories

### P1: User identity ⭐ MVP

**User Story**: As a Telegram user, I want FinBot to remember me and hide everyone else's finances so that my data stays mine.

**Why P1**: Every later story looks up this user.

**Acceptance Criteria**:

1. WHEN a message arrives with a Telegram user id that has no FinBot user THEN the system SHALL create one user with that id and seed categories named Transporte and Alimentação.
2. WHEN a later message arrives with the same Telegram user id THEN the system SHALL resolve the same user and SHALL NOT insert a second user or a second pair of seed categories.
3. WHEN the same Telegram update id is delivered twice THEN the system SHALL apply the intent once and SHALL NOT create a second user, account, category, draft, or ledger row.
4. WHEN user A requests a list of accounts, categories, transactions, or a balance THEN the system SHALL return only rows whose owner is user A.
5. The system SHALL log the Telegram update id and the resolved intent name, and SHALL NOT log the raw message text or the amount.
6. IF the Telegram user id is missing THEN the system SHALL ignore the update and SHALL NOT create a user.

**Independent Test**: Send one message from Telegram user 100 and one from user 200. User 100's account list does not contain user 200's account.

---

### P1: Accounts ⭐ MVP

**User Story**: As a user, I want more than one named account and a single default so that a transaction lands on the account I mean.

**Why P1**: Transactions and balances need an account.

**Acceptance Criteria**:

1. WHEN the user sends `Criar conta Nubank` and has no account with that name THEN the system SHALL store an account named Nubank with balance 0 and SHALL reply `Conta Nubank criada.`
2. WHEN that account is the user's first account THEN the system SHALL mark it as the only default account.
3. WHEN the user sends `Criar conta Inter` and already has a default THEN the system SHALL store Inter as not default and SHALL leave the previous default in place.
4. WHEN the user sends `Definir conta padrão Inter` and Inter exists THEN the system SHALL mark Inter as the only default and SHALL reply `Conta padrão: Inter.`
5. WHEN the user sends `Listar contas` and owns Nubank (default, 0 centavos) and Inter (1000 centavos) THEN the system SHALL reply `Contas:` followed by `- Nubank (padrão): R$ 0,00` and `- Inter: R$ 10,00`.
6. WHEN the user sends `Renomear conta Inter para C6` and no account named C6 exists ignoring case THEN the system SHALL persist the name C6 and SHALL reply `Conta renomeada para C6.`
7. IF the user sends `Criar conta Nubank` and a name equal to Nubank ignoring case already exists THEN the system SHALL NOT create a second account and SHALL reply `Você já tem uma conta com esse nome.`
8. IF the user sends `Renomear conta Inter para Nubank` and a name equal to Nubank ignoring case already exists THEN the system SHALL keep both names unchanged and SHALL reply `Você já tem uma conta com esse nome.`
9. IF the user sends `Excluir conta Nubank` and that account has any confirmed transaction THEN the system SHALL keep the account and SHALL reply `Não é possível excluir uma conta que possui lançamentos.`
10. WHEN the user sends `Excluir conta Inter` and Inter has no confirmed transactions and is not the only account THEN the system SHALL delete Inter and SHALL reply `Conta Inter excluída.`
11. WHEN the user deletes the default account and at least one other account remains and the deleted account has no confirmed transactions THEN the system SHALL delete it and SHALL mark the remaining account with the earliest creation time as the only default.
12. IF the user sends `Excluir conta Nubank` and Nubank is the only account THEN the system SHALL keep it and SHALL reply `Mantenha ao menos uma conta.`
13. IF the account name is empty or longer than 40 characters THEN the system SHALL NOT create the account and SHALL reply `Use um nome de conta entre 1 e 40 caracteres.`
14. IF the named account does not exist on rename, set-default, or delete THEN the system SHALL reply `Não encontrei a conta "<nome>".`
15. The system SHALL keep exactly one default account whenever the user has at least one account.

**Independent Test**: Create Nubank, create Inter, set Inter as default, list accounts, and see only Inter tagged `(padrão)`.

---

### P1: Categories ⭐ MVP

**User Story**: As a user, I want categories I can name so that spending questions can filter by them.

**Why P1**: The expense example and the alimentação question depend on categories.

**Acceptance Criteria**:

1. WHEN a user is created THEN the system SHALL seed exactly two categories, Transporte and Alimentação, owned by that user.
2. WHEN the user sends `Criar categoria Lazer` THEN the system SHALL store Lazer and SHALL reply `Categoria Lazer criada.`
3. WHEN the user sends `Listar categorias` after the seed and after creating Lazer THEN the system SHALL reply `Categorias:` followed by lines `- Transporte`, `- Alimentação`, and `- Lazer`.
4. IF the user sends `Criar categoria transporte` and Transporte already exists ignoring case THEN the system SHALL NOT create a duplicate and SHALL reply `Você já tem uma categoria com esse nome.`
5. IF the category name is empty or longer than 40 characters THEN the system SHALL NOT create it and SHALL reply `Use um nome de categoria entre 1 e 40 caracteres.`

**Independent Test**: A new user lists categories and sees Transporte and Alimentação before creating any category by hand.

---

### P1: Register a transaction ⭐ MVP

**User Story**: As a user, I want to type what I spent or received and confirm it before it hits my balance so that a misunderstood sentence is not saved.

**Why P1**: This is the core loop from the project overview.

**Acceptance Criteria**:

1. WHEN the user has default account Nubank and category Transporte and sends `Gastei R$ 42 no Uber hoje` THEN the system SHALL create one pending draft and SHALL NOT insert a ledger row, and SHALL reply with `Identifiquei uma despesa:`, `- Valor: R$ 42,00`, `- Categoria: Transporte`, `- Descrição: Uber`, `- Conta: Nubank`, and buttons labeled Confirmar, Editar, and Cancelar.
2. WHEN the user sends `Recebi R$ 3.500 de salário` with default account Nubank and no category named salário THEN the system SHALL create a pending income draft of 350000 centavos, description salário, category empty, date today in America/Sao_Paulo, account Nubank, and SHALL reply with `Identifiquei uma receita:`, `- Valor: R$ 3.500,00`, `- Categoria: sem categoria`, `- Descrição: salário`, `- Conta: Nubank`, and buttons labeled Confirmar, Editar, and Cancelar.
3. WHEN the user presses Confirmar on a pending draft THEN the system SHALL store one confirmed transaction with the draft's type, amount, category, description, account, and date, SHALL mark the draft confirmed, and SHALL reply `Despesa salva.` or `Receita salva.` matching the type.
4. WHEN Confirmar is delivered again for a draft that is already confirmed THEN the system SHALL NOT insert a second ledger row and SHALL reply `Esse lançamento já foi salvo.`
5. WHEN the user presses Cancelar on a pending draft THEN the system SHALL mark the draft cancelled, SHALL NOT insert a ledger row, and SHALL reply `Lançamento cancelado.`
6. IF the user presses Confirmar or Cancelar on a cancelled draft THEN the system SHALL NOT insert a ledger row and SHALL reply `Esse lançamento foi cancelado.`
7. WHEN the user presses Editar on a pending draft THEN the system SHALL keep the draft pending and SHALL reply `Para editar, envie: valor R$ <valor>, categoria <nome>, descrição <texto> ou conta <nome>.`
8. WHILE a draft is pending, WHEN the user sends `valor R$ 50,00` THEN the system SHALL set the amount to 5000 centavos and SHALL show the draft again with the Confirmar, Editar, and Cancelar buttons.
9. WHILE a draft is pending, WHEN the user sends `categoria Alimentação` and that category exists THEN the system SHALL set the category and SHALL show the draft again.
10. WHILE a draft is pending, WHEN the user sends `descrição Corrida` THEN the system SHALL set the description to Corrida and SHALL show the draft again.
11. WHILE a draft is pending, WHEN the user sends `conta Inter` and Inter exists THEN the system SHALL set the account to Inter and SHALL show the draft again.
12. IF an edit names a category or account the user does not own THEN the system SHALL keep the previous category or account and SHALL reply `Não encontrei a categoria "<nome>".` or `Não encontrei a conta "<nome>".`
13. WHEN a new expense or income sentence arrives while a draft is pending THEN the system SHALL cancel the previous pending draft and SHALL open one new pending draft.
14. WHILE a draft is pending, WHEN the user sends `Saldo`, `Histórico`, an account command, a category command, or one of the four questions THEN the system SHALL handle that message and SHALL leave the pending draft unchanged.
15. IF the user has no account and sends an expense or income sentence THEN the system SHALL NOT create a draft and SHALL reply `Crie uma conta antes. Exemplo: "Criar conta Nubank".`
16. IF the amount is `R$ 0`, `R$ 42.50`, `R$ 42,5`, or above `R$ 1.000.000,00` THEN the system SHALL NOT create a draft and SHALL reply `Valor inválido. Use o formato R$ 42,00.`
17. IF the date is `em 31/02` or a future `em DD/MM` / `em D/M` in the current America/Sao_Paulo year THEN the system SHALL NOT create a draft and SHALL reply `Data inválida. Use hoje, ontem ou em DD/MM.`
18. WHEN the description is Uber ignoring case and the user owns Transporte THEN the system SHALL suggest Transporte.
19. WHEN the description (not the date or account clause) contains exactly one owned category name as a whole word, ignoring case, and the description is not Uber THEN the system SHALL suggest that category.
20. IF the description contains two or more owned category names as whole words, ignoring case, and the description is not Uber THEN the system SHALL leave the draft category empty (`sem categoria`).
21. WHEN the sentence contains `na conta Inter` and Inter exists THEN the system SHALL put the draft on Inter instead of the default account.
22. IF `na conta Inter` names an account the user does not own THEN the system SHALL NOT create a draft and SHALL reply `Não encontrei a conta "Inter".`
23. The system SHALL compute each account balance as confirmed income centavos minus confirmed expense centavos on that account, and SHALL NOT store a separately updated balance.
24. IF sending the Telegram reply fails after Confirmar has committed THEN the system SHALL keep the confirmed transaction.
25. IF the description is empty or longer than 80 characters THEN the system SHALL NOT create a draft and SHALL reply `Use uma descrição entre 1 e 80 caracteres.`
26. Confirmar, Editar, and Cancelar SHALL be offered only as Telegram buttons; text messages `Confirmar`, `Cancelar`, or `Editar` SHALL receive the unparsed reply and SHALL NOT confirm, cancel, or edit a draft.

**Independent Test**: Send `Gastei R$ 42 no Uber hoje`, see the draft and a zero balance, press Confirmar, and see the balance become `-R$ 42,00` on Nubank.

---

### P1: History and balances ⭐ MVP

**User Story**: As a user, I want the recent transactions and the balance of each account so that I can check what was saved.

**Why P1**: A saved transaction the user cannot see is not a usable ledger.

**Acceptance Criteria**:

1. WHEN the user sends `Saldo` and owns Nubank at -4200 centavos and Inter at 350000 centavos THEN the system SHALL reply `Contas:` with `- Nubank (padrão): -R$ 42,00` and `- Inter: R$ 3.500,00`, then `Total: R$ 3.458,00`.
2. WHEN the user sends `Saldo` and owns no account THEN the system SHALL reply `Você ainda não tem contas. Exemplo: "Criar conta Nubank".`
3. WHEN the user sends `Histórico` and has confirmed transactions THEN the system SHALL reply with the 10 most recent, newest first, each line `DD/MM/AAAA — Despesa|Receita — <descrição> — <categoria ou sem categoria> — <conta> — R$ <valor>`.
4. WHEN the user sends `Histórico` and has no confirmed transactions THEN the system SHALL reply `Você ainda não tem lançamentos.`
5. The system SHALL omit pending and cancelled drafts from `Saldo` and from `Histórico`.

**Independent Test**: Confirm one expense and one income, send `Histórico`, and see both lines. A cancelled draft does not appear.

---

### P1: Answer the four questions ⭐ MVP

**User Story**: As a user, I want to ask what I spent and what I have so that I do not add the numbers myself.

**Why P1**: These are the questions in the project overview.

**Acceptance Criteria**:

1. WHEN the user sends `Quanto gastei este mês?` THEN the system SHALL reply `Você gastou R$ <total> este mês.` where `<total>` is the pt-BR sum of confirmed expenses whose date falls in the current calendar month in America/Sao_Paulo.
2. WHEN that sum is 0 THEN the system SHALL reply `Você gastou R$ 0,00 este mês.`
3. WHEN the user sends `Qual é o saldo total das minhas contas?` THEN the system SHALL reply `Seu saldo total é R$ <total>.` or `Seu saldo total é -R$ <total>.` equal to the sum of account balances.
4. WHEN the user has no accounts THEN the system SHALL reply `Seu saldo total é R$ 0,00.`
5. WHEN the user sends `Qual foi minha maior despesa?` and confirmed expenses exist THEN the system SHALL reply `Sua maior despesa foi R$ <valor> em <descrição> (<categoria ou sem categoria>) no dia DD/MM/AAAA.` for the greatest amount, breaking ties by the latest confirmedAt.
6. WHEN the user has no confirmed expense THEN the system SHALL reply `Você ainda não tem despesas.`
7. WHEN the user sends `Quanto gastei com alimentação?` and a category matches Alimentação ignoring case THEN the system SHALL reply `Você gastou R$ <total> com Alimentação este mês.` using the stored category name and the current month's confirmed expenses in that category.
8. IF no category name matches the text after `com` THEN the system SHALL reply `Não encontrei a categoria "<texto>".` and SHALL NOT treat the sum as zero.
9. The system SHALL ignore income when answering `Quanto gastei este mês?`, `Qual foi minha maior despesa?`, and `Quanto gastei com <categoria>?`.
10. IF the message matches no phrase in the parser contract THEN the system SHALL reply `Não entendi. Exemplos: "Gastei R$ 42 no Uber hoje" ou "Quanto gastei este mês?"` and SHALL NOT change stored data.

**Independent Test**: Confirm a R$ 42,00 Uber expense today and a R$ 10,00 income today. `Quanto gastei este mês?` answers `Você gastou R$ 42,00 este mês.`

---

## Edge Cases

- IF a confirmed transaction exists on an account THEN the system SHALL reject deletion of that account with `Não é possível excluir uma conta que possui lançamentos.`
- IF the amount uses a dot as the decimal mark THEN the system SHALL reject it with `Valor inválido. Use o formato R$ 42,00.`
- IF Confirmar is repeated THEN the system SHALL keep a single ledger row.
- WHEN the largest expenses share an amount THEN the system SHALL return the one with the latest confirmedAt.
- IF the user has no pending draft and sends `valor R$ 10,00` THEN the system SHALL treat the message as unparsed.
- WHILE a draft is pending, WHEN the user sends `Saldo` THEN the system SHALL reply with balances and SHALL keep the draft pending.
- IF `em DD/MM` names a future day in the current year THEN the system SHALL reject it with `Data inválida. Use hoje, ontem ou em DD/MM.`
- IF rename would create a case-insensitive duplicate account name THEN the system SHALL reject it with `Você já tem uma conta com esse nome.`
- IF two category names match in the description THEN the system SHALL leave the draft as `sem categoria`.

---

## Implicit-requirement dimensions

| Dimension | Resolution |
| --- | --- |
| Input validation and bounds | Amount syntax and bounds, name lengths, date validity in the transaction and account stories |
| Failure and partial failure | A failed Telegram reply after commit keeps the ledger row (TX-24) |
| Idempotency and duplicates | Repeated update id and repeated confirm (USER-03, TX-04) |
| Auth boundaries and rate limits | Owner filter on every read (USER-04). Rate limiting is N/A because Telegram is the only ingress and this spec adds no public HTTP API |
| Concurrency and ordering | One pending draft per user; a second expense/income parse replaces it; other commands leave it pending; confirm of a finished draft does not insert again |
| Data lifecycle and expiry | N/A for deletion and TTL: confirmed rows are kept and this spec has no delete of transactions |
| Observability | Log update id and intent; do not log raw text or amount (USER-05) |
| External-dependency failure | Telegram delivery failure does not roll back a committed confirm (TX-24) |
| State-transition integrity | Draft moves pending to confirmed or pending to cancelled; confirm and cancel are refused afterward; Editar does not change draft status |

---

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| --- | --- | --- | --- |
| USER-01 | P1: User identity | Specify | Pending |
| USER-02 | P1: User identity | Specify | Pending |
| USER-03 | P1: User identity | Specify | Pending |
| USER-04 | P1: User identity | Specify | Pending |
| USER-05 | P1: User identity | Specify | Pending |
| USER-06 | P1: User identity | Specify | Pending |
| ACCT-01 | P1: Accounts | Specify | Pending |
| ACCT-02 | P1: Accounts | Specify | Pending |
| ACCT-03 | P1: Accounts | Specify | Pending |
| ACCT-04 | P1: Accounts | Specify | Pending |
| ACCT-05 | P1: Accounts | Specify | Pending |
| ACCT-06 | P1: Accounts | Specify | Pending |
| ACCT-07 | P1: Accounts | Specify | Pending |
| ACCT-08 | P1: Accounts | Specify | Pending |
| ACCT-09 | P1: Accounts | Specify | Pending |
| ACCT-10 | P1: Accounts | Specify | Pending |
| ACCT-11 | P1: Accounts | Specify | Pending |
| ACCT-12 | P1: Accounts | Specify | Pending |
| ACCT-13 | P1: Accounts | Specify | Pending |
| ACCT-14 | P1: Accounts | Specify | Pending |
| ACCT-15 | P1: Accounts | Specify | Pending |
| CAT-01 | P1: Categories | Specify | Pending |
| CAT-02 | P1: Categories | Specify | Pending |
| CAT-03 | P1: Categories | Specify | Pending |
| CAT-04 | P1: Categories | Specify | Pending |
| CAT-05 | P1: Categories | Specify | Pending |
| TX-01 | P1: Register a transaction | Specify | Pending |
| TX-02 | P1: Register a transaction | Specify | Pending |
| TX-03 | P1: Register a transaction | Specify | Pending |
| TX-04 | P1: Register a transaction | Specify | Pending |
| TX-05 | P1: Register a transaction | Specify | Pending |
| TX-06 | P1: Register a transaction | Specify | Pending |
| TX-07 | P1: Register a transaction | Specify | Pending |
| TX-08 | P1: Register a transaction | Specify | Pending |
| TX-09 | P1: Register a transaction | Specify | Pending |
| TX-10 | P1: Register a transaction | Specify | Pending |
| TX-11 | P1: Register a transaction | Specify | Pending |
| TX-12 | P1: Register a transaction | Specify | Pending |
| TX-13 | P1: Register a transaction | Specify | Pending |
| TX-14 | P1: Register a transaction | Specify | Pending |
| TX-15 | P1: Register a transaction | Specify | Pending |
| TX-16 | P1: Register a transaction | Specify | Pending |
| TX-17 | P1: Register a transaction | Specify | Pending |
| TX-18 | P1: Register a transaction | Specify | Pending |
| TX-19 | P1: Register a transaction | Specify | Pending |
| TX-20 | P1: Register a transaction | Specify | Pending |
| TX-21 | P1: Register a transaction | Specify | Pending |
| TX-22 | P1: Register a transaction | Specify | Pending |
| TX-23 | P1: Register a transaction | Specify | Pending |
| TX-24 | P1: Register a transaction | Specify | Pending |
| TX-25 | P1: Register a transaction | Specify | Pending |
| TX-26 | P1: Register a transaction | Specify | Pending |
| QRY-01 | P1: History and balances | Specify | Pending |
| QRY-02 | P1: History and balances | Specify | Pending |
| QRY-03 | P1: History and balances | Specify | Pending |
| QRY-04 | P1: History and balances | Specify | Pending |
| QRY-05 | P1: History and balances | Specify | Pending |
| QRY-06 | P1: Answer the four questions | Specify | Pending |
| QRY-07 | P1: Answer the four questions | Specify | Pending |
| QRY-08 | P1: Answer the four questions | Specify | Pending |
| QRY-09 | P1: Answer the four questions | Specify | Pending |
| QRY-10 | P1: Answer the four questions | Specify | Pending |
| QRY-11 | P1: Answer the four questions | Specify | Pending |
| QRY-12 | P1: Answer the four questions | Specify | Pending |
| QRY-13 | P1: Answer the four questions | Specify | Pending |
| QRY-14 | P1: Answer the four questions | Specify | Pending |
| QRY-15 | P1: Answer the four questions | Specify | Pending |

**Coverage:** 67 total, 0 mapped to tasks, 67 unmapped.

---

## Success Criteria

- [ ] From a new Telegram user, `Criar conta Nubank`, `Gastei R$ 42 no Uber hoje`, and Confirmar persist one expense of 4200 centavos on Nubank in category Transporte.
- [ ] `Quanto gastei este mês?`, `Qual é o saldo total das minhas contas?`, `Qual foi minha maior despesa?`, and `Quanto gastei com alimentação?` return the amounts defined in this spec.
- [ ] A second Telegram user cannot see the first user's account, category, or transaction.
- [ ] Delivering the same Telegram update twice does not duplicate the user or the ledger row.
