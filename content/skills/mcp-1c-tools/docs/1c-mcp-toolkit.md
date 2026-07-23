# 1c-mcp-toolkit - tool catalog

Live infobase access through [ROCTUP/1c-mcp-toolkit](https://github.com/ROCTUP/1c-mcp-toolkit). (https://github.com/ROCTUP/1c-mcp-toolkit/releases/tag/v1.8.0). The server is expected at `http://localhost:6003/mcp` and is started by the user through the `MCP_Toolkit.epf` processing, either in embedded-server mode or through the Python proxy. This ruleset assumes TOON output is enabled in the processing / proxy configuration.

> Load this file only if the `1c-mcp-toolkit` server is actually available in the current session. The catalog entry alone does not count as availability - the Toolkit processing/proxy must be running and the tools must be visible in the agent tool schema.

## Tool catalog

| Tool | Key parameters | Purpose | When to use |
|---|---|---|---|
| **execute_query** | `query` - 1C query text; `params` - optional query parameters; `limit` - row limit, default 100, max 1000; `include_schema` - include column type schema | Executes a 1C query against the connected infobase. Reference values from previous results can be passed back through `params`. | Read-only live-data checks: row counts, sample values, virtual-table behavior, data-state hypotheses. |
| **execute_code** | `code` - BSL fragment; `execution_context` - `server` (default) or `client` | Executes a BSL statement block. Return data by assigning `Результат = ...`. The server blocks or asks approval for dangerous operations depending on its configuration. | Read-only BSL probes: platform behavior, metadata checks, feature options, `ЗначениеЗаполнено`, parse-only query validation. |
| **get_metadata** | `filter`, `meta_type`, `name_mask`, `limit`, `offset`, `sections`, `extension_name`, `attribute_mask` | Reads live metadata structure: summary, filtered list, exact object details, extension objects, attribute search. | Confirm object / attribute shape in the currently running infobase when static source/index data may be stale. Prefer project-index servers for source-code analysis. |
| **get_event_log** | `start_date`, `end_date`, `levels`, `events`, `limit`, `same_second_offset`, `metadata_type`, `user`, `session`, `application`, `computer`, `comment_contains`, `transaction_status`, `link` / `data` / `object_description` | Reads event-log records with filtering and cursor pagination. | Reproduce/debugging phase: exact runtime errors, warnings, affected metadata, user/session/application filters, older or narrower log searches. |
| **get_object_by_link** | `link` - navigation link (`e1cib/data/...`) | Returns object data by navigation link. | Inspect one known reference from UI/log/query output. |
| **get_link_of_object** | `object_description` - object description from Toolkit results | Generates a navigation link for an object description. | Round-trip object references between query results, event-log filters and UI navigation. |
| **find_references_to_object** | object reference / link parameters, optional search scope | Finds references to a live object across supported collections. | Runtime reference-impact checks against current data. For source-level impact analysis prefer `1c-mcp-metacode` or `rlm-tools-bsl`. |
| **get_access_rights** | `metadata_object`; optional `user_name`, `rights_filter`, `roles_filter` | Returns role rights and optional effective rights for a user. RLS and contextual restrictions are not authoritative here. | Quick rights inspection for a metadata object or suspected access issue. |
| **get_bsl_syntax_help** | `keywords`, `match`, `limit`, `offset`, `content_page` | Searches BSL syntax help bundled with Toolkit. | Fallback syntax lookup when `1c-syntax` is unavailable or when Toolkit is already the active live-IB context. Prefer `1c-syntax` for platform syntax reference in this ruleset. |
| **get_screenshot** | `form_name`, `scale_percent`, `show_grid`, `region`, `highlight_rects` | Captures the active 1C window as PNG. Windows-only; may be disabled. | Visual confirmation of a running form when browser/UI automation is not the right tool. |
| **restart_1c_session** | no parameters | Restarts the current 1C session that serves Toolkit. | Only after an explicit user request or a documented deployment/update step that requires a session restart. |
| **close_1c_session** | no parameters | Closes the current 1C session and returns a launcher command for a new one. | Only after explicit user consent, typically for exclusive-access operations. |
| **submit_for_deanonymization** | `text` | Available only when Toolkit anonymization is enabled. Submits the final user-facing answer for de-anonymization. | Call once before the final answer only when the final text contains anonymization tokens like `[ORG-00001]`. |

## Common live-IB operations

| Need | Toolkit call |
|---|---|
| Run a read-only BSL probe | `execute_code(code=<fragment>, execution_context="server")` |
| Run a read-only query | `execute_query(query=<text>, params={...}, limit=..., include_schema=...)` |
| Fetch the latest relevant error | `get_event_log(levels=["Error"], start_date=<recent ISO date>, limit=1)`; add `metadata_type`, `user`, `application`, `comment_contains` when narrowing is useful. |
| Parse-check a query string | No direct tool. Prefer the `execute_code` parse-only wrapper below. Use `execute_query` only when the query can be safely rewritten to return no rows. |

### Parse-checking a query

Preferred pattern:

```bsl
Попытка
    Запрос = Новый Запрос("<query text with doubled quotes>");
    Запрос.НайтиПараметры();
    Результат = "нет ошибок";
Исключение
    Результат = ОписаниеОшибки();
КонецПопытки;
```

Call it through:

```json
{
  "code": "<BSL wrapper above>",
  "execution_context": "server"
}
```

This checks query parsing and parameter discovery through the live platform without reading data. It does not prove that every table/field exists under every runtime context and does not evaluate RLS.

Alternative when parse-only wrapping is impractical: call `execute_query` with a deliberately empty result, for example by wrapping a simple SELECT as `ВЫБРАТЬ * ИЗ (<original query>) КАК T ГДЕ ЛОЖЬ`, or by adding an equivalent domain-safe false condition. Use this only when the rewrite is obviously valid for that query shape.

## Safety and discipline

`execute_code` and `execute_query` run against the connected infobase with the rights of the current Toolkit 1C session.

- **Read-only first.** Default to `execute_query` for data-state checks and to non-mutating `execute_code` fragments for platform/metadata checks.
- **No mutation without explicit user consent.** Before `Записать()`, `Удалить()`, `НачатьТранзакцию()`, DML, session restart, or session close - ask the user, name the affected objects, and have a rollback plan. On production infobases, request a copy/test IB instead.
- **Use TOON-aware reading.** Toolkit may return compact TOON instead of JSON. Treat TOON as structured output and preserve field names exactly when feeding values into the next tool call.
- **Do not pipe AI output blindly.** Output from `ask_1c_ai` or `modify_1c_code` is non-deterministic. Validate BSL through `diagnostics` / `check_1c_code`; validate query strings through the parse-only `execute_code` wrapper before executing them.
- **Prefer static/indexed servers when live data is not required.** For source navigation and impact analysis use `1c-mcp-metacode` and `rlm-tools-bsl`; use Toolkit only when the question is about the running infobase.
