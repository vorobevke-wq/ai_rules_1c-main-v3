# rlm-tools-bsl — tool catalog

Token-efficient exploration server for 1C BSL repositories.
Upstream: <https://github.com/Dach-Coin/rlm-tools-bsl>.

`rlm-tools-bsl` does not expose one MCP tool per 1C operation. It exposes a small session API; inside `rlm_execute`, the agent runs Python code that calls sandbox helpers for BSL files, metadata XML, forms, usages, and call graphs.

> Load this file only if the `rlm-tools-bsl` server is actually available in the current session.

## Top-level MCP tools

| Tool | Stable documented input | Purpose | When to use |
|---|---|---|---|
| **rlm_projects** | `action`, optional `name`, `path`, `description`, `new_name`, `password`, `clear_password` | Manage the server-side project registry. Mutating actions require a user-supplied project password when the server returns `approval_required`. | List registered projects; register a frequently used 1C source tree so later calls can use `rlm_start(project="...")`. |
| **rlm_index** | `action`, `project` or `path`; optional `no_calls`, `no_metadata`, `no_fts`, `no_synonyms`, `confirm` | Build/update/drop/show the optional SQLite index. Build/update run in the background for registered projects. | Before large repositories, stale search results, or when helpers requiring index tables return degraded results. |
| **rlm_start** | `query`, `path` or `project`; optional `effort`, `max_output_chars`, `max_llm_calls`, `max_execute_calls`, `execution_timeout_seconds`, `include_metadata` | Open a read-only exploration session and return `session_id`, strategy, limits, index status, and available helper signatures. | First call for repository exploration. If the user names a registered project, prefer `project`; otherwise pass an absolute 1C source path. |
| **rlm_help** | optional `topic`, `helpers`, `category`, `section`, `format`, `include_code` | Slim-mode companion that returns recipes, helper details, disambiguation rules, and workflow sections omitted from `rlm_start`. | Call before `rlm_execute` on non-trivial analysis; call with `helpers=[...]` before using unfamiliar helpers. |
| **rlm_execute** | `session_id`, `code`, optional `detail_level`, `max_new_variables` | Execute Python in the sandbox. Variables persist between calls; only printed output is returned to the agent context. | Batch several related helper calls, filter/summarize server-side, and print compact conclusions. |
| **rlm_end** | `session_id` | Close the exploration session and free resources. | Always call when the RLM exploration is complete. |

## Session discipline

1. Start with `rlm_start(query=..., project=...)` or `rlm_start(query=..., path=...)`.
2. In default slim strategy mode, call `rlm_help(...)` before non-trivial `rlm_execute` calls. Use `topic` for business recipes (`"проведение"`, `"печать"`, `"права"`, `"интеграция"`, `"ссылки"`, `"структура объекта"`), `helpers` for exact helper docs, or `category` for helper lists.
3. Batch operations inside one `rlm_execute`: search candidates, read the top files/procedures, compute a summary, then `print()` only the useful result.
4. Never run broad `grep` over `path='.'` on large 1C configurations. Prefer metadata/code helpers and `git_search` where available.
5. End with `rlm_end(session_id)` unless the parent explicitly needs to continue the same session.

## Sandbox helper map

The complete helper list is returned by `rlm_start.available_functions`; detailed helper docs are available through `rlm_help(helpers=[...])`. Common replacements for the old indexed metadata server:

| Need | RLM helper(s) inside `rlm_execute` |
|---|---|
| Broad search by object / method / attribute / region | `search(query, scope="all", limit=...)`; refine with `search_objects`, `search_methods`, `search_regions`, `search_module_headers`, `find_attributes`, `find_predefined`. |
| Find metadata objects by category | `find_by_type(category)` for folders/types; `search_objects(query)` for technical name or synonym. |
| Full object structure | `get_object_full_structure(name)`; use `parse_object_xml(path)` when you need live XML completeness or index freshness is uncertain. |
| Find modules of an object | `find_module(name)`, then `read_file(path)`, `extract_procedures(path)`, `find_exports(path)`, `code_metrics(path)`. |
| Find a routine / read its body | `search_methods(query)` with an index; otherwise `find_module(name)` + `extract_procedures(path)` + `read_procedure(path, proc_name)`. |
| Full-text code or XML search | `git_search(pattern, path="", file_types="", regex=False, mode="lines", max_results=...)` for git-backed sources; `safe_grep(pattern, name_hint, max_files)` for targeted BSL search; standard `grep_read` only for narrowed paths. |
| Callers / call hierarchy | `find_callers(name, hint, max_files)`, `find_callers_context(proc_name, module_hint, offset, limit)`, `find_call_hierarchy(name, direction="callers", depth=1..3, module_hint="")`. |
| Metadata usages and references | `find_references_to_object(object_ref, kinds=None, limit=...)` for declarative metadata references; `find_code_usages(object_ref, kind=None, limit=...)` for code usages. |
| Register movements / writers | `find_register_movements(document_name)`, `find_register_writers(register_name)`, `analyze_document_flow(document_name)`. |
| Forms | `parse_form(object_name, form_name="", handler="")`; use `find_module(name)` + `read_procedure` for form-module handler bodies. |
| Roles / rights | `find_roles(object_name)`. |
| Event subscriptions / scheduled jobs | `find_event_subscriptions(object_name, ...)`, `find_scheduled_jobs(name)`. |
| Integrations | `find_http_services(name)`, `find_web_services(name)`, `find_xdto_packages(name)`, `find_exchange_plan_content(name)`. |
| Enumerations / predefined / defined types | `find_enum_values(enum_name)`, `find_predefined(name, object_name)`, `find_defined_types(name)`. |
| Extension overrides | `detect_extensions()`, `find_ext_overrides(extension_path, object_name)`, `get_overrides(object_name, method_name)`. |
| Index status | `get_index_info()` inside the session; `rlm_index(action="info", project=...)` outside the session. |

## No direct replacement

`rlm-tools-bsl` is an exploration server, not an XSD/XML validator and not a platform-reference server.

- For metadata XML validation use the `1c-metadata-manage` validators (`form-validate.ps1`, `meta-validate.ps1`, `role-validate.ps1`, `skd-validate.ps1`, `mxl-validate.ps1`, `cf-validate.ps1`, etc.) or the available XML-generation/validation skill for the active tool.
- For platform API/member discovery use `1C-docs-mcp` (`docinfo` / `docsearch`) and, when needed, the live syntax-reference tools exposed by the environment.
- For 1C help / methodological documentation use `1C-docs-mcp`, `1c-code-check-mcp` (`search_1c_documentation`, `its_help` → `fetch_its`), or repository help files found through `git_search`.

## Replacement map from the retired server

| Retired tool | Use with `rlm-tools-bsl` |
|---|---|
| `metadatasearch` | `search_objects`, `find_by_type`, `find_attributes`, `search(scope="objects"|"attributes")`; use `git_search` for raw XML literals. |
| `get_metadata_details` | `get_object_full_structure`; `parse_object_xml` for live XML structure. |
| `codesearch` | `search`, `search_methods`, `git_search`, `safe_grep`, `grep_read` on narrowed paths. |
| `search_function` | `search_methods`; fallback to `extract_procedures` / `find_exports` over paths from `find_module`. |
| `get_module_structure` | `extract_procedures`, `find_exports`, `code_metrics`, `read_file`. |
| `get_method_call_hierarchy` | `find_call_hierarchy` or `find_callers_context`. |
| `graph_dependencies` | `find_references_to_object`, `find_code_usages`, `find_register_movements`, `find_register_writers`, `analyze_document_flow`, `analyze_subsystem`. |
| `bsl_scope_members` | No direct RLM equivalent; use platform docs for built-in APIs and `search_methods` / `find_exports` for project routines. |
| `search_forms` | `parse_form` plus `search_objects` / `git_search` for form names and XML literals. |
| `inspect_form_layout` | `parse_form`; for handler bodies use `read_procedure` on the returned form module path. |
| `helpsearch` | No direct RLM equivalent; use `docinfo` / `docsearch`, `search_1c_documentation`, or `git_search` over dumped help files. |
| `get_xsd_schema` / `verify_xml` | No direct RLM equivalent; use metadata-management validators / XML-generation validation tools. |
| `reindex` / `stats` | `rlm_index(action="build"|"update"|"info"|"drop")` and `get_index_info()`. |
