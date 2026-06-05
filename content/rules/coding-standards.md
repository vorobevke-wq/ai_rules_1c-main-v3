---
description: Coding standards — forbidden constructs, comments, code review, module regions, queries, data access, performance (headlines + pointers)
alwaysApply: false
category: development
---

# Coding Standards (headlines)

Authoritative content for code style, naming, comments, queries, data access and performance lives in the detailed on-demand rules: `dev-standards-core.md`, `dev-standards-architecture.md`, `dev-standards-forms.md`, `module-structure.md`, `anti-patterns.md`, `platform-solutions.md`, `locks-and-transactions.md`, `logging-strategy.md`. This file is the index of headlines and anchors. **Before writing or reviewing code, load the relevant detail file.**

## Forbidden Calls and Constructs (project-wide)

Single source of truth — `dev-standards-core.md §2 → "Forbidden Calls and Constructs"` (ternary `?(...)`, `Выполнить()` / `Вычислить()`, hardcoded credentials, `COMОбъект`, `Сообщить()`, `ЗаписьЖурналаРегистрации()` without explicit task, `Попытка ... Исключение` around DB reads/writes, boolean comparison against `Истина` / `Ложь`, Yoda syntax). Naming bans (Hungarian notation, names from the 1C global context, magic numbers, negative boolean names) and the `[Project rule — stricter than ITS standard]` markers also live there.

Do not duplicate the list here — when the rule changes, only `dev-standards-core.md §2 → "Forbidden Calls and Constructs"` is updated.

## Comments

Prefer self-documenting code. Comments are appropriate only when they add value: motivation, non-trivial algorithm, constraints / side effects, technical-debt markers (`TODO No.<task>: ...`), platform hacks. Comments that paraphrase the code or decorate modules with author / history banners are forbidden — git tracks that. Examples and the verification rule — `dev-standards-core.md §7`.

## Code Review After Each Edit

After any code edit, perform an internal review: style, readability, correctness, edge cases, security, concurrency, locks, transactions. Always consider whether an outer transaction already exists (e.g., the object-write transaction) before opening a new one. Loop until clean within the verification budget from `AGENTS.md`; after the budget is exhausted, fix substantive issues and report any remaining style noise. Full guidance — `dev-standards-core.md §8`.

## Code Reuse

Before writing new code — check common and manager modules for an existing export method that can be reused. Use `search_function`, `ssl_search`, `templatesearch`, and `codesearch` **before** writing.

## Module Regions

Canonical region names — Russian, БСП-style. Templates per module type (common module, object / manager module, form module) — `module-structure.md`. Regions inside procedures / functions are forbidden; pseudo-regions via comments are forbidden.

## Queries

Authoritative rules and the formatting template — `dev-standards-architecture.md §3 → "Queries"`. Headlines:

- Verify metadata before writing a query (`metadatasearch` / `get_metadata_details`).
- No queries inside loops — use batch queries with temporary tables (`ВТ_*`).
- Always parameterize (`Запрос.УстановитьПараметр()`), never concatenate strings.
- Always use `КАК` aliases. Use `ПЕРВЫЕ N` when only a subset is needed.
- Filter virtual tables by parameters, not by `ГДЕ`.
- Always use an intermediate variable for the query result (`РезультатЗапроса = Запрос.Выполнить();`); method chaining is forbidden.

## Data Access — Reference Attributes

Do not access reference attributes via dot notation (`Контрагент.ИНН`). Use `ОбщегоНазначения.ЗначениеРеквизитаОбъекта` / `ЗначенияРеквизитовОбъекта` / `ЗначениеРеквизитаОбъектов` / `ЗначенияРеквизитовОбъектов`. **[Project rule — stricter than ITS standard.]** Full method table and caching / batch templates — `dev-standards-architecture.md §4 → "Data Access — Reference Attribute Access"`.

## Performance

Authoritative baseline (server-side bulk, queries, privileged mode, caching, collections, transactions, managed locks) — `dev-standards-architecture.md §5`. Detailed anti-pattern catalog with severity — `anti-patterns.md`. Platform pitfalls (long-running operations, temporary storage, transactions, deadlocks, dates, collection search, external components) — `platform-solutions.md`.

## Project Rules Stricter Than the ITS Standard

Some project rules are intentionally **stricter** than the official 1C ITS standard. Each such rule in this file and in the on-demand rules is tagged with **`[Project rule — stricter than ITS standard]`**. When discussing such a rule with the user or in code review:

- Refer to it as a **project decision**, not as an ITS requirement.
- If asked — explicitly state the delta vs the ITS standard.
- Do not silently weaken these rules "to match ITS"; raise the question and let the user decide.
