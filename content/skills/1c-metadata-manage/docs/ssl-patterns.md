# 1C SSL/БСП Subsystems Reference

For basic SSL usage (attribute access, user messages) — see `dev-standards-architecture.md §4 → "Data Access — Reference Attribute Access"` and `dev-standards-core.md §2`.

## When to Use

Invoke this skill when:
- Working with users and access rights
- Working with files and attachments
- Implementing print forms
- Managing background jobs
- Working with object versioning
- Sending emails
- Need common utility functions (arrays, structures, strings)

## Core Principle

**ALWAYS check if SSL has a solution before writing custom code.**

## БСП Skill Workflow

When implementing new functionality:

1. **First, check БСП skills** — use locally installed skill packages whose folder names contain `bsp-`; search/read their docs with keywords describing your need
   - Example: find "фоновое задание прогресс" in installed `bsp-*` skill docs
   - Example: find "копирование структуры" in installed `bsp-*` skill docs

2. **Check existing patterns** — use `search_code` / `rlm-tools-bsl` (`search`, `git_search`) to find how similar tasks are solved in the codebase

3. **Use БСП if available** — it is tested, optimized, and maintained

4. **Only then write custom code** — and document why БСП was not suitable

## Key БСП Modules

- **Пользователи** — users, roles, access rights
- **РаботаСФайлами** — file storage and attachments
- **УправлениеПечатью** — print forms
- **ДлительныеОперации** — background jobs with progress
- **ВерсионированиеОбъектов** — object history
- **РаботаСПочтовымиСообщениями** — email sending
- **ОбщегоНазначения** / **ОбщегоНазначенияКлиентСервер** — common utilities
- **СтроковыеФункцииКлиентСервер** — string functions

---

**Remember**: БСП is your first stop for common functionality. Writing custom code when БСП has a solution is technical debt.
