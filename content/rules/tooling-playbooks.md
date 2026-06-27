---
description: Per-task MCP tool playbooks (writing code, review, refactoring, error fixing, performance, forms, integrations, documentation)
alwaysApply: false
category: tooling
---

# Tool Usage by Task — Playbooks

The MCP server catalog, fallback order (`metacode → rlm-tools-bsl → RLM literal / narrowed retry → Grep` for project-source search), and per-server tool descriptors live in the `mcp-1c-tools` skill (`content/skills/mcp-1c-tools/SKILL.md`, `docs/<server>.md`). `AGENTS.md` only defines the short obligation rules and points here.

## Minimum Evidence Matrix

Use the smallest set that closes the real context gaps. Do not promote a task to a heavier path just to satisfy a generic checklist.

| Task shape | Required before edit | Required after edit |
|---|---|---|
| **Quick-fix BSL** (single procedure, no metadata / transaction / public API impact) | Read the target module / procedure and any directly referenced helper needed to understand the bug | `diagnostics` on the touched `.bsl` file |
| **Full-cycle BSL** | `templatesearch` when a reusable pattern may exist; `search_code` / RLM helpers (`search`, `search_methods`, `find_definition`, `git_search`, `safe_grep`) for local patterns; `search_metadata` (`object_structure` / focused templates) or RLM helpers (`get_object_full_structure`, `find_attributes`) when metadata shape affects the code; platform / БСП / ITS docs only when versioned API or standard behaviour matters | `diagnostics` → `check_1c_code`; impact analysis when public surface or metadata usage changed |
| **Metadata XML / forms** | Similar object/form examples, metadata lookup through `search_metadata` / RLM helpers; prefer `1c-metadata-manage` over hand edits | Relevant `1c-metadata-manage` validator (`form-validate`, `meta-validate`, `role-validate`, `skd-validate`, etc.); metadata validation / form compilation where applicable |
| **Integrations / platform APIs** | Existing integrations, templates, relevant БСП APIs, platform docs for exact API names / version availability, security requirements | `diagnostics` → `check_1c_code`; ITS check when relying on an ITS standard |
| **Markdown / rules / docs** | Read affected docs and referenced files needed for consistency | Structural checks only: paths, links, anchors, duplicate / conflicting wording |

## Writing New Code

1. **templatesearch** — find similar implementations.
2. **search_metadata** — verify the target metadata object with focused templates (`object_structure`, forms, modules, rights, events as needed).
3. **search_code** → **rlm-tools-bsl** (`search`, `search_methods`, `find_definition`, `git_search`) — review existing patterns in the configuration.
4. **rlm-tools-bsl** (`find_definition`, `search_methods`; fallback `find_module` + `extract_procedures`) — find an existing procedure/function by name for reuse.
5. **rlm-tools-bsl** (`find_module` + `get_module_outline` + `extract_procedures` + `code_metrics`) — overview of the module you intend to edit.
6. **rlm-tools-bsl** (`get_object_full_structure`, `find_attributes`, `parse_object_xml`) — verify metadata structure and attribute types.
7. **get_function_info / search_syntax** for platform APIs; **rlm-tools-bsl** (`search_methods`, `find_exports`) for project routines.
8. **get_function_info** — verify built-in functions by exact name; **search_syntax / suggest_completion** — find candidates by name or prefix.
9. **Installed `bsp-*` skills** — find reusable БСП functions and patterns; confirm project-specific availability through source search when needed.
10. **diagnostics** — verify touched `.bsl` files after writing.
11. **check_1c_code** — find syntax, logic, performance, style, and ITS-compliance defects.
12. **execute_code** query parse wrapper (`1c-mcp-toolkit`, if available) — when the change introduces a new / non-trivial query string (module code, DCS data set, dynamic list), parse-check it against the live IB by running `Новый Запрос(<query>).НайтиПараметры()` inside a read-only `execute_code` fragment before delivery. Especially important after non-deterministic AI generation (`modify_1c_code` / `ask_1c_ai`). Use `execute_query` with an intentionally empty result only when the query can be safely rewritten to return no rows.

## Code Review

1. **search_code** → **rlm-tools-bsl** (`search`, `search_methods`, `find_definition`, `git_search`) — verify pattern compliance.
2. **search_metadata** usage / movement / call-graph templates → **rlm-tools-bsl** (`find_references_to_object`, `find_code_usages`, `find_register_movements`) — impact analysis of the change.
3. **search_metadata** call-graph templates → **rlm-tools-bsl** (`find_call_hierarchy`, `find_callers_context`, `find_path`) — BSL call chains, callers/callees, and routine-to-routine reachability.
4. **rlm-tools-bsl** (`get_object_full_structure`, `find_attributes`, `find_references_to_object`) — correct metadata usage.
5. **get_function_info** — verify method/property existence; **search_syntax** — search by name fragment.
6. **check_1c_code** — bugs, performance issues, style, and ITS compliance.
7. **search_its** → **fetch_its** — cross-check against ITS standards.

## Architecture Design

1. **search_metadata** — focused passports of key metadata objects (`object_structure` plus forms / rights / modules / events when needed).
2. **rlm-tools-bsl** (`search_objects`, `get_object_full_structure`, `find_attributes`) — existing metadata structure.
3. **search_metadata** usage / movement / call-graph templates → **rlm-tools-bsl** (`find_references_to_object`, `find_code_usages`, `find_register_movements`) — dependency map across metadata references, code usages, movements and calls.
4. **search_metadata** (`find_objects_using_object`) — find all objects referencing the given one.
5. **search_code** → **rlm-tools-bsl** (`search`, `search_methods`, `find_definition`, `git_search`) — existing architectural patterns.
6. **search_metadata** call-graph templates → **rlm-tools-bsl** (`find_call_hierarchy`, `find_callers_context`, `find_path`) — code coupling, call chains, and routine-to-routine reachability.
7. **templatesearch** — architectural templates.
8. **ask_1c_ai** — architectural questions to 1С:Напарник (treat as a hint, not authority).

## Error Fixing

1. **get_event_log** (`1c-mcp-toolkit`, if available) — fetch the exact text, timestamp and affected metadata of the relevant live-IB error before forming hypotheses. Start with `levels=["Error"]`, a recent `start_date`, and `limit=1`; add `metadata_type`, `user`, `application`, or `comment_contains` when the log is noisy. Avoids guessing what the user "probably saw". Skip when the failing scenario is not yet reproduced in the connected IB.
2. **diagnostics** — syntax and analyzer errors.
3. **check_1c_code** — logic, performance, style, and ITS compliance issues.
4. **rlm-tools-bsl** (`find_definition`, `search_methods`; fallback `find_module` + `extract_procedures`) — locate the failing procedure/function.
5. **search_code** → **rlm-tools-bsl** (`search`, `git_search`, `read_procedure`) — related patterns and the minimal routine body needed.
6. **rlm-tools-bsl** (`find_module` + `get_module_outline` + `extract_procedures` + `code_metrics`) — module context around the error.
7. **search_metadata** call-graph templates → **rlm-tools-bsl** (`find_call_hierarchy`, `find_callers_context`, `find_path`) — how the error propagates through the call chain.
8. **get_function_info** — verify function/method names; **search_syntax / suggest_completion** — fallback by name or prefix.
9. **rlm-tools-bsl** (`search_objects`, `get_object_full_structure`, `find_attributes`) — verify metadata names and attributes.
10. **execute_code** query parse wrapper (`1c-mcp-toolkit`, if available) — when the suspect path is a query string, parse-check it through `Новый Запрос(<query>).НайтиПараметры()` before deeper investigation.
11. **execute_query** (`1c-mcp-toolkit`, if available) — read-only query against the live IB to confirm a data-state hypothesis without changing production code.
12. **execute_code** (`1c-mcp-toolkit`, if available) — run a small read-only BSL fragment in the live IB to verify a platform-version-specific behaviour. Default to read-only; **never** wrap a mutation without explicit user consent (see `docs/1c-mcp-toolkit.md → Safety and discipline`).
13. **modify_1c_code** — targeted AI fix (treat output as a draft, re-validate).

## Performance Optimization

1. **search_code** → **rlm-tools-bsl** (`search`, `search_methods`, `git_search`) — locate slow patterns ("медленный запрос", "цикл по выборке").
2. **search_metadata** call-graph templates → **rlm-tools-bsl** (`find_call_hierarchy`, `find_callers_context`, `find_path`) — identify hot call chains and reachability between routines.
3. **search_metadata** call-graph / usage templates → **rlm-tools-bsl** (`find_references_to_object`, `find_code_usages`, `find_register_movements`) — objects that cause cascading issues.
4. **rlm-tools-bsl** (`get_object_full_structure`, `find_attributes`, `find_register_movements`) — verify indexes and metadata structure.
5. **check_1c_code** — bottleneck analysis.
6. **modify_1c_code** — targeted AI optimization only with an explicit instruction; re-validate with `diagnostics` and `check_1c_code`.
7. **templatesearch** — optimized templates.
8. **search_its** → **fetch_its** — ITS performance standards.
9. **execute_code** query parse wrapper → **execute_query** (`1c-mcp-toolkit`, if available) — parse-check the rewritten query through `Новый Запрос(<query>).НайтиПараметры()`, then run it read-only against the live IB to compare row counts / spot Cartesian explosions / confirm a virtual-table state. Use only on a test or copy IB when production data volumes matter.

## Refactoring

1. **search_metadata** — passport of the object being refactored (`object_structure` plus focused facets).
2. **search_metadata** usage / movement / call-graph templates → **rlm-tools-bsl** (`find_references_to_object`, `find_code_usages`, `find_register_movements`) — what breaks on change.
3. **search_metadata** (`list_callers_of_routine` / `call_graph_subtree`) → **rlm-tools-bsl** (`find_call_hierarchy`, `find_callers_context`, `find_path`) — all callers and reachability where needed.
4. **search_metadata** (`find_objects_using_object` / `find_usages_of_object`) — every type reference before renaming/removing.
5. **search_code** → **rlm-tools-bsl** (`search`, `git_search`, `find_code_usages`) — every code pattern related to the object.
6. **search_code** → **rlm-tools-bsl** (`git_search`, `find_code_usages`) — post-refactor verification that no old references remain.
7. **check_1c_code** — validate the result.

## Generating / Modifying Metadata XML

1. **rlm-tools-bsl** (`search_objects`, `find_by_type`, `get_object_full_structure`) — similar objects as examples.
2. **1c-metadata-manage** domain docs / validators — exact XML structure and validation command for the target metadata type.
3. Write/modify XML against the schema and examples.
4. **Relevant `1c-metadata-manage` validator** — validate the generated XML; fix errors.
5. Use the **1c-metadata-manage** skill for compilation and deployment.

## Form Analysis and Generation

1. **rlm-tools-bsl** (`parse_form`, `search_objects`, `git_search`) — similar existing forms in the configuration.
2. **rlm-tools-bsl** (`parse_form`) — structure of the found form (elements, bindings, commands, events).
3. **rlm-tools-bsl** (`search_objects`, `find_by_type`, `get_object_full_structure`) — metadata objects for XML references.
4. **1c-metadata-manage form docs** — valid `Form.xml` structure and generation / edit commands.
5. Generate `Form.xml` based on examples and schema.
6. **form-validate** from `1c-metadata-manage` — validate `Form.xml`.
7. **1c-metadata-manage** skill (form-manage) — compilation and validation.

## Integrations

Use this playbook when writing HTTP services / clients, REST integrations, file or message-queue exchanges, webhooks. Domain rules — `integrations-add.md`.

1. **Installed `bsp-*` skills** — check for ready-made БСП subsystems ("Интернет-поддержка пользователей", "Обмен данными", "Получение файлов из Интернета", "Цифровая подпись"); confirm availability in this project through `search_code` / `rlm-tools-bsl` when the implementation will rely on them.
2. **templatesearch** — integration templates (HTTP request, JSON parsing, signed payloads, retry policy).
3. **search_code** → **rlm-tools-bsl** (`find_http_services`, `find_web_services`, `search`, `git_search`) — existing integrations in the configuration ("HTTP запрос", "отправка JSON", "парсинг ответа").
4. **get_function_info** — verify platform types by exact name (`HTTPСоединение`, `HTTPЗапрос`, `ЧтениеJSON`, `ЗаписьJSON`, `ЗаписьXML`, `ЧтениеXML`).
5. **search_syntax / suggest_completion** — fallback when the exact platform-API name is unknown.
6. **1c-metadata-manage / XML validation tooling** — when the contract is XML with a known XSD.
7. **search_its** → **fetch_its** — ITS articles on long-running operations, secure password storage, asynchronous external components.
8. **rlm-tools-bsl** (`search_methods`, `find_module`, `extract_procedures`) — locate or extend the integration common module (typically `*HTTPClient`, `*Integration`, `*Exchange`).
9. After implementation: **diagnostics** → **check_1c_code**.

## Documentation

1. **rlm-tools-bsl** (`search`, `search_methods`, `git_search`) — find code to document.
2. **rlm-tools-bsl** (`get_object_full_structure`, `find_attributes`, `find_references_to_object`) — metadata structure.
3. **rlm-tools-bsl** (`find_module`, `get_module_outline`, `extract_procedures`, `find_definition`) — module outline and list of procedures/functions.
4. **get_function_info** — syntax-reference information by exact name; **search_syntax** — search by name fragment.
5. **search_1c_documentation** — existing help articles.
6. **search_its** → **fetch_its** — methodological ITS articles.
7. **search_1c_documentation** — version-specific platform documentation.

## Comparing Platform Versions

1. **diff_1c_documentation_versions** — what changed between versions.
2. **search_1c_documentation** — documentation for a specific version.
