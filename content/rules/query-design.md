---
description: Entry point for 1C query work — pick the exact companion rules and skill docs for writing, optimizing, and reviewing queries. Load first for any non-trivial query task; load companions only via the routing table below.
alwaysApply: false
category: development
---

# Query Design — Entry Point

This file is the **router** for query work. Load it first, then load only the companion sources selected by the table below — companions are not auto-attached by file pattern.

> **Scope.** This file owns routing and load order. Authoritative hard rules (formatting, aliases, parameters, bans) live in `dev-standards-architecture.md §3 → "Queries"`. Query composition and optimization live in the `1c-metadata-manage` skill docs. Severity catalog and fix templates live in `anti-patterns.md`.

## Routing

| Task | Load |
|---|---|
| Project hard rules (formatting, `КАК`, parameters, no queries in loops, intermediate result variable, virtual-table filters) | `dev-standards-architecture.md §3 → "Queries"` |
| Write a new query from scratch (skeleton, virtual tables, temp tables, joins, totals) | `content/skills/1c-metadata-manage/docs/query-writing.md` |
| Tune an existing query (joins vs. subqueries, temp-table indexing, index alignment, composite-type dereferencing, DCS specifics) | `content/skills/1c-metadata-manage/docs/query-optimization.md` — **mandatory for every "optimize this query" task**; walk its *Mandatory Optimization Checklist* item by item |
| Anti-patterns and severity (query in loop, correlated subquery, virtual-table filter in `ГДЕ`, missing `ПЕРВЫЕ N`, unindexed temp table, redundant `РАЗЛИЧНЫЕ`, batch + temp table) | `anti-patterns.md` (§1, §3a, §4, §5, §5a, §7b, Optimized Patterns → Batch Query with Temp Table) |
| Query inside a DCS / SKD report | `dcs-design.md` + `query-optimization.md` (DCS section) |
| Query against a register being designed or restructured | `registers-design.md` first, then this router |

Each companion is self-contained — load only the ones that match the task. Do not preload the whole set "to be safe".

## Pre-flight (every non-trivial query)

1. **Verify metadata** before the first `ВЫБРАТЬ` — use `1c-mcp-metacode` `search_metadata` template operations or `rlm-tools-bsl` helpers (`get_object_full_structure`, `find_attributes`); use `1c-mcp-toolkit` `get_metadata` only when the live infobase is the required source of truth. Do not invent attribute or tabular-section names.
2. **Find a proven shape** before inventing a new skeleton — `templatesearch`, `search_code`, or `rlm-tools-bsl` (`search`, `search_methods`, `git_search`) according to the fallback chain in `content/skills/mcp-1c-tools/SKILL.md`.
3. **Pick the right source** — catalog, document, information-register slice, or accumulation-register virtual table (`Остатки`, `Обороты`, `ОстаткиИОбороты`). Wrong source is a design defect, not a tuning problem.
4. **Apply hard bans** from `dev-standards-architecture.md §3` — no queries in loops, always parameterize, always use `КАК`, filter virtual tables through parameters rather than `ГДЕ`, and use an intermediate variable for `Запрос.Выполнить()`.
5. **Temp-table / union checklist** — for every multi-batch query before delivery: each temp table later used in a `СОЕДИНЕНИЕ`, `ОБЪЕДИНИТЬ`, or `В (ВЫБРАТЬ …)` has `ИНДЕКСИРОВАТЬ ПО` on its join keys (2–3 most selective fields, not the full list); no `РАЗЛИЧНЫЕ` inside `ОБЪЕДИНИТЬ` operands or on top of `СГРУППИРОВАТЬ ПО`; correlated subqueries are replaced by an indexed temp table + join; virtual-table periodicity matches the join keys. Canon — `dev-standards-architecture.md §3` and `query-optimization.md → Mandatory Optimization Checklist`.

## Load order

```text
query-design.md (this file)
  → dev-standards-architecture.md §3 → "Queries"   # hard rules
  → query-writing.md OR query-optimization.md      # how-to
  → anti-patterns.md                               # review / remediation only
```

## Out of scope

- Metadata XML for registers or documents — `registers-design.md` and the `1c-metadata-manage` skill.
- Form-module data-loading patterns — `forms.md` → `form-module.md`.
- Lock and transaction boundaries around query + write — `locks-and-transactions.md`.
