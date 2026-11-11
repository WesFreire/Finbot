# FinBot MVP Context

**Gathered:** 2026-11-10
**Spec:** `.specs/features/finbot-mvp/spec.md`
**Status:** Ready for design

---

## Feature Boundary

Telegram conversational MVP: identity and isolation, accounts, categories, confirm-before-save income/expense drafts, balances, history, and four fixed Portuguese questions. No LLM, no Open Finance, no edit/delete of confirmed ledger rows.

---

## Implementation Decisions

### Pending draft vs other commands

- While a draft is pending, non-edit / non-confirm / non-cancel messages (`Saldo`, `Histórico`, account and category commands, the four questions, unparsed) are handled normally.
- The pending draft is left unchanged.

### Delete default account

- Deleting the default account is allowed when it is not the only account and has no confirmed transactions.
- The remaining account with the earliest creation time becomes the only default.

### Editar button

- Edit follow-ups (`valor`, `categoria`, `descrição`, `conta`) are accepted anytime while a draft is pending; the user does not need to press Editar first.
- Pressing Editar keeps the draft pending and replies with short instructions for those four phrases.

### Confirm / Cancel channel

- Confirmar, Editar, and Cancelar are Telegram buttons (callbacks) only.
- Expense and income drafts both show the same three buttons.
- Text `Confirmar` / `Cancelar` / `Editar` are not parser phrases; they get the unparsed reply when no draft-edit phrase matches.

### Rename onto an existing account name

- `Renomear conta <nome> para <novo>` rejects when `<novo>` already exists ignoring case.
- Reply matches duplicate create: `Você já tem uma conta com esse nome.`

### Category suggestion

- Whole-word, case-insensitive category matching runs on `<description>` only (not date or `na conta` clauses).
- Description exactly `Uber` (ignoring case) still suggests `Transporte` when that category exists.
- If more than one owned category matches as a whole word in the description, suggest none (`sem categoria`).

### `em DD/MM` dates

- Valid calendar day in the current America/Sao_Paulo year only.
- Future dates in that year are rejected with `Data inválida. Use hoje, ontem ou em DD/MM.`
- Both `D/M` and `DD/MM` are accepted when they form a valid past-or-today date.

### Agent's Discretion

None — every raised gray area was decided by the user.

### Declined / Undiscussed Gray Areas → Assumptions

None from this round.

---

## Specific References

Decisions recorded from the requirements-ambiguity review (options 1-A, 2-B, 3-C, 4-A, 5-A, 6-C+A, 7-B).

---

## Deferred Ideas

None - discussion stayed within feature scope
