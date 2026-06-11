# onec-buddy-mcp — tool catalog

1C Buddy is a Streamable HTTP MCP gateway to 1С:Напарник. It exposes AI-assisted 1C advice, BSL code checks / targeted code modification, platform documentation search, ITS search, and documentation version comparison.

> Load this file only if the `onec-buddy-mcp` server is actually available in the current session.

Source repository: <https://github.com/ROCTUP/1c-buddy>. The documented MCP endpoint is `http://localhost:6002/mcp`.

## Code analysis & modification

| Tool | Parameters from upstream docs | Purpose | When to use |
|---|---|---|---|
| **ask_1c_ai** | `question`, optional `programming_language`, `ssl_version`, `configuration` | Free-form question to 1С:Напарник about the platform and practical 1C scenarios | Architectural questions, concept explanations, advice. **Non-deterministic — treat as a hint, not authority** |
| **explain_1c_syntax** | `syntax_element`, optional `context`, `ssl_version`, `configuration` | Explain a specific 1C object, method, or language construct | When `1c-syntax` found a name but broader explanation or usage context is needed |
| **check_1c_code** | `code`, optional `check_type` (`syntax`, `review`, `logic`, `performance`), optional `extended` | Syntax check or code review for a BSL fragment. `logic` / `performance` are accepted for compatibility and handled as `review` by the server | After writing code — find syntax, logic, performance, style, and ITS-compliance issues. Use `check_type="review"` when a style / standards review is needed |
| **modify_1c_code** | `instruction`, `code` | Modify 1C code by an explicit user instruction | Targeted fixes, specific bug fixes, feature additions. **Non-deterministic — mandatory re-validation** |

## Documentation & knowledge base

| Tool | Parameters from upstream docs | Purpose | When to use |
|---|---|---|---|
| **search_1c_documentation** | `query`, optional `version` (server default `v8.5.1`) | Search 1C:Enterprise platform documentation for a specific version | Verify method signatures in a specific version, version-specific platform features |
| **search_its** | `query`, optional `ssl_version`, `configuration` | Search the ITS knowledge base | Find standards, methodology, best practices. **Returns document IDs → use `fetch_its`** |
| **fetch_its** | `id` | Read the full content of an ITS document, directory, or section by ID | **Always after `search_its`** — read every relevant found article. Special IDs include `root`, `superior`, and `v8std` |
| **diff_1c_documentation_versions** | `version_a`, `version_b`, `query` | Compare platform documentation between two versions | Check behaviour or API changes between platform versions |

## Key ITS workflow

`search_its` → get document IDs → `fetch_its` for each relevant ID → read the full content. **Never rely on ITS search snippets without `fetch_its`.**

## Notes on AI tools

`ask_1c_ai` and `modify_1c_code` are non-deterministic. Their output is a draft hint, not authority. Generated / modified code is **always** re-validated: `diagnostics` + `check_1c_code`.

## Call limit

`check_1c_code` — **1 call per module-edit cycle by default**, up to 3 total **only** when the previous run returned a substantive defect (logic / metadata / data integrity / security / transaction / lock / performance-critical). Style warnings, naming nits, BSLLS noise do **not** justify a re-run — fix them in the edit and move on. Do not re-call when the code has not changed since the previous run. For pure metadata-XML changes with no BSL touched, this tool is usually irrelevant — skip in favor of the relevant `1c-metadata-manage` validator (`form-validate`, `meta-validate`, `role-validate`, `skd-validate`, etc.). Full policy: `AGENTS.md -> MCP Tool Calling -> B. Limits and non-determinism`.
