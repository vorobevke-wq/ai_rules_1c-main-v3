# rlm-tools-bsl — tool catalog

Intent-first, token-efficient search and navigation backend for 1C BSL repositories.
Upstream: <https://github.com/Dach-Coin/rlm-tools-bsl>.
Last synced against upstream `v1.27.0` / `master` on 2026-07-06.

`rlm-tools-bsl` does not expose one MCP tool per 1C operation. It exposes a small session API and 62 sandbox helpers; inside `rlm_execute`, the agent runs Python code that calls helpers for BSL files, metadata XML, forms, references, call graphs, data paths, object profiles, and full-text search. Prefer this server over raw agent-side `Grep` / `rg` and direct file reads whenever a task touches 1C source: it keeps large file bodies on the server and returns only compact `print()` output.

> Load this file only if the `rlm-tools-bsl` server is actually available in the current session.

## Read order for agents

Use this file in layers:

1. For ordinary repository search, read `Top-level MCP tools` -> `Session discipline` -> `Sandbox helper map`.
2. For subsystem/configuration reports, audits, and specifications, also apply `Agent analysis method`.
3. For migration from the retired indexed metadata server, use `Replacement map from the retired server` only after the normal RLM helper choice is clear.

Do not treat every section as a separate workflow. `Session discipline` is the default workflow; later sections add routing details, report-only gates, or migration aliases.

## Top-level MCP tools

| Tool | Stable documented input | Purpose | When to use |
|---|---|---|---|
| **rlm_projects** | `action`, optional `name`, `path`, `description`, `new_name`, `password`, `clear_password` | Manage the server-side project registry. `path` may point to a configuration root or a parent container where the main configuration is auto-detected. Mutating actions require a user-supplied project password when the server returns `approval_required`. | List registered projects; register a frequently used 1C source tree so later calls can use `rlm_start(project="...")`. Never invent a password; ask the user only after `approval_required`. |
| **rlm_index** | `action`, `project` or `path`; optional `no_calls`, `no_metadata`, `no_fts`, `no_synonyms`, `confirm` | Manage the optional SQLite index. `build` / `update` run in the background and return immediately; check progress through `info`. `build` / `update` / `drop` require a registered `project` and project password (`confirm`). `info` / `drop` can recover the effective path from index metadata when the source directory has been removed. | Before large repositories, stale search results, call-graph/data-path analysis, or when helpers requiring index tables return degraded results. Use `path` mainly for `info`; register the project before mutating index actions. |
| **rlm_start** | `query`, `path` or `project`; optional `effort`, `max_output_chars`, `max_llm_calls`, `max_execute_calls`, `execution_timeout_seconds`, `include_metadata` | Open a fast read-only search and navigation session and return `session_id`, strategy, `effective_effort`, limits, warnings, extension context, `index` (`index_status`, `builder_version`, feature flags/counts), and available helper signatures. Default effort is server-side `auto` unless overridden. `path` may be the configuration root or a parent container with auto-detected main configuration. | First call for repository exploration after `rlm_projects(action="list")`. Use `project` when the workspace path or user input matches a registered project; use `path` only when no matching project/index exists or the user explicitly requested path-based fallback. For full reports and audits pass `effort="high"` explicitly. |
| **rlm_help** | optional `topic`, `helpers`, `category`, `section`, `format`, `include_code` | Default slim-mode companion that returns recipes, exact helper details, disambiguation rules, workflow / performance / batching / IO / critical sections, and helper categories. In `RLM_STRATEGY_MODE=full` this tool is not registered because the strategy is fully inlined in `rlm_start`. | Call before `rlm_execute` on non-trivial analysis; call with `helpers=[...]` before unfamiliar helpers; call `section="disambiguation"` when choosing between similar helpers. |
| **rlm_execute** | `session_id`, `code`, optional `detail_level`, `max_new_variables` | Run BSL search and navigation helpers by executing Python in the sandbox. Variables persist between calls; only printed output is returned to the agent context; response metadata may include `efficiency_hints`. | Batch several related helper calls, filter/summarize server-side, and print compact conclusions. This is where `get_object_profile`, `get_object_modules`, `find_module`, `find_definition`, `get_module_outline`, `find_call_hierarchy`, `find_path`, `find_data_path`, `git_search`, `parse_form`, etc. are called. |
| **rlm_end** | `session_id` | Close the exploration session and free resources. | Always call when the RLM exploration is complete. |

## Session discipline

1. Before the first `rlm_start` in a 1C repository, call `rlm_projects(action="list")`.
2. If the current workspace path is inside a registered project path, or the user mentions a registered project name, start with `rlm_start(query=..., project="<name>")`, not with `path`.
3. After `rlm_start`, verify `index.loaded`. If `index.loaded=false` and a matching registered project exists, immediately call `rlm_end` and restart with `project="<name>"`.
4. Do not proceed with filesystem fallback unless no matching registered project/index exists or the user explicitly requested path-based fallback.
5. Read the `rlm_start` response before acting: check `effective_effort`, `limits`, `warnings`, `extension_context`, `detected_custom_prefixes`, and `index.index_status` (`ok` / `stale_age` / `stale_content` / `missing` / `incomplete`). `rlm_start.index` already carries `builder_version`, feature flags, and counts, so a separate startup `get_index_info()` call is usually unnecessary. If the index is missing/stale/incomplete or a helper returns `partial=True`, do not present partial helper output as complete. If `extension_context.nearby_extensions_truncated=true`, treat `nearby_extensions` as a top-N hint only and call `detect_extensions()` / `get_overrides()` when complete extension facts are needed.
6. In default slim strategy mode, call `rlm_help(...)` before non-trivial `rlm_execute` calls. Choose one route from `rlm_help routing`: `topic` for a business recipe, `helpers` for exact helper signatures, `category` for a helper list, or `section` for workflow / disambiguation / performance / batching / IO / critical guidance.
7. Batch operations inside one `rlm_execute`: search candidates, read the top files/procedures, compute a summary, then `print()` only the useful result. Prefer aggregate helpers (`get_object_profile`, `get_object_modules`) before assembling the same overview manually. Use list overloads where available (`read_procedure`, `find_callers_context`, `find_enum_values`). Variables persist between calls; reuse assigned results instead of repeating identical helper calls. If `rlm_execute` returns `efficiency_hints`, treat them as actionable guidance for later calls.
8. Never run broad agent-side `Grep` / `rg` over `path='.'` or whole top-level directories on large 1C configurations. Prefer metadata/code helpers, `find_module` + targeted reads, and RLM full-text helpers (`git_search`, then `safe_grep` / `grep_read` on narrowed paths) before leaving the server.
9. End with `rlm_end(session_id)` unless the parent explicitly needs to continue the same session.

## Agent analysis method

Upstream `docs/AGENT_INSTRUCTIONS.md` defines a reporting discipline for agents that analyze a 1C configuration through `rlm-tools-bsl`. Apply this block when the task asks for a subsystem/configuration analysis, specification, audit, or other report where completeness matters.

Do not apply the hard gates below to a quick lookup such as "find this procedure" or "where is this attribute used". Quick lookups still follow `Session discipline`.

Base rule: facts come from data, not from memory or object names. Any claim about object composition, counts, attribute types, links, or business logic must be backed by helper output that actually read the relevant source. If the helper did not read it, treat it as unknown.

Two hard gates for subsystem-composition reports:

1. Composition and object counts come only from raw subsystem XML `Content` (`<xr:Item>` entries). Summary helpers such as `analyze_subsystem` may be used to locate the subsystem file, but not as the authoritative source of the object list or counts.
2. The last substantive `rlm_execute` before delivery must re-read `Content`, count `M` programmatically, compare it with the report total and the sum by object type, and print `СВЕРКА ОК: итог = сумма_по_типам = M = N`. If the numbers differ, fix the report and repeat the check.

Report-specific operational rules from the upstream agent instructions:

- Follow `Session discipline` for `rlm_start`, `rlm_help`, batching, and final `rlm_end`; for reports, use `effort="high"` unless the task is explicitly narrow.
- Use sandbox helpers as the source for 1C project data. Do not replace them with agent-side file reads or raw `Grep` / `rg` over large source files; that defeats the token-saving purpose of the server. Saving the final report is the normal file-tool exception.
- Check `rlm_start.index` at the start of analytic work. It carries `index_status`, `builder_version`, feature flags, and table counts; call `get_index_info()` only when you need an explicit in-session refresh or diagnostics beyond startup data. If the index is absent, stale, incomplete, or a helper returns `partial=True`, do not present partial data as complete; state the limitation and consider `rlm_index(action="build"|"update")` when the user can provide the project password.
- Treat a helper miss as inconclusive until you have done a broader search. Escalate from structural helpers to `search(...)` and then `git_search(...)` for GUIDs, user messages, query fragments, form/right/DCS XML attributes, and technical literals before writing "not found".
- For key logic, read procedure bodies, not only signatures or names. Posting and movements may be implemented through common modules, event subscriptions, or register-writer helpers rather than directly in the object module.
- Keep the report slice targeted: filter event subscriptions, movements, references, and counts to the subsystem/custom prefixes where possible. Do not report configuration-wide noise as subsystem-specific evidence.
- Separate objects included in subsystem `Content` from related objects that are merely used by attributes, queries, forms, or movements. Put related-but-not-included objects in a separate section.
- Use exact metadata types for attributes, dimensions, resources, and columns (`CatalogRef.X`, `DocumentRef.X`, `Number`, `Boolean`, `EnumRef.X`, etc.).

Recommended workflow for a full subsystem/configuration report:

1. `rlm_start(..., effort="high")`, inspect warnings and `rlm_start.index` (`index_status`, `builder_version`, feature flags/counts).
2. `rlm_help` for workflow and task-specific recipes.
3. Locate the subsystem XML, read raw `Content`, and compute composition counts by type.
4. For every object from `Content`, use `get_object_profile(name)` as the first compact overview when applicable, then collect missing structure, tabular sections, dimensions/resources, forms, and relevant flags with exact types.
5. Collect applicable surroundings: roles, functional options, subscriptions, based-on links, print forms, register writers/movements, and integrations.
6. Read key procedure bodies and follow actual execution through calls/subscriptions/common modules.
7. Run the final programmatic reconciliation check and self-review before delivery.
8. Call `rlm_end` after the analysis is complete.

## `rlm_help` routing

`rlm_help` returns JSON `{mode, result, warnings}` and has priority-ordered dispatch:

| Input | Result |
|---|---|
| no arguments | Menu of topics, categories, sections, and helper count. |
| `topic="..."` | Compact or full recipe for a business domain / alias. Current notable topics include rights, posting, forms, references, integrations, reachability, and data path analysis. |
| `section="disambiguation"` | Structured helper-choice rules; pass `helpers=["a", "b"]` to narrow to a specific pair. |
| `section="workflow" / "performance" / "batching" / "io" / "critical"` | Raw strategy section. |
| `helpers=["name1", "name2"]` | Exact signatures, keywords, and recipes for helpers. |
| `category="discovery" / "code" / "xml" / "composite" / "business" / "extension" / "navigation"` | One-line helper list for the category. |

## Sandbox helper map

The complete helper list is returned by `rlm_start.available_functions`; detailed helper docs are available through `rlm_help(helpers=[...])`. Use this table to choose helpers for new RLM work:

| Need | RLM helper(s) inside `rlm_execute` |
|---|---|
| Broad search by object / method / attribute / region | `search(query, scope="all", limit=...)`; refine with `search_objects`, `search_methods`, `search_regions`, `search_module_headers`, `find_attributes`, `find_predefined`. |
| Find metadata objects by category | `find_by_type(category, name="")` for folders/types; `search_objects(query)` for technical name or synonym. |
| Full object structure / overview | `get_object_profile(name, sections=None, include_flow=False, include_code_usages=False, limit=20)` for a compact one-call overview (structure, modules, registers, subscriptions, roles, functional options); `get_object_full_structure(name)` for metadata-side structure; `parse_object_xml(path)` when you need live XML completeness or index freshness is uncertain. |
| Find modules of an object | `get_object_modules(name, include_methods=False)` for all modules plus outlines and totals in one call; otherwise `find_module(name="", module_type="", category="")`, then `get_module_outline(path, include_methods=False)`, `extract_procedures(path)`, `find_exports(path)`, `code_metrics(path)`, or `read_file(path)` when the exact file is already narrowed. |
| Find a routine / read its body | `find_definition(name, module_hint="", limit=50)` for go-to-definition; `search_methods(query)` with an index; otherwise `find_module(name)` + `extract_procedures(path)` + `read_procedure(path, proc_name)`. `read_procedure(path, [names...])` batches several procedure bodies from one module. Use `module_hint` for common 1C names such as `ОбработкаПроведения` / `ПередЗаписью`. |
| Module outline before reading bodies | `get_module_outline(path_or_object, include_methods=True/False, no_live=False)` for `#Область` tree, method/export counts, orphan methods, and line ranges. Use it as the first hop for 5K-15K line modules; with an object name, inspect `_meta.resolved_from_name`, `candidates`, and `ambiguous`. |
| Full-text code or XML search | `git_search(pattern, path="", file_types="", regex=False, ignore_case=False, mode="lines", max_results=..., exclude_path="")` for git-backed sources and technical literals. Use `exclude_path="Forms,Templates"` or similar literal names to suppress noisy zones. If `git_search` is unavailable or too broad, use `safe_grep(pattern, name_hint, max_files)` for targeted BSL search; `grep` / `safe_grep` reject catastrophic-backtracking regex patterns with `ValueError`. Use `grep_read` only on narrowed paths. Raw agent-side `Grep` / `rg` is the last fallback, after RLM helpers are insufficient. |
| Callers / call hierarchy | `find_callers(name, module_hint, max_files)` for a compact first page; `find_callers_context(proc_name, module_hint, offset, limit)` for one-level callers with context (`proc_name` may be a list); `find_call_hierarchy(name, direction="callers", depth=1..3, module_hint="", include_triggers=False)` for multi-level callers. `include_triggers=True` annotates nodes with non-call inbound edges: event subscriptions, form events, scheduled jobs, and CFE overrides. |
| Call-path reachability | `find_path(from_name, to_name, max_depth=4, from_hint="", to_hint="", include_triggers=False)` for a forward path through the BSL call graph. Read `_meta.precision` (`exact` vs `heuristic`), `_meta.budget_exceeded`, and `_meta.ambiguous` before treating a missing path as proof of absence; if ambiguous candidates are returned, repeat with `from_hint` / `to_hint`. |
| Metadata usages, references, and data paths | `find_references_to_object(object_ref, kinds=None, limit=..., include_code=False)` for declarative metadata references; `find_code_usages(object_ref, kind=None, limit=...)` for code usages; `find_data_path(from_object, to_object, max_depth=4, kinds=None)` for an N-hop metadata-reference path. `find_data_path` endpoints must include metadata prefixes such as `Документ.X` / `Справочник.Y` or `Document.X` / `Catalog.Y`. |
| Register movements / writers | `find_register_movements(document_name)`, `find_register_writers(register_name)`, `analyze_document_flow(document_name)`. These helpers and `get_object_profile(...).sections.registers` filter movement-collection methods such as `Движения.Записать()` / `Движения.Очистить()` at read time, so legacy indexes do not need a rebuild for this noise fix. |
| Forms | `parse_form(object_name, form_name="", handler="")`; use `find_module(name)` + `read_procedure` for form-module handler bodies. |
| Roles / rights | `find_roles(object_name)` for detailed role rows; `get_object_profile(..., sections=["roles"])` for the compact aggregate, including object-level and member-level grants. |
| Event subscriptions / scheduled jobs | `find_event_subscriptions(object_name, ...)`, `find_scheduled_jobs(name)`. |
| Integrations | `find_http_services(name)`, `find_web_services(name)`, `find_xdto_packages(name)`, `find_exchange_plan_content(name)`. |
| Enumerations / predefined / defined types | `find_enum_values(enum_name_or_list)`, `find_predefined(name, object_name)`, `find_defined_types(name)`. |
| Extension overrides | `detect_extensions()` (`nearby_extensions[].overrides_count`), `find_ext_overrides(extension_path, object_name)` and `get_overrides(object_name, method_name)` (`overrides` capped at 200, check `total` and `truncated`). |
| Index status | `rlm_start.index.index_status` for startup state; `get_index_info()` inside the session when an explicit refresh/diagnostic is needed; `rlm_index(action="info", project=...)` outside the session. `info` / `drop` can work after the source directory is gone if index metadata contains the base path; `drop` also removes the file cache when present and may return `dropped_cache`. |

## Helper and runtime notes

- `rlm_start` supports server-side `effort="auto"`: the server chooses `medium` for simple lookups and `high` for multi-aspect analysis. Precedence is `RLM_FORCE_EFFORT` > explicit `effort` > `auto`; numeric limits are explicit `max_execute_calls` / `max_llm_calls` > `RLM_MAX_EXECUTE_CALLS` / `RLM_MAX_LLM_CALLS` > the effort preset. Read `effective_effort` and `limits`; a `== SERVER LIMIT OVERRIDE ==` strategy banner means server env changed the numeric limits. Full subsystem/configuration reports still use explicit `effort="high"`.
- On extension-heavy main configurations, agent-facing extension lists can be capped through `RLM_EXT_LIST_CAP` (default `20`). When `nearby_extensions_truncated=true`, use `detect_extensions()` for the full list and `get_overrides()` / `find_ext_overrides()` for details. `RLM_EXT_OVERRIDE_DETAIL` controls how many named override lines are included in the startup strategy; default `0` keeps only counters and on-demand pointers.
- `rlm_start.index` carries startup discovery data: stable `index_status`, `builder_version`, `has_*` feature flags, and row counts for indexed tables. Use this before spending a separate `get_index_info()` call.
- The current index schema uses `BUILDER_VERSION` `14`. A production index rebuild is not required solely because the server package was upgraded when the schema version is unchanged. During an in-place rebuild, `index_status="incomplete"` means new sessions must not treat the index as usable until the build finishes.
- Register movement helpers and the `get_object_profile` register section apply a read-time filter for movement-collection method calls. Treat `Движения.Записать()`, `Движения.Очистить()`, `Движения.Прочитать()`, and similar method calls as helper-filtered noise, not as real register names; no index rebuild is required for old indexes.
- `get_object_profile` is the preferred first hop for a compact object overview. Its roles section counts both object-level grants and member-level grants such as commands, so use it for the first aggregate view; use `find_roles` or role XML reads when detailed permission rows are needed. `get_object_modules` is the code-side aggregate for modules and outlines; combine it with `get_object_full_structure` for metadata-side details.
- `rlm_execute` may return `efficiency_hints` such as batch, reuse-variable, read-files, or `redundant_get_index_info` guidance. Follow them in later calls; they are response metadata, not helper stdout. If `redundant_get_index_info` appears, use the already returned `rlm_start.index` data instead of spending an early extra `get_index_info()` call.
- `read_procedure`, `find_callers_context`, and `find_enum_values` accept list inputs for batching. Keep scalar calls for one-off reads, but pass lists when collecting several procedures/callers/enumerations in one module/task.
- Sandbox error hints can suggest real helper names for guessed calls and print actual signatures for unsupported keyword arguments. Use those hints to self-correct, but still confirm unfamiliar helper signatures through `rlm_help(helpers=[...])`.
- The sandbox intentionally rejects common unsafe Python patterns, including dangerous dunder/frame introspection, `operator` imports, and attribute assignment/deletion. Use plain functions, `lambda`, dictionaries, and local variables in `rlm_execute` snippets.
- `find_definition(name, module_hint="", limit=50)` and `get_module_outline(path, include_methods=True)` are read-only helper additions. They use existing index tables when available and degrade to live parsing where possible; no index rebuild is required solely because these helpers appeared.
- `find_path(...)`, `find_data_path(...)`, and `find_call_hierarchy(..., include_triggers=True)` depend on index freshness for trustworthy graph analysis. If the index is old, missing, or marked partial, report that limitation and verify critical findings with targeted reads / references. If `find_path` reports an ambiguous endpoint, rerun with the relevant `from_hint` / `to_hint`.
- `search_regions(query, count_only=True)` and `search_module_headers(query, count_only=True)` provide cheap counts when a full result list would be too noisy.
- `find_files(name)` is instant on index hits, but a zero-hit index result can fall back to filesystem scanning; this is useful for stale indexes but may be slower on large trees.
- `get_overrides()` and `find_ext_overrides()` return the first 200 overrides plus `total` and `truncated`; filter by object/method/extension when `truncated=true`.
- `git_search(..., exclude_path="Forms,Templates")` excludes literal file or directory names at any depth. Do not pass glob patterns to `exclude_path`; invalid elements should be treated as a helper error, not silently broadened.
- `rlm_index(action="build"|"update"|"drop")` is an MCP-side administrative action: it requires a registered project and the project password. When the tool returns `approval_required`, ask the user for the password; do not guess or reuse unrelated credentials. `info` / `drop` can inspect or remove an index even after the source directory was deleted by recovering the base path from index metadata; `drop` also removes the file cache when possible and may include `dropped_cache` in the response.
- Runtime logs: `RLM_LOG_EXECUTE_CODE` logs the Python code passed to `rlm_execute` into `server.log` by default, capped at 300 characters unless overridden; set it to `0` / `false` / `off` / `no` when helper calls may expose sensitive object names, query fragments, or parameters in logs. `RLM_LOG_RETENTION_DAYS` trims `server.log` on startup (default `20`; `0` or negative disables time-based trimming).

## No direct replacement

`rlm-tools-bsl` is an exploration server, not an XSD/XML validator and not a platform-reference server.

- For metadata XML validation use the `1c-metadata-manage` validators (`form-validate.ps1`, `meta-validate.ps1`, `role-validate.ps1`, `skd-validate.ps1`, `mxl-validate.ps1`, `cf-validate.ps1`, etc.) or the available XML-generation/validation skill for the active tool.
- For platform API/member discovery use `1c-syntax` (`search_syntax`, `get_function_info`, `suggest_completion`, `validate_syntax`) and, when needed, the live syntax-reference tools exposed by the environment.
- For 1C help / methodological documentation use `onec-buddy-mcp` (`search_1c_documentation`, `search_its` → `fetch_its`) or repository help files found through `git_search`.

## Replacement map from the retired server

| Retired tool | Use with `rlm-tools-bsl` |
|---|---|
| `metadatasearch` | `search_objects`, `find_by_type`, `find_attributes`, `search(scope="objects"|"attributes")`; use `git_search` for raw XML literals. |
| `get_metadata_details` | `get_object_full_structure`; `parse_object_xml` for live XML structure. |
| `codesearch` | `search`, `search_methods`, `git_search`, `safe_grep`, `grep_read` on narrowed paths. |
| `search_function` | `find_definition` for exact go-to-definition; `search_methods`; fallback to `extract_procedures` / `find_exports` over paths from `find_module`. |
| `get_module_structure` | `get_module_outline`, `extract_procedures`, `find_exports`, `code_metrics`, `read_file` on narrowed paths. |
| `get_method_call_hierarchy` | `find_call_hierarchy`, `find_callers_context`, or `find_path` for reachability between two routines. |
| `graph_dependencies` | `find_references_to_object`, `find_code_usages`, `find_data_path`, `find_register_movements`, `find_register_writers`, `analyze_document_flow`, `analyze_subsystem`. |
| `bsl_scope_members` | No direct RLM equivalent; use platform docs for built-in APIs and `search_methods` / `find_exports` for project routines. |
| `search_forms` | `parse_form` plus `search_objects` / `git_search` for form names and XML literals. |
| `inspect_form_layout` | `parse_form`; for handler bodies use `read_procedure` on the returned form module path. |
| `helpsearch` | No direct RLM equivalent; use `1c-syntax` for syntax-reference names, `search_1c_documentation`, or `git_search` over dumped help files. |
| `get_xsd_schema` / `verify_xml` | No direct RLM equivalent; use metadata-management validators / XML-generation validation tools. |
| `reindex` / `stats` | `rlm_index(action="build"|"update"|"info"|"drop")` and `get_index_info()`. |
