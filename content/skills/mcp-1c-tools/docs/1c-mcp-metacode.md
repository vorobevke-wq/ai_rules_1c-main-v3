# 1c-mcp-metacode — tool catalog

Graph-backed metadata and BSL code search server based on Neo4j.
Upstream: <https://github.com/ROCTUP/1c-mcp-metacode>.

> Load this file only if the `1c-mcp-metacode` server is actually available in the current session (its tools are exposed in the tool schema).

> **Argument naming — do not invent.** The upstream documentation exposes three top-level MCP tools: `search_metadata`, `search_metadata_by_description`, and `search_code`. Pass the request text or JSON template in **`query`** unless the live tool schema for the current session says otherwise. Do not substitute `q`, `text`, `prompt`, `search_query`, `template`, or `json_query`.

## Top-level tools

| Tool | Stable documented input | Purpose | When to use |
|---|---|---|---|
| **search_metadata** | `query` | Structural metadata queries, object relationships, rights, form structure, event handlers, GUID lookup, and BSL call graph queries. Supports natural-language requests and deterministic JSON template operations when template mode is enabled. | First choice for metadata structure, object usage, register movements, call graph, roles, forms, commands, predefined values, and other graph facts. Prefer JSON templates over natural language when an operation exists. |
| **search_metadata_by_description** | `query` | Semantic / full-text search over metadata descriptions, comments, synonyms, help, and names. | Find objects by Russian business description, purpose, synonym, or domain wording when the technical name is unknown. |
| **search_code** | `query` | Search routines by description and retrieve BSL routine source with context such as owner, file, and line. | First choice for BSL code search by behaviour or description, and for getting the body of a known routine after `search_metadata` identified it. |

If the live schema exposes optional parameters (for example result limits, filters, project selection, or search mode), confirm the exact parameter names from the live schema before using them.

## `search_metadata` template operations

Use a JSON string in `query`, for example:

```json
{"operation":"object_structure","object":"Документ.Счет"}
```

Documented operation groups:

| Area | Operations |
|---|---|
| Object structure | `list_attributes`, `list_tabular_parts`, `list_resources`, `list_dimensions`, `object_structure` |
| Object usage and movements | `find_objects_using_object`, `find_usages_of_object`, `find_documents_making_movements_into_register` |
| Forms and UI | `list_forms`, `get_default_forms`, `list_form_controls`, `list_form_events`, `list_form_commands`, `list_form_bindings`, `list_form_attributes`, `list_form_event_handlers` |
| Commands and layouts | `list_commands`, `list_layouts`, `find_objects_by_command`, `find_objects_by_layout` |
| Types | `get_attribute_type`, `get_tabular_attribute_type`, `list_attributes_with_type`, `get_resource_type`, `get_dimension_type` |
| Enumerations and predefined values | `list_enum_values`, `list_predefined_of_object`, `find_predefined_by_name_in_object`, `find_predefined_by_flag` |
| HTTP services | `list_http_services`, `list_url_templates_of_service`, `list_url_methods_of_template` |
| Rights | `list_roles_with_access_to_target`, `list_access_targets_of_role`, `get_access_of_role_to_target` |
| Events | `list_event_subscriptions`, `list_event_subscriptions_of_object`, `get_event_subscription_sources` |
| Search and navigation | `list_objects_by_category`, `list_objects_by_name`, `resolve_qn`, `find_by_guid` |
| BSL call graph | `list_callees_of_routine`, `list_callers_of_routine`, `call_graph_subtree`, `find_calls_between_owners`, `find_unused_routines`, `list_exported_routines` |
| Modules and routines | `list_modules_of_owner`, `list_module_routines`, `list_common_module_routines`, `find_routines_by_name`, `find_routines_by_signature`, `list_routine_handled_events`, `list_event_subscription_handlers` |

## Replacement map for retired graph-server tools

| Retired tool | Use with `1c-mcp-metacode` |
|---|---|
| `get_object_dossier` | `search_metadata` with `object_structure` plus focused facet templates (`list_forms`, `list_modules_of_owner`, `list_module_routines`, rights / event / layout operations as needed). |
| `trace_impact` | `search_metadata` with usage and call-graph templates: `find_objects_using_object`, `find_usages_of_object`, `find_documents_making_movements_into_register`, `list_callers_of_routine`, `call_graph_subtree`; fallback to `graph_dependencies` on `1c-code-metadata-mcp` for flat dependency overview. |
| `trace_call_chain` | `search_metadata` with `list_callers_of_routine`, `list_callees_of_routine`, or `call_graph_subtree`; fallback to `get_method_call_hierarchy`. |
| `find_register_movement_docs` | `search_metadata` with `find_documents_making_movements_into_register`. |
| `resolve_qualified_name` | `search_metadata` with `resolve_qn`. |
| `find_by_guid` | `search_metadata` with `find_by_guid`. |
| `business_search` | `search_metadata_by_description`; use natural-language `search_metadata` only when a structural answer is needed and template operations are insufficient. |
| `answer_metadata_question` | Natural-language `search_metadata` for structural questions, `search_metadata_by_description` for discovery, then verify important facts with template operations. |
| `execute_metadata_cypher` / `get_metadata_prompt` | No documented direct replacement. Use documented templates first; if a needed query is not covered, rely on the live tool schema or fall back to `1c-code-metadata-mcp` / project source search per the fallback chain. |
| `compare_base_and_extension` | No documented direct replacement. Compare focused `object_structure`, forms, modules, and routines through `search_metadata` / `search_code`, or use extension-specific project tools when available. |
