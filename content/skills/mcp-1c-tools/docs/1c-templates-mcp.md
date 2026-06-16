# 1c-templates-mcp — tool catalog

1C BSL code-template library from [`Desko77/1c-templates-mcp`](https://github.com/Desko77/1c-templates-mcp). The server provides semantic + full-text template search, a web UI, and MCP tools for template search and read-only browsing. It does **not** provide project-memory tools.

> Load this file only if the `1c-templates-mcp` server is actually available in the current session.

| Tool | Parameters | Purpose | When to use |
|---|---|---|---|
| **templatesearch** | `query: str` | Hybrid semantic + full-text search over 1C BSL code templates | Find architectural patterns and implementation examples **before** writing code |
| **list_templates** | `offset?`, `limit?` | List templates with pagination; default limit is 50, maximum is 200 | Browse the template library or inspect nearby candidates after search |
| **get_template** | `template_id: int` | Return the full template, including code, by template ID | Read a specific search/list result before reusing it |

## Usage notes

- Use `templatesearch` for ordinary development, review, architecture, refactoring, performance, and integration tasks when reusable 1C patterns may exist.
- Use `get_template` before copying or adapting a concrete template; do not rely only on a search-result summary when exact code matters.
- Mutating template-library tools are intentionally not documented in this ruleset and must not be used by agents.
- No project-memory tools are exposed by this server. Project memory lives only in `memory.md`.

## Availability check

Treat the server as available when its MCP tools are exposed in the current session's tool schema, at minimum `templatesearch` for read-only template search. Exact template inspection requires `list_templates` and `get_template` to be visible.
