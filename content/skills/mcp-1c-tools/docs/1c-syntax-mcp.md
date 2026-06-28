# 1c-syntax-mcp — tool catalog

1C platform syntax reference built from the local `shcntx_ru.hbk` file by `Starik2005/1c-syntax-mcp`.

> Load this file only if the `1c-syntax` server is actually available in the current session.

| Tool | Parameters | When to use |
|---|---|---|
| **search_syntax** | `query` (string, required), `limit` (number, optional, default `10`) | Search 1C functions, methods, or objects by Russian or English name. Use when the exact full name is not known, but a name fragment is available. |
| **get_function_info** | `name` (string, required) | Get detailed information for an exact function or method name (`"СтрДлина"`, `"StrLen"`, `"ТаблицаЗначений"`, `"Массив.Найти"`). |
| **suggest_completion** | `prefix` (string, required), `limit` (number, optional, default `10`) | Get completion candidates by a partial function name or prefix. |
| **validate_syntax** | `code` (string, required) | Validate a single function-call expression, for example `СтрДлина("текст")`. This is not a full `.bsl` module validator; use `analyze_file` for BSL files. |

## Notes

- Prefer `get_function_info` when the exact function or method name is known.
- Use `search_syntax` or `suggest_completion` when only a partial name is known.
- For broader platform articles, methodology, or version comparison, use `onec-buddy-mcp` documentation tools (`search_1c_documentation`, `diff_1c_documentation_versions`) or ITS (`search_its` -> `fetch_its`) when exposed.
