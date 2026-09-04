# 1c-lsp-mcp-skill — MCP tool catalog

`1c-lsp-mcp-skill` is the upstream package that runs `bsl-language-server` for indexed 1C projects. This ruleset uses its diagnostics MCP server as the BSL validation gate.
Upstream: <https://github.com/fserg/1c-lsp-mcp-skill>.

| MCP server id | Default endpoint | Purpose |
|---|---:|---|
| **1c-lsp-diagnostics** | `http://127.0.0.1:9011/mcp` | Static BSL diagnostics: syntax errors, warnings, analyzer remarks |

> Load this file only if the `diagnostics` tool from `1c-lsp-diagnostics` is actually available in the current session.

## Connection and paths

- Every MCP call requires the HTTP header `x-project-id` with the project id from `lsp-skill-server`.
- `content/mcp-servers.json` stores this header as the `<project-id>` placeholder. After installing rules, replace it in the active MCP client config for `1c-lsp-diagnostics` with the real project id from `lsp-skill-server`.
- All `file_path` arguments are paths to `.bsl` files **relative to `project_root_path`**, not absolute filesystem paths and not raw BSL text.
- Preserve Cyrillic directory names exactly.
- LSP coordinates are zero-based: `line` starts at `0`, `character` starts at `0`.

## Diagnostics server

| Tool | Parameters | Purpose | When to use |
|---|---|---|---|
| **diagnostics** | `file_path` | Run static analysis for one `.bsl` file through `bsl-language-server`; returns LSP diagnostics with `range`, `severity`, `message`, `source`, `code` | Before edits for a baseline, after BSL edits to verify the file, and when explaining analyzer errors |

### Diagnostics rules

- Pass a relative `.bsl` file path. Do not pass BSL source text.
- The first request to a file may take longer because the file is opened in the LSP session.
- Empty output is not absolute proof that the file is clean; indexing may still be in progress or diagnostics may be partial during `warming_up`.
- Re-run `diagnostics` only after an actual file edit or when the previous run reported a substantive defect that was fixed.

## Optional navigation sibling

Upstream also exposes a separate `1c-lsp-navigation` server at `http://127.0.0.1:9012/mcp`. It is **not** part of the default `content/mcp-servers.json` catalog because this ruleset already routes code navigation and impact analysis through `1c-mcp-metacode` and `rlm-tools-bsl`.

If a project explicitly configures that optional sibling, its tools are:

Prefer these tools over text search when symbol-aware navigation is available.

| Tool | Parameters | Purpose |
|---|---|---|
| **symbols** | `file_path` | Return module structure: procedures, functions, variables, and regions with ranges and hierarchy |
| **definition** | `file_path`, `line`, `character` | Find the declaration / definition of a symbol at a precise position |
| **references** | `file_path`, `line`, `character` | Find all project usages of a symbol, including the declaration |
| **incoming_calls** | `file_path`, `line`, `character` | Find direct callers of the procedure or function at the position |
| **outgoing_calls** | `file_path`, `line`, `character` | Find direct callees used from the procedure or function at the position |
| **workspace_symbols** | `query` | Search project symbols by exact name or distinctive fragment |

### Navigation rules

- Check the exact character position before calling position-based tools; off-by-one errors often return `null` or irrelevant results.
- Treat missing results as ambiguous, not final. Common causes: indexing in progress, wrong relative path, wrong coordinates, or dynamic dispatch.
- Use text search only as a secondary check or to inspect context around symbols already found through LSP.

## Response format

By default, tools return standard LSP JSON. If `use_toon_format` is enabled in `lsp-skill-server`, responses use compact TOON aliases:

- `range` -> `range_sl`, `range_sc`, `range_el`, `range_ec`
- `selectionRange` -> `selection_range_sl`, `selection_range_sc`, `selection_range_el`, `selection_range_ec`
- `location` -> `location_uri`, `location_sl`, `location_sc`, `location_el`, `location_ec`
- `targetUri` -> `target_uri`
- `containerName` -> `container_name`
- call hierarchy `from` / `to` fields are inlined with `from_` / `to_` prefixes

## Verification budget

- `diagnostics` follows the same validator budget as other BSL gates: 1 call per validator per cycle by default; up to 3 only when the previous run returned a substantive defect.
- A cycle is one logical edit of one module.
- Style-only warnings, naming nits, formatting issues, or analyzer noise do not justify repeated runs.
- `check_1c_code` remains the AI-assisted validator from `onec-buddy-mcp`; run it after `diagnostics` when BSL validation is required by `AGENTS.md`.
