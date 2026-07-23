# 1C Query Optimization Skill (Advanced Patterns)

Entry point for every non-trivial query task — `content/rules/query-design.md`.

For basic query rules (formatting, aliases, parameters, no queries in loops) — see `dev-standards-architecture.md §3 → "Queries"`.
For anti-patterns with examples (query in loop, subquery in SELECT, virtual table filter in WHERE, missing TOP N) — see the `anti-patterns` rule.

## When to Use

Invoke this skill when:
- Working with complex multi-step data processing
- Optimizing joins and subqueries
- Implementing DCS reports
- Processing large datasets in portions

## Mandatory Optimization Checklist

Walk this list explicitly for **every** query-optimization task and every new multi-batch query:

1. Virtual-table filters are in **parameters**, not `ГДЕ` (`anti-patterns.md §4`).
2. Virtual-table **periodicity matches the join granularity** — joining by `Регистратор` requires periodicity `Регистратор`, not `Авто`.
3. Every temp table later used in a `СОЕДИНЕНИЕ`, `ОБЪЕДИНИТЬ`, or `В (ВЫБРАТЬ …)` filter is created with `ИНДЕКСИРОВАТЬ ПО` on the join or deduplication keys.
4. No redundant deduplication — no `РАЗЛИЧНЫЕ` inside `ОБЪЕДИНИТЬ` operands or on top of `СГРУППИРОВАТЬ ПО`.
5. Correlated or per-row subqueries are replaced with a pre-collected indexed temp table + join.
6. A heavy join feeding `СГРУППИРОВАТЬ ПО` is narrowed and joined through an indexed temp table first.
7. Field lists are minimal — temp tables carry only join keys and fields consumed downstream.
8. Composite references are dereferenced through `ВЫРАЗИТЬ`; display-only fields use `ПРЕДСТАВЛЕНИЕ`.

## Temporary Tables

Use temporary tables for:
- Complex multi-step data processing
- Joining data from multiple sources
- Reusing intermediate results

### Temporary Table Indexing (`ИНДЕКСИРОВАТЬ ПО`) — Mandatory Cases

A temp table has no indexes by default. `ИНДЕКСИРОВАТЬ ПО` is mandatory when the temp table:

1. **Participates in a `СОЕДИНЕНИЕ`** — index the join-condition fields.
2. **Participates in an `ОБЪЕДИНИТЬ`** without `ВСЕ` — index the deduplication keys.
3. **Feeds a `В (ВЫБРАТЬ …)` filter** over a large set — index the filtered field.

Pick the **2–3 most selective fields**; do not enumerate every column. When a join spans many fields, indexing every field can cost more to build than it saves.

```bsl
// ❌ SLOW: temp table is joined later, but has no index
"ВЫБРАТЬ РАЗЛИЧНЫЕ
|	Товары.Номенклатура КАК Номенклатура,
|	Товары.Склад КАК Склад,
|	Товары.Заказ КАК Заказ
|ПОМЕСТИТЬ ВТ_ДвиженияПриЗаписи
|ИЗ
|	&ТаблицаТовары КАК Товары
|ГДЕ
|	Товары.Активность"

// ✅ OPTIMIZED: indexed by selective join/deduplication keys
"ВЫБРАТЬ РАЗЛИЧНЫЕ
|	Товары.Номенклатура КАК Номенклатура,
|	Товары.Склад КАК Склад,
|	Товары.Заказ КАК Заказ
|ПОМЕСТИТЬ ВТ_ДвиженияПриЗаписи
|ИЗ
|	&ТаблицаТовары КАК Товары
|ГДЕ
|	Товары.Активность
|ИНДЕКСИРОВАТЬ ПО
|	Номенклатура, Склад"
```

### Pre-collect and Index Before Join / Group

Two related patterns replace per-row work with one indexed pass:

**A. Correlated subquery → indexed temp table + join.** A `ГДЕ ИСТИНА В (ВЫБРАТЬ ПЕРВЫЕ 1 …)` or another subquery referencing outer-query fields executes for every source row. If the inner data set does not depend on the current outer row, collect it once, index it, and join.

```bsl
// ❌ SLOW: semi-join subquery is executed for every register row
"ВЫБРАТЬ РАЗЛИЧНЫЕ
|	Значения.ТипЗначений КАК ТипЗначений
|ПОМЕСТИТЬ ВТ_Значения
|ИЗ
|	РегистрСведений.ЗначенияПоУмолчанию КАК Значения
|ГДЕ
|	ИСТИНА В (ВЫБРАТЬ ПЕРВЫЕ 1 ИСТИНА
|		ИЗ Справочник.ГруппыДоступа.Пользователи КАК ГруппыПользователи
|		ГДЕ ГруппыПользователи.Ссылка = Значения.ГруппаДоступа
|			И ГруппыПользователи.Пользователь = &Пользователь)"

// ✅ OPTIMIZED: collect the independent set once, index, and join
"ВЫБРАТЬ РАЗЛИЧНЫЕ
|	ГруппыПользователи.Ссылка КАК ГруппаДоступа
|ПОМЕСТИТЬ ВТ_ГруппыПользователя
|ИЗ
|	Справочник.ГруппыДоступа.Пользователи КАК ГруппыПользователи
|ГДЕ
|	ГруппыПользователи.Пользователь = &Пользователь
|ИНДЕКСИРОВАТЬ ПО
|	ГруппаДоступа
|;
|ВЫБРАТЬ РАЗЛИЧНЫЕ
|	Значения.ТипЗначений КАК ТипЗначений
|ПОМЕСТИТЬ ВТ_Значения
|ИЗ
|	РегистрСведений.ЗначенияПоУмолчанию КАК Значения
|	ВНУТРЕННЕЕ СОЕДИНЕНИЕ ВТ_ГруппыПользователя КАК Группы
|	ПО Группы.ГруппаДоступа = Значения.ГруппаДоступа"
```

**B. Narrow keys → virtual table with parameters → join → group.** When a virtual table (`Обороты` or `Остатки`) is joined with a data set and then grouped, build a small key temp table first (`РАЗЛИЧНЫЕ` + `ИНДЕКСИРОВАТЬ ПО`), push selective filters into the virtual-table parameters through `В (ВЫБРАТЬ … ИЗ ВТ_Ключи)`, and only then join and group. Set virtual-table periodicity to match the join: joining by `Регистратор` requires explicit periodicity `Регистратор`.

```bsl
"ВЫБРАТЬ РАЗЛИЧНЫЕ
|	Движения.Номенклатура КАК Номенклатура,
|	Движения.Ячейка КАК Ячейка,
|	Движения.Регистратор КАК Регистратор
|ПОМЕСТИТЬ ВТ_Ключи
|ИЗ
|	ВТ_ДвиженияПоНазначению КАК Движения
|ИНДЕКСИРОВАТЬ ПО
|	Номенклатура, Ячейка, Регистратор
|;
|ВЫБРАТЬ
|	Обороты.Номенклатура КАК Номенклатура,
|	СУММА(Обороты.КоличествоОборот) КАК КоличествоОборот
|ИЗ
|	РегистрНакопления.ТоварыВЯчейках.Обороты(
|		, , Регистратор,
|		Номенклатура В (ВЫБРАТЬ Ключи.Номенклатура ИЗ ВТ_Ключи КАК Ключи)
|			И Ячейка В (ВЫБРАТЬ Ключи.Ячейка ИЗ ВТ_Ключи КАК Ключи)) КАК Обороты
|	ВНУТРЕННЕЕ СОЕДИНЕНИЕ ВТ_Ключи КАК Ключи
|	ПО Обороты.Номенклатура = Ключи.Номенклатура
|		И Обороты.Ячейка = Ключи.Ячейка
|		И Обороты.Регистратор = Ключи.Регистратор
|СГРУППИРОВАТЬ ПО
|	Обороты.Номенклатура"
```

The virtual-table parameter filter should use only the selective key fields; the join condition applies the full key. If a field is not required by the business logic, remove it from both the join and periodicity.

### Join vs Subquery

```bsl
// ✅ PREFERRED: Join (usually faster)
"ВЫБРАТЬ
|	Заказы.Ссылка КАК Заказ,
|	Контрагенты.ИНН КАК ИНН
|ИЗ
|	Документ.ЗаказКлиента КАК Заказы
|		ЛЕВОЕ СОЕДИНЕНИЕ Справочник.Контрагенты КАК Контрагенты
|		ПО Заказы.Контрагент = Контрагенты.Ссылка"

// ⚠️ AVOID: Subquery in SELECT (N+1 problem)
"ВЫБРАТЬ
|	Заказы.Ссылка КАК Заказ,
|	(ВЫБРАТЬ К.ИНН ИЗ Справочник.Контрагенты КАК К 
|	 ГДЕ К.Ссылка = Заказы.Контрагент) КАК ИНН
|ИЗ
|	Документ.ЗаказКлиента КАК Заказы"
```

### Avoid Aggregation in Subqueries

```bsl
// ❌ SLOW: Subquery with aggregation
"ВЫБРАТЬ
|	Номенклатура.Ссылка,
|	(ВЫБРАТЬ СУММА(Остатки.Количество) ...) КАК Остаток
|ИЗ
|	Справочник.Номенклатура КАК Номенклатура"

// ✅ FAST: Join with pre-aggregated data
"ВЫБРАТЬ
|	Номенклатура.Ссылка КАК Номенклатура,
|	ЕСТЬNULL(Остатки.КоличествоОстаток, 0) КАК Остаток
|ИЗ
|	Справочник.Номенклатура КАК Номенклатура
|		ЛЕВОЕ СОЕДИНЕНИЕ РегистрНакопления.ТоварыНаСкладах.Остатки КАК Остатки
|		ПО Номенклатура.Ссылка = Остатки.Номенклатура"
```

## DCS (Data Composition System) Optimization

### Efficient DCS Queries

1. **Use parameters in query text:**
   ```bsl
   // Pass parameters to virtual table
   РегистрНакопления.Остатки.Остатки(&Период, Склад = &Склад)
   ```

2. **Limit data at source:**
   ```bsl
   // Add conditions in DataSet query, not in DCS settings
   ГДЕ Период >= &НачалоПериода
   ```

3. **Use ЕСТЬNULL for outer joins:**
   ```bsl
   ЕСТЬNULL(Остатки.Количество, 0) КАК Количество
   ```

## Composite Type Dereferencing (ITS Standard)

Avoid dereferencing composite type reference fields directly — the system creates queries for **ALL** possible types.

```bsl
// ❌ SLOW: Dereferences ALL registrar types (can be hundreds)
"ВЫБРАТЬ
|	ТоварыНаСкладах.Регистратор.Дата КАК ДатаДокумента
|ИЗ
|	РегистрНакопления.ТоварыНаСкладах КАК ТоварыНаСкладах"

// ✅ FAST: Use ВЫРАЗИТЬ to specify exact type
"ВЫБРАТЬ
|	ВЫРАЗИТЬ(ТоварыНаСкладах.Регистратор КАК Документ.ПоступлениеТоваровУслуг).Дата КАК ДатаДокумента
|ИЗ
|	РегистрНакопления.ТоварыНаСкладах КАК ТоварыНаСкладах"

// ✅ For multiple known types, use ВЫБОР/КОГДА
"ВЫБРАТЬ
|	ВЫБОР
|		КОГДА ТоварыНаСкладах.Регистратор ССЫЛКА Документ.ПоступлениеТоваровУслуг
|			ТОГДА ВЫРАЗИТЬ(ТоварыНаСкладах.Регистратор КАК Документ.ПоступлениеТоваровУслуг).Дата
|		КОГДА ТоварыНаСкладах.Регистратор ССЫЛКА Документ.РеализацияТоваровУслуг
|			ТОГДА ВЫРАЗИТЬ(ТоварыНаСкладах.Регистратор КАК Документ.РеализацияТоваровУслуг).Дата
|	КОНЕЦ КАК ДатаДокумента
|ИЗ
|	РегистрНакопления.ТоварыНаСкладах КАК ТоварыНаСкладах"
```

## Use ПРЕДСТАВЛЕНИЕ for Display (ITS Standard)

When you only need text representation, use `ПРЕДСТАВЛЕНИЕ()` to avoid extra joins:

```bsl
// ❌ Creates additional subquery for Справочник.Склады
"ВЫБРАТЬ
|	ТоварыНаСкладах.Склад.Наименование
|ИЗ
|	РегистрНакопления.ТоварыНаСкладах КАК ТоварыНаСкладах"

// ✅ Optimal: No extra join
"ВЫБРАТЬ
|	ПРЕДСТАВЛЕНИЕ(ТоварыНаСкладах.Склад) КАК СкладПредставление
|ИЗ
|	РегистрНакопления.ТоварыНаСкладах КАК ТоварыНаСкладах"
```

## Avoid Joins with Subqueries (ITS Standard)

Never use subqueries in JOIN — use temporary tables instead:

```bsl
// ❌ WRONG: Join with subquery
"ВЫБРАТЬ ...
|ИЗ
|	Документ.Заказ КАК Заказы
|		ЛЕВОЕ СОЕДИНЕНИЕ (
|			ВЫБРАТЬ Товары.Заказ, СУММА(Товары.Сумма) КАК Сумма
|			ИЗ Документ.Заказ.Товары КАК Товары
|			СГРУППИРОВАТЬ ПО Товары.Заказ
|		) КАК ИтогиТоваров
|		ПО Заказы.Ссылка = ИтогиТоваров.Заказ"

// ✅ CORRECT: Use temporary table
"ВЫБРАТЬ
|	Товары.Ссылка КАК Заказ,
|	СУММА(Товары.Сумма) КАК Сумма
|ПОМЕСТИТЬ ИтогиТоваров
|ИЗ
|	Документ.Заказ.Товары КАК Товары
|СГРУППИРОВАТЬ ПО
|	Товары.Ссылка
|ИНДЕКСИРОВАТЬ ПО
|	Заказ
|;
|ВЫБРАТЬ ...
|ИЗ
|	Документ.Заказ КАК Заказы
|		ЛЕВОЕ СОЕДИНЕНИЕ ИтогиТоваров КАК ИтогиТоваров
|		ПО Заказы.Ссылка = ИтогиТоваров.Заказ"
```

## Avoid Joins with Virtual Tables (ITS Standard)

Extract virtual table results to temporary table before joining:

```bsl
// ⚠️ May be slow: Direct join with virtual table
"ВЫБРАТЬ ...
|ИЗ
|	Справочник.Номенклатура КАК Номенклатура
|		ЛЕВОЕ СОЕДИНЕНИЕ РегистрНакопления.ТоварыНаСкладах.Остатки(&Дата,) КАК Остатки
|		ПО Номенклатура.Ссылка = Остатки.Номенклатура"

// ✅ BETTER: First extract to temporary table
"ВЫБРАТЬ
|	Остатки.Номенклатура КАК Номенклатура,
|	Остатки.КоличествоОстаток КАК Остаток
|ПОМЕСТИТЬ ВТОстатки
|ИЗ
|	РегистрНакопления.ТоварыНаСкладах.Остатки(&Дата,) КАК Остатки
|ИНДЕКСИРОВАТЬ ПО
|	Номенклатура
|;
|ВЫБРАТЬ ...
|ИЗ
|	Справочник.Номенклатура КАК Номенклатура
|		ЛЕВОЕ СОЕДИНЕНИЕ ВТОстатки КАК Остатки
|		ПО Номенклатура.Ссылка = Остатки.Номенклатура"
```

## Avoid OR in WHERE — Use ОБЪЕДИНИТЬ ВСЕ (ITS Standard)

`OR` in `WHERE` prevents index usage. Split into UNION queries:

```bsl
// ❌ SLOW: OR prevents index usage
"ВЫБРАТЬ
|	Товары.Ссылка
|ИЗ
|	Справочник.Номенклатура КАК Товары
|ГДЕ
|	Товары.Артикул = &Артикул
|	ИЛИ Товары.Код = &Код"

// ✅ FAST: Two indexed queries with UNION
"ВЫБРАТЬ
|	Товары.Ссылка
|ИЗ
|	Справочник.Номенклатура КАК Товары
|ГДЕ
|	Товары.Артикул = &Артикул
|
|ОБЪЕДИНИТЬ ВСЕ
|
|ВЫБРАТЬ
|	Товары.Ссылка
|ИЗ
|	Справочник.Номенклатура КАК Товары
|ГДЕ
|	Товары.Код = &Код"
```

## ОБЪЕДИНИТЬ vs ОБЪЕДИНИТЬ ВСЕ (ITS Standard)

Prefer `ОБЪЕДИНИТЬ ВСЕ` when no duplicate rows expected:

```bsl
// ⚠️ SLOWER: ОБЪЕДИНИТЬ performs additional grouping
"ВЫБРАТЬ ... ИЗ Документ.Приход
|ОБЪЕДИНИТЬ
|ВЫБРАТЬ ... ИЗ Документ.Расход"

// ✅ FASTER: ОБЪЕДИНИТЬ ВСЕ skips grouping
"ВЫБРАТЬ ... ИЗ Документ.Приход
|ОБЪЕДИНИТЬ ВСЕ
|ВЫБРАТЬ ... ИЗ Документ.Расход"
```

### No `РАЗЛИЧНЫЕ` Inside `ОБЪЕДИНИТЬ` Operands

`ОБЪЕДИНИТЬ` without `ВСЕ` already collapses duplicates over the combined result. `РАЗЛИЧНЫЕ` in its operands adds redundant sort/group passes:

```bsl
// ❌ Redundant: three deduplication passes
"ВЫБРАТЬ РАЗЛИЧНЫЕ Поля... ИЗ ВТ_Движения
|ОБЪЕДИНИТЬ
|ВЫБРАТЬ РАЗЛИЧНЫЕ Поля... ИЗ РегистрНакопления.Запасы"

// ✅ One deduplication pass — the union itself
"ВЫБРАТЬ Поля... ИЗ ВТ_Движения
|ОБЪЕДИНИТЬ
|ВЫБРАТЬ Поля... ИЗ РегистрНакопления.Запасы"
```

Keep `РАЗЛИЧНЫЕ` only where it does unique work, for example when first materializing a temp table. The same rule forbids `РАЗЛИЧНЫЕ` on top of `СГРУППИРОВАТЬ ПО` over the same fields.

## Index Alignment (ITS Standard)

Ensure query conditions match available indexes:

**Index requirements:**
1. Index must contain **all fields** from the condition
2. Fields must be at the **beginning** of the index
3. Fields must be **consecutive** (no gaps)

```bsl
// Given index: (Организация, Контрагент, Дата)

// ✅ Index will be used — fields are at the beginning
"ГДЕ Организация = &Орг И Контрагент = &Контр"

// ❌ Index NOT used — skipped first field
"ГДЕ Контрагент = &Контр И Дата = &Дата"

// ⚠️ Partial use — gap in fields
"ГДЕ Организация = &Орг И Дата = &Дата"
```

**Creating additional indexes:**
- Set "Индексировать" = "Индексировать с доп. упорядочиванием" for frequently filtered attributes
- Add `ИНДЕКСИРОВАТЬ ПО` for temporary tables used in joins

---

**Reference**: [ITS Query Optimization Standards](https://its.1c.ru/db/v8std/browse/13/-1/26/28)

**Remember**: Verify metadata attributes exist using `search_metadata` templates or `rlm-tools-bsl` helpers (`get_object_full_structure`, `find_attributes`) before writing queries.
