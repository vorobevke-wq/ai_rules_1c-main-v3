---
description: MCP-first search discipline — explicit priority of MCP project-index tools over Grep / Glob, with a mandatory "what was tried" note before any fallback. Load before any code / metadata / usage search in a 1C project.
alwaysApply: false
category: tooling
---

# MCP-first search discipline

For any 1C **project-source search** (code, metadata, usages, call chains, structure, forms, layouts) — MCP project-index tools come **first**. `Grep` / `Glob` are the **last resort**, gated by an explicit justification note.

Applies to every subagent except `1c-explorer`, which already encodes the same rule in its own prompt. The canonical fallback chain owner is `content/skills/mcp-1c-tools/SKILL.md → Fallback chain → Project-source search before Grep / rg`. This file does not redefine it — it makes the rule salient inside subagent prompts that previously only had a soft pointer.

---

## Hard rule

1. **Before any `Grep` / `Glob` call on project source**, you MUST first exhaust the project-index path:
   1. `1c-mcp-metacode` — `search_code`, `search_metadata`, `search_metadata_by_description` as applicable. Express structure, usage, impact, movements, and call-graph questions through `search_metadata` JSON template operations.
   2. `rlm-tools-bsl` — open an RLM session (`rlm_start`; `rlm_help` for non-trivial tasks), then batch helper calls inside `rlm_execute`: `search`, `search_objects`, `search_methods`, `find_definition`, `get_module_outline`, `find_module`, `get_object_full_structure`, `parse_form`, `find_call_hierarchy`, `find_path`, `find_data_path`, `find_references_to_object`, `find_code_usages`, and related helpers.
   3. `rlm-tools-bsl` literal / narrowed retry — **only** after step 2 returned not enough: use `git_search` for repository-wide exact text / regex over git-backed source trees (`exclude_path` can suppress noisy zones such as `Forms` / `Templates`), or `safe_grep` / `grep_read` on narrowed modules and paths. Typical triggers: exact identifier, fragment of a query, metadata path, event-handler name, error text, literal string.
2. **Only then `Grep` / `Glob`** — and only when you can state, in one or two sentences inside the response, **which MCP attempts were tried and why they did not return what was needed**. Silent fallback to `Grep` / `Glob` is a defect.
3. **Tune the query before re-calling.** If the first MCP call returned nothing, do **not** immediately fall through to the next tool — reformulate: broaden / narrow the query, switch `search_type` (`fulltext` ↔ `semantic` ↔ `hybrid`), adjust `detail_level`, lower `exact`, raise `top_k`, drop or change `project_name` / category filters. Use the per-server parameter docs in `content/skills/mcp-1c-tools/docs/<server>.md`.
4. **No-change repeats are forbidden.** Do not re-run the same MCP call against the same unchanged state. A new call must change parameters substantively, or the project state must have changed (file edit, new generation, resumed session).

External-knowledge, validation, and live-IB servers (`1c-templates-mcp`, `1c-syntax`, `onec-buddy-mcp`, `1c-lsp-diagnostics`, `1c-mcp-toolkit`) have **no `Grep` / `rg` equivalent** — they are called only when their knowledge is needed, not as part of the fallback above. БСП reference is skill-based: use locally installed skills whose folder names contain `bsp-` when the task needs БСП APIs or patterns.

---

## Quick first-pick table

| Need | First call (MCP) | If empty — next |
|---|---|---|
| Find BSL code by behaviour / description | `search_code(query)` | `rlm_execute`: `search` / `search_methods` / `find_definition`; if literal, `git_search` |
| Find BSL code by exact identifier / literal | `search_code(query)` | `rlm_execute`: `find_definition` / `git_search` → narrowed `safe_grep` / `grep_read` → only then `Grep` |
| Find a routine by name | `search_metadata` (`find_routines_by_name`) | `rlm_execute`: `find_definition` → `search_methods` → `find_module` + `extract_procedures` |
| Understand a metadata object | `search_metadata` (`object_structure` plus focused facet templates) | `rlm_execute`: `get_object_full_structure` / `parse_object_xml` |
| Metadata search by name / structure | `search_metadata` (JSON template) | `rlm_execute`: `search_objects`, `find_by_type`, `find_attributes` |
| Metadata search by Russian description / synonym | `search_metadata_by_description` | `rlm_execute`: `search_objects`, `search`, `git_search` over XML/help text |
| Usages of an object | `search_metadata` (`find_usages_of_object` / `find_objects_using_object` template operations) | `rlm_execute`: `find_references_to_object`, `find_code_usages` |
| Impact of a change | `search_metadata` usage / movement / call-graph templates (`find_objects_using_object`, `find_usages_of_object`, `find_documents_making_movements_into_register`, `call_graph_subtree`) | `rlm_execute`: references, code usages, data path, register movements, call hierarchy helpers |
| Call graph (who calls / who is called) | `search_metadata` (`list_callers_of_routine`, `list_callees_of_routine`, `call_graph_subtree`) | `rlm_execute`: `find_call_hierarchy` / `find_callers_context` / `find_path`; add `include_triggers=True` when non-call triggers matter |
| Module structure overview | `rlm_execute`: `find_module` + `get_module_outline` + `extract_procedures` + `code_metrics` | `read_file` / `read_procedure` on narrowed paths |
| Form layout | `rlm_execute`: `parse_form(object_name, form_name="")` | `git_search` over form XML or `read_file` on narrowed form paths |
| Canonical pattern / template | `templatesearch(query)` (+ installed `bsp-*` skills for БСП) | — |
| Platform API verification | `get_function_info(name)` or `search_syntax(query)` | `search_1c_documentation` when exposed |
| ITS standards | `search_its(query)` → `fetch_its(id)` for **every** relevant doc | — |

`Grep` / `Glob` are absent from this table on purpose — they are not a first pick for any of these needs.

---

## When `Grep` / `Glob` are legitimately the right tool

The MCP-first rule applies to **1C project-source search**. `Grep` / `Glob` are appropriate, with no need for an MCP attempt first, when the target is **outside the MCP index**:

- non-BSL / non-metadata files: `.md` documentation, `.json` / `.yaml` configs, slash-command sources, rule files, `openspec/` artifacts, deployment logs;
- text fixtures, sample payloads, or generated reports under `handoffs/`, `dist/`, build output;
- a file you have already read in this session and are scanning for a literal string locally.

In all 1C project-source cases — follow the hard rule above.

---

## Response gate

Before delivering a result that involved `Grep` / `Glob` on project source, include a short line in the response, e.g.:

> *Tried `search_code(query="...")` (empty), then `rlm-tools-bsl` helpers `search_methods(...)` and `git_search(...)` (no match); fell back to `Grep` for the literal `<...>`.*

One or two sentences. No bullet list of every parameter tried.

---

## Success criteria

- ✅ MCP project-index path attempted before any `Grep` / `Glob` call on 1C project source.
- ✅ Each failed MCP call closed a concrete context gap before the next call (no blind chaining, no "just to be safe").
- ✅ `Grep` / `Glob` usage on project source is justified inline.
- ✅ No duplicated calls against unchanged state.
