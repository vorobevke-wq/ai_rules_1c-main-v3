---
name: mcp-1c-tools
description: "Catalog of MCP servers for 1C development — search, code navigation, metadata, diagnostics, code review, docs, ITS, templates. Use whenever a 1C task requires calling tools from any 1C MCP server. Each server has its own detail file under `docs/` — load it when you are about to call tools from that server, and only if the server is actually available in the current session."
---

# MCP tools for 1C — dispatcher

This skill is the single source of truth for the project's MCP server catalog, task→tool mapping, fallback order, and project-index search retries. Detailed per-tool descriptions for each server live in separate files under `docs/`. **Load a specific `docs/<server>.md` when you are about to call tools from that server and want to tune parameters; the server must be actually available in the current session** (its tools are exposed in the tool schema; the mere presence of an entry in `mcp-servers.json` does not count as availability).

## What is mandatory vs. conditional

- **Mandatory for risk-bearing 1C work.** If a relevant server is exposed, call the fitting MCP tool for BSL / metadata edits or review, metadata XML, forms, integrations, refactoring, performance, runtime errors, platform API checks, impact analysis, syntax / quality validation, and project-memory operations.
- **Conditional for external knowledge.** Use platform docs, БСП / SSL, and ITS MCP tools when the task depends on versioned platform behavior, reusable БСП APIs, or standards compliance. Do not call them for generic prose cleanup or rule-file editing unless such a fact is actually needed.
- **Not required for Markdown / rules / documentation-only work.** For rule files, README, commands documentation, and similar prose-only edits, validate structure, links, paths, and internal consistency instead of calling 1C project MCP tools.
- **Recommended: reading `docs/<server>.md` before parameter-rich calls.** Reading the schema is for parameter tuning, not a hard gate. Skipping it is acceptable only when the call is genuinely simple (a one-shot lookup with obvious arguments) and you are not invoking a parameter-rich tool listed below.

### Parameter-rich tools — read the doc first

For these tools default parameters are usually suboptimal; consult the server's `docs/<server>.md` before the first call in the session and adjust the parameters to the task:

- `1c-mcp-metacode`: `search_metadata` (JSON template operations and natural-language mode), `search_metadata_by_description`, `search_code`. Use `docs/1c-mcp-metacode.md` to choose the right template operation before calling.
- `rlm-tools-bsl`: `rlm_start`, `rlm_help`, `rlm_execute`, `rlm_projects`, `rlm_index`; inside `rlm_execute`, tune helper choice and arguments (`search`, `search_objects`, `search_methods`, `find_definition`, `get_module_outline`, `git_search(..., exclude_path=...)`, `get_object_full_structure`, `parse_form`, `find_call_hierarchy(..., include_triggers=...)`, `find_path`, `find_data_path`, `find_references_to_object`, etc.).
- `1c-lsp-diagnostics`: `diagnostics` (path-based BSL validation); use `docs/1c-lsp-mcp-skill.md` to confirm relative path rules.
- `1c-mcp-toolkit`: `execute_query`, `execute_code`, `get_metadata`, `get_event_log`, `get_object_by_link`, `get_link_of_object`, `find_references_to_object`, `get_access_rights`, `get_bsl_syntax_help`, `get_screenshot`; use `docs/1c-mcp-toolkit.md` to confirm parameters, TOON output assumptions, and live-IB safety rules.

If `docs/<server>.md` conflicts with the descriptor exposed by the current environment, the environment descriptor wins.

## When to use this skill

- Before writing code / a query / metadata XML — pick the MCP tool that best fits the task (template search, metadata check, syntax validation, code review).
- For impact analysis and code navigation — decide which server to use first (`graph` → `rlm-tools-bsl` → `Grep` — see *Fallback chain* below).
- For ITS standards (`search_its` → `fetch_its`) and platform syntax reference (`search_syntax` / `get_function_info` / `suggest_completion` / `validate_syntax`).
- For 1C code templates (`templatesearch`) and read-only template browsing tools when explicitly needed.

> Short obligation rules and verification budgets live in `AGENTS.md → MCP Tool Calling` (sections A, B, C). This skill owns the MCP catalog, routing, and fallback details.

## Server catalog

| Server (id) | Purpose | Details |
|---|---|---|
| **1c-mcp-metacode** | Graph metadata and BSL code search (Neo4j): structure, usage, call graph, forms, rights, routines, semantic metadata/code search. Default id; per-project MCP keys may be suffixed when installing multiple Metacode project IDs, as long as the same tools are exposed. | [`docs/1c-mcp-metacode.md`](docs/1c-mcp-metacode.md) |
| **rlm-tools-bsl** | Intent-first 1C BSL repository search and navigation through an RLM sandbox: project registry, optional SQLite index, metadata/code/form/usages/call-graph/data-path helpers | [`docs/rlm-tools-bsl.md`](docs/rlm-tools-bsl.md) |
| **1c-templates-mcp** | 1C BSL code-template library: semantic search and read-only template browsing tools | [`docs/1c-templates-mcp.md`](docs/1c-templates-mcp.md) |
| **1c-syntax** | 1C platform syntax reference from the local Syntax Assistant (`shcntx_ru.hbk`): functions, methods, completions, call syntax checks | [`docs/1c-syntax-mcp.md`](docs/1c-syntax-mcp.md) |
| **onec-buddy-mcp** | 1С:Напарник via 1C Buddy — AI advice, syntax explanation, code check / targeted modify, platform documentation, ITS documentation | [`docs/onec-buddy-mcp.md`](docs/onec-buddy-mcp.md) |
| **1c-lsp-diagnostics** | BSL diagnostics via `1c-lsp-mcp-skill` / BSL Language Server | [`docs/1c-lsp-mcp-skill.md`](docs/1c-lsp-mcp-skill.md) |
| **1c-mcp-toolkit** | Live-IB access through ROCTUP/1c-mcp-toolkit in TOON mode: query/code execution, metadata, event log, object links, references, access rights, screenshots and session helpers | [`docs/1c-mcp-toolkit.md`](docs/1c-mcp-toolkit.md) |

## Fallback chain (highest priority to lowest)

Use only the applicable branch; stop as soon as the collected evidence is sufficient. Before each call, check that it closes a concrete context gap and is not a duplicate of an earlier call.

### Project-source search before `Grep` / `rg`

`Grep` / `rg` substitute only the project-indexing layer. Before falling back to them for 1C project-source search, exhaust:

1. `1c-mcp-metacode` — `search_code`, `search_metadata`, `search_metadata_by_description` as appropriate. Express structural, usage, impact, and call-graph questions through `search_metadata` JSON template operations whenever possible.
2. `rlm-tools-bsl` — session-based repository exploration. Start with `rlm_start`, consult `rlm_help` for non-trivial analysis, then batch helper calls inside `rlm_execute` (`search`, `search_objects`, `search_methods`, `find_definition`, `get_module_outline`, `find_module`, `get_object_full_structure`, `parse_form`, `find_call_hierarchy`, `find_path`, `find_data_path`, `find_references_to_object`, etc.).
3. Literal / substring retry inside `rlm-tools-bsl` — use `git_search` for repository-wide literal or regex search when the source tree is git-backed; use `exclude_path` to suppress noisy raw XML zones such as `Forms` / `Templates` when needed; use `safe_grep` or `grep_read` only on narrowed modules / paths. Typical scenarios: exact identifier, fragment of a query, metadata path, event handler name, error text, or literal string where semantic / indexed search is likely to miss.
4. Only then `Grep` / `rg` — with a mandatory short note in the response listing which project-index MCP attempts were tried and why they did not return what was needed.

### External knowledge

These servers have no `Grep` / `rg` equivalent; call them only when their knowledge is needed:

1. `1c-templates-mcp` — 1C code templates (`templatesearch`; read-only browsing tools only when exact template contents are needed).
2. `1c-syntax` — local platform syntax reference for exact function / method names and call syntax.
3. `onec-buddy-mcp` — 1С:Напарник checks, ITS standards (`search_its` → `fetch_its` for every document used), AI drafts.
4. `1c-lsp-diagnostics` — BSL syntax / analyzer diagnostics after edits.
5. `1c-mcp-toolkit` — live infobase access through Toolkit processing / proxy: run a query, run a read-only BSL fragment, emulate query parse-checking through `execute_code`, inspect event-log records, metadata, links, references and access rights. No `Grep` / `rg` equivalent — there is no offline substitute for "what does this running IB do right now". Call only when the question genuinely requires the live IB; default to read-only fragments and ask before any mutation. Details — [`docs/1c-mcp-toolkit.md`](docs/1c-mcp-toolkit.md).

## Quick map: "task → MCP tool"

| Task | First choice (metacode) | Fallback (`rlm-tools-bsl`) |
|---|---|---|
| BSL code search | `search_code` | `rlm_execute`: `search`, `search_methods`, `find_definition`, `git_search`, `safe_grep` / `grep_read` on narrowed paths |
| Metadata object structure | `search_metadata` (`object_structure` plus focused facet templates) | `rlm_execute`: `get_object_full_structure` or `parse_object_xml` |
| Impact analysis before refactoring | `search_metadata` usage / movement / call-graph templates (`find_objects_using_object`, `find_usages_of_object`, `find_documents_making_movements_into_register`, `call_graph_subtree`) | `rlm_execute`: `find_references_to_object`, `find_code_usages`, `find_data_path`, `find_register_movements`, `find_register_writers`, `analyze_document_flow` |
| Call graph | `search_metadata` (`list_callers_of_routine`, `list_callees_of_routine`, `call_graph_subtree`) | `rlm_execute`: `find_call_hierarchy` / `find_callers_context`; use `find_path` for routine-to-routine reachability and `include_triggers=True` when event subscriptions, form events, scheduled jobs, or CFE overrides matter |
| Metadata search by name / structure | `search_metadata` (JSON templates) | `rlm_execute`: `search_objects`, `find_by_type`, `find_attributes`, `search(scope="objects"|"attributes")` |
| Object usage search | `search_metadata` (`find_objects_using_object` / `find_usages_of_object` template operations) | `rlm_execute`: `find_references_to_object`, `find_code_usages` |
| Description / synonym / comment search | `search_metadata_by_description` | `rlm_execute`: `search_objects`, `search`, `git_search` over raw XML/help text |

Step-by-step playbooks per task type (writing code, review, architecture, error fixing, performance, refactoring, metadata XML, forms, integrations, documentation, comparing platform versions) — `content/rules/tooling-playbooks.md`.
