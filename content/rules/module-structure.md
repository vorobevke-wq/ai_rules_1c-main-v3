---
description: Canonical region templates for common modules, object / manager modules, and form modules. Single source of truth for `#Область` names, preprocessor directives, and rules for including regions. Load before creating a new module or restructuring an existing one.
alwaysApply: false
category: development
---

# Module Structure Templates

Canonical region names are **Russian, БСП-style**. English names (`Public` / `Internal` / `Private`) MUST NOT be used — except when the whole configuration is already maintained in English locale with regions defined uniformly across the codebase.

Headlines and anchors for module-region policy — `coding-standards.md → Module Regions` (anchor from `AGENTS.md → Coding Standards`).

## Common Module

```bsl
#Область ПрограммныйИнтерфейс

#КонецОбласти

#Область СлужебныйПрограммныйИнтерфейс

#КонецОбласти

#Область СлужебныеПроцедурыИФункции

#КонецОбласти
```

- `ПрограммныйИнтерфейс` — public exported procedures and functions; what other modules of the configuration may call.
- `СлужебныйПрограммныйИнтерфейс` — exported procedures and functions intended only for the project's own modules (БСП convention: do not call from third-party code).
- `СлужебныеПроцедурыИФункции` — internal helpers, never `Экспорт`.

## Object / Manager Module

```bsl
#Если Сервер Или ТолстыйКлиентОбычноеПриложение Или ВнешнееСоединение Тогда

#Область ПрограммныйИнтерфейс

#КонецОбласти

#Область ОбработчикиСобытий

#КонецОбласти

#Область СлужебныеПроцедурыИФункции

#КонецОбласти

#КонецЕсли
```

The outer `#Если Сервер Или ТолстыйКлиентОбычноеПриложение Или ВнешнееСоединение Тогда` preprocessor block is **mandatory** for object and manager modules.

## Form Module

```bsl
#Область ОбработчикиСобытийФормы

#КонецОбласти

#Область ОбработчикиСобытийЭлементовШапкиФормы

#КонецОбласти

#Область ОбработчикиСобытийЭлементовТаблицыФормыИмяТаблицы

#КонецОбласти

#Область ОбработчикиКомандФормы

#КонецОбласти

#Область СлужебныеПроцедурыИФункции

#КонецОбласти
```

The template above defines canonical region names and their relative order;
it is not mandatory boilerplate.

Create only regions containing actual code. Do not generate empty regions
for completeness or possible future use.

Create `ОбработчикиСобытийЭлементовТаблицыФормы<ИмяТаблицы>` only when
the corresponding form table exists and has handlers. Replace `<ИмяТаблицы>`
with the actual table name; never generate a placeholder region.

## Global Rules

- Regions inside procedures / functions are **forbidden**.
- Pseudo-regions via comments are **forbidden** — use only `#Область` / `#КонецОбласти`.
- Use canonical region names and preserve their prescribed relative order.
  Unused regions may be omitted.
- When creating modules or adding regions, do not generate empty scaffolding.
- Remove regions made empty by the current change. Do not remove unrelated
  pre-existing empty regions unless cleanup is explicitly in scope.
