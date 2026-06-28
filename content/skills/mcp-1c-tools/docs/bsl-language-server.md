# bsl-language-server — MCP tool catalog

`bsl-language-server` is the native MCP mode of [1c-syntax/bsl-language-server](https://github.com/1c-syntax/bsl-language-server). This ruleset uses it as the BSL Language Server validation and local code-intelligence gate.

| MCP server id | Default endpoint | Purpose |
|---|---:|---|
| **bsl-language-server** | `http://localhost:32101/mcp` | Static BSL diagnostics and position-based code intelligence through BSL Language Server MCP mode |

> The endpoint port is a slot from `32101` to `32199` (`http://localhost:321**/mcp`, where `**` is `01..99`). The catalog uses `32101` as the default example. Replace it in the active MCP client config with the slot used by your manually started server.

## Connection and workspace

- Start BSL Language Server manually in MCP Streamable HTTP mode, for example `java -jar bsl-language-server.jar mcp --protocol streamable --server.port=32101`, or with `--mcp` beside LSP/websocket mode.
- Workspaces are provided through MCP roots; no separate project header from the MCP client is needed.
- File arguments use the parameter name `file` and accept a `.bsl` / `.os` path that is absolute or relative to the server working directory.
- Position arguments use zero-based LSP coordinates: `line` starts at `0`, `character` starts at `0`.
- Root-scoped type tools use `root`: the URI of one workspace root declared by the MCP client. Use the active project root when the answer depends on configuration metadata.

## Preferred tools for this ruleset

| Tool | Parameters | Purpose | When to use |
|---|---|---|---|
| **analyze_file** | `file` | Run BSL diagnostics for a single 1C/OneScript file and return the list of issues | Before edits for a baseline, after BSL edits to verify the file, and when explaining analyzer errors |
| **hover** | `file`, `line`, `character` | Return hover information for the symbol at a position: signature, inferred type, documentation | During review and code explanation when a precise symbol position is known |
| **type_info** | `typeName`, `fileType`, `root`, `language?` | Look up a BSL type by Russian or English name and return properties, methods, events and constructors with signatures and metadata | Before writing code that depends on exact members of a known platform / configuration / OneScript type |
| **type_at_position** | `file`, `line`, `character` | Infer expression type(s) under the cursor and return available methods and properties | Before writing or changing code when the receiver type or available members are uncertain |

### Preferred-tool rules

- Use `analyze_file` as the BSL Language Server validation tool throughout this ruleset.
- Use `hover` for review and explanations only when you have a concrete file position. It is a local code-intelligence hint, not an impact-analysis substitute.
- Use `type_at_position` before writing code when the expression receiver type controls the correct method/property call.
- Use `type_info` when the type name is already known and you need the available members or signatures.
- If a position-based tool returns no result, treat that as ambiguous: common causes are wrong coordinates, missing MCP roots, stale indexing, or dynamic dispatch.

## Other exposed tools

The upstream v1.0.2 MCP mode also exposes these tools. They are listed for general server awareness, but this ruleset does not route normal work to them unless a user explicitly asks or the live schema makes them the best fit:

| Tool | Purpose |
|---|---|
| **document_symbols** | Return the symbol tree of a file: regions, methods and variables |
| **find_references** | Find all references to the symbol at a zero-based file position |
| **call_hierarchy** | Resolve the method / procedure at a position and return direct incoming and outgoing calls |
| **definition** | Resolve the symbol at a position and return where it is declared |
| **global_member_info** | Look up a global function, property or system enum by exact Russian or English name |
| **global_member_search** | Search global context members with fuzzy matching, grouped by function / property / enum categories |

## Validation budget

- `analyze_file` follows the same validator budget as other BSL gates: 1 call per validator per cycle by default; up to 3 only when the previous run returned a substantive defect.
- A cycle is one logical edit of one module.
- Style-only warnings, naming nits, formatting issues, or analyzer noise do not justify repeated runs.
- `check_1c_code` remains the AI-assisted validator from `onec-buddy-mcp`; run it after `analyze_file` when BSL validation is required by `AGENTS.md`.

## Sources

- Upstream MCP mode documentation: <https://github.com/1c-syntax/bsl-language-server/blob/v1.0.2/docs/features/McpMode.md>
- Tool descriptions and parameters: `src/main/java/com/github/_1c_syntax/bsl/languageserver/mcp/tools/*Tool.java` in the same `v1.0.2` tag.