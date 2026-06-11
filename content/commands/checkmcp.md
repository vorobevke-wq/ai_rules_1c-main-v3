---
description: Check availability of 1C MCP servers and install/start the missing ones
---

# /checkmcp — check and install 1C MCP servers

This command checks that all MCP servers from the project catalog (`content/mcp-servers.json`; after 1c-rules installation, rendered into the active tool config such as `.cursor/mcp.json` / `.mcp.json` / `.kilo/kilo.json` / `opencode.json` / `.codex/config.toml`) are actually available in the current session, and helps start or install missing ones. For Kilo Code the rendered file uses the top-level `mcp` key with per-server `{ "type": "remote", "url": "...", "headers": { ... }, "enabled": true }` when headers are needed — **not** the legacy `.kilocode/mcp.json` with `mcpServers` (current Kilo CLI / Kilo Code v7.x+ does not read that file).

The source of truth for legacy 1C MCP Docker images, ports, and environment variables is [docs.onerpa.ru/mcp-servery-1c](https://docs.onerpa.ru/mcp-servery-1c) and [vibecoding1c.ru/mcp_server](https://vibecoding1c.ru/mcp_server). For `rlm-tools-bsl`, use the upstream repository: <https://github.com/Dach-Coin/rlm-tools-bsl>. For `1c-lsp-mcp-skill`, use the upstream repository: <https://github.com/fserg/1c-lsp-mcp-skill>. For `1c-syntax`, use the upstream repository: <https://github.com/Starik2005/1c-syntax-mcp>. For `onec-buddy-mcp`, use the upstream repository: <https://github.com/ROCTUP/1c-buddy>. For `1c-mcp-toolkit`, use the upstream repository: <https://github.com/ROCTUP/1c-mcp-toolkit>.

## Target server catalog

| id | Port | Docker image | Purpose | Requires data |
|---|---|---|---|---|
| `1c-lsp-diagnostics` | 9011 | native service / `lsp-skill-server` from `1c-lsp-mcp-skill` | BSL diagnostics (BSL Language Server) | Yes — configured `lsp-skill-server` project id in the active MCP config `x-project-id` header |
| `1c-templates-mcp` | 8004 | `comol/1c_templates_mcp:latest` | Templates and project memory (`remember`/`recall`) | No |
| `1c-ssl-mcp` | 8008 | `comol/mcp_ssl_server:latest` | BSP/SSL search | No (`SSL_VERSION`) |
| `1c-syntax` | local | native local Python / `1c-syntax-mcp` | 1C platform syntax reference (`search_syntax`, `get_function_info`, `suggest_completion`, `validate_syntax`) | Yes — installed 1C platform with `shcntx_ru.hbk`, 7z, and the configured local server path |
| `rlm-tools-bsl` | 9000 | native service / `rlm-tools-bsl` package | Token-efficient BSL source exploration (RLM sandbox, optional SQLite index) | Yes — 1C source directory or registered project |
| `1c-mcp-metacode` | 6001 | `roctup/1c-mcp-metacode` | Graph metadata/code search (Neo4j) | Yes — configuration report + code dump + Neo4j |
| `onec-buddy-mcp` | 6002 | user-managed service / `ROCTUP/1c-buddy` | 1C:Assistant, code check, platform docs, ITS | No (1C.ai token) |
| `1c-mcp-toolkit` | 6003 | user-managed processing / `MCP_Toolkit.epf` from `ROCTUP/1c-mcp-toolkit` | Live infobase access: query/code execution, metadata, event log, links, rights, screenshots, session helpers | Yes — Toolkit processing running in embedded-server mode or connected to the proxy; TOON response format enabled |

> Exact image names may differ by version. If `docker pull` fails with `manifest unknown`, check the current list at [docs.onerpa.ru/mcp-servery-1c/servery.md](https://docs.onerpa.ru/mcp-servery-1c/servery.md).

> `1c-mcp-toolkit` is user-managed and is usually started from the external processing `build/MCP_Toolkit.epf`. In embedded-server mode the processing itself exposes `http://localhost:6003/mcp`; in proxy mode the Python proxy exposes the same endpoint and the processing connects to it. Enable TOON response format in the processing/proxy settings before using it with these rules.

## Algorithm

### Step 1. Determine the server set

1. If the project has `.ai-rules.json`, take the catalog from the active tool config referenced by the manifest (`.cursor/mcp.json` / `.mcp.json` / `.kilo/kilo.json` under the `mcp` key / `opencode.json` under the `mcp` key / `.codex/config.toml` under `[mcp_servers."<id>"]`). For the local `1c-syntax` server the rendered Codex block is the bare-key table `[mcp_servers.1c-syntax]` shown below. A leftover `.kilocode/mcp.json` is **legacy** — ignore it; current Kilo CLI / Kilo Code (v7.x+) does not read it. In `opencode.json` the server keys are letter-normalized to `onec-...` (e.g. `onec-lsp-diagnostics`, `onec-syntax`) because OpenCode names tools `<server-key>_<tool>` and providers like Moonshot/Kimi reject digit-leading function names — match them to the canonical `1c-...` ids by the bare tool names below, not by the prefix.
2. Otherwise use `content/mcp-servers.json` from the rules repository.
3. If neither source exists, use the table above as the default set.

### Step 2. Check availability in the current agent session

For each `id`, determine **TOOLS_OK** / **TOOLS_MISSING**:

- **TOOLS_OK** — this server's tools are visible in the current session tool schema (for example, `diagnostics` for `1c-lsp-diagnostics`, `templatesearch`/`recall` for `1c-templates-mcp`, `ssl_search` for `1c-ssl-mcp`, `search_syntax`/`get_function_info` for `1c-syntax`, `rlm_start`/`rlm_execute` for `rlm-tools-bsl`, `search_metadata`/`search_code` for `1c-mcp-metacode`, `check_1c_code`/`search_its` for `onec-buddy-mcp`, `execute_query`/`execute_code`/`get_event_log` for `1c-mcp-toolkit`).
- **TOOLS_MISSING** — no tools are visible in the schema.

If status is **TOOLS_OK**, treat the server as working and do not check it further.

### Step 3. Check HTTP endpoint

For servers with **TOOLS_MISSING**, call the HTTP endpoint. PowerShell (Windows):

```powershell
$servers = @(
    @{ Id = 'rlm-tools-bsl';          Port = 9000 },
    @{ Id = '1c-lsp-diagnostics';     Port = 9011 },
    @{ Id = '1c-templates-mcp';       Port = 8004 },
    @{ Id = '1c-mcp-metacode';        Port = 6001 },
    @{ Id = 'onec-buddy-mcp';         Port = 6002 },
    @{ Id = '1c-mcp-toolkit';         Port = 6003 },
    @{ Id = '1c-ssl-mcp';             Port = 8008 }
)
foreach ($s in $servers) {
    $url = "http://localhost:$($s.Port)/mcp"
    try {
        $r = Invoke-WebRequest -Uri $url -Method Get -TimeoutSec 3 -UseBasicParsing -ErrorAction Stop
        Write-Host ("{0,-26} {1,-5} HTTP {2}" -f $s.Id, $s.Port, $r.StatusCode)
    } catch {
        $code = if ($_.Exception.Response) { [int]$_.Exception.Response.StatusCode } else { 'down' }
        Write-Host ("{0,-26} {1,-5} {2}" -f $s.Id, $s.Port, $code)
    }
}
```

Any HTTP response (even `405`/`400`/`406`) means an MCP endpoint is listening on the port — status **HTTP_OK**. Full timeout / `Connection refused` means **HTTP_DOWN**.

For `1c-syntax` (local stdio server), there is no HTTP endpoint to probe. If its tools are missing, check that the active MCP config contains the expected local command and that both paths exist:

```powershell
$python = 'C:/Users/Lenovo PC/1c-syntax-mcp-master/venv/Scripts/python.exe'
$server = 'C:/Users/Lenovo PC/1c-syntax-mcp-master/server.py'
foreach ($path in @($python, $server)) {
    if (Test-Path -LiteralPath $path) {
        Write-Host "OK      $path"
    } else {
        Write-Host "MISSING $path"
    }
}
```

Copy-ready MCP entry for the common `mcpServers` shape:

```json
{
  "mcpServers": {
    "1c-syntax": {
      "command": "C:/Users/Lenovo PC/1c-syntax-mcp-master/venv/Scripts/python.exe",
      "args": ["C:/Users/Lenovo PC/1c-syntax-mcp-master/server.py"]
    }
  }
}
```

Codex entry:

```toml
[mcp_servers.1c-syntax]
enabled = true
command = "C:/Users/Lenovo PC/1c-syntax-mcp-master/venv/Scripts/python.exe"
args = ["C:/Users/Lenovo PC/1c-syntax-mcp-master/server.py"]
cwd = "C:/Users/Lenovo PC/1c-syntax-mcp-master"
env = { PYTHONIOENCODING = "utf-8", PYTHONUTF8 = "1" }
startup_timeout_sec = 60
tool_timeout_sec = 60
required = true
enabled_tools = ["search_syntax", "get_function_info", "suggest_completion", "validate_syntax"]
default_tools_approval_mode = "approve"
```

For `onec-buddy-mcp`, there is no installer-managed Docker start in this command. If its tools are missing or the endpoint is down, verify that the upstream 1C Buddy service is running at `http://localhost:6002/mcp`, then restart the client so the MCP session is rebuilt.

For `1c-mcp-toolkit`, there is no installer-managed Docker start in this command. If its tools are missing or the endpoint is down, verify that `MCP_Toolkit.epf` is opened in 1C, embedded-server mode is running on port `6003` or the Python proxy is running and connected, and TOON response format is enabled. Then restart the client so the MCP session is rebuilt.

### Step 4. Check Docker state

If at least one Docker-based server is **HTTP_DOWN**:

```powershell
docker version --format '{{.Server.Version}}'
docker ps --all --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
```

Possible outcomes:

- `docker version` fails with an engine connection error → **DOCKER_DOWN** (Docker Desktop is not running). Ask the user to start Docker Desktop and repeat `/checkmcp`.
- The container is visible in `docker ps -a`, but its state is `Exited` → **CONTAINER_STOPPED**. Start it:

  ```powershell
  docker start <container_name>
  ```

  Default names: `1c_templates_mcp`, `mcp_ssl_server`, `1c-metacode-<METACODE_PROJECT_ID>` (check the actual name in `docker ps -a`). `rlm-tools-bsl`, `1c-syntax`, `1c-lsp-mcp-skill`, `onec-buddy-mcp`, and `1c-mcp-toolkit` are usually user-managed native services / local processes / 1C processings / containers; check them through upstream service scripts or the Toolkit processing before assuming Docker is involved.

- The container is absent from `docker ps -a` → **CONTAINER_MISSING**. The image may already be cached (`docker images`), but the container was not created. Create and start it — see Step 5.

### Step 5. Install missing server

**Do not install or start services silently.** For Docker-based servers, do not run `docker run` without user confirmation. For `rlm-tools-bsl` or `1c-lsp-mcp-skill`, do not download / run the upstream installer or binaries without user confirmation.
`onec-buddy-mcp` is user-managed: follow the upstream repository instructions and expose `http://localhost:6002/mcp`; do not try to auto-install it from this command.
`1c-mcp-toolkit` is user-managed: open/configure `MCP_Toolkit.epf` or start its Python proxy and expose `http://localhost:6003/mcp`; do not try to auto-install it from this command.

- `LICENSE_KEY` — shared MCP server license key.
- Local data paths for servers that need them:
  - `1c-syntax` — local server path (`C:/Users/Lenovo PC/1c-syntax-mcp-master/server.py`), virtual-environment Python path (`C:/Users/Lenovo PC/1c-syntax-mcp-master/venv/Scripts/python.exe`), installed 1C platform containing `shcntx_ru.hbk`, and 7z for first-run extraction.
  - `rlm-tools-bsl` — 1C source directory (CF / EDT / MDO / extension source) or a registered project name; no shared `LICENSE_KEY` is required.
  - `1c-lsp-mcp-skill` — JVM, `bsl-language-server` JAR path, configured project in `lsp-skill-server`, and the `x-project-id` header configured manually in the active MCP client config for `1c-lsp-diagnostics`; no shared `LICENSE_KEY` is required.
  - `1c-mcp-metacode` — configuration report text file directory plus optional configuration-code dump directory, mounted into `/app/data/metadata` and `/app/data/code`.
  - `1c-ssl-mcp` — BSP/SSL version (`SSL_VERSION`, for example `3.1.11`).
  - `onec-buddy-mcp` — 1C.ai token and a running 1C Buddy service, if it will be used.
  - `1c-mcp-toolkit` — `MCP_Toolkit.epf` opened in the target infobase, embedded-server mode or proxy mode running on port `6003`, and TOON response format enabled.
- Index volume directory (`-v ...:/app/chroma_db`) — common folder such as `E:\bases\mcp_<id>`.

Command templates (minimal set without data preparation):

```powershell
# 1c-templates-mcp
docker run -d -p 8004:8004 --name 1c_templates_mcp `
  -e LICENSE_KEY={LICENSE_KEY} `
  -v "{DATA_ROOT}\mcp_templates:/app/chroma_db" `
  comol/1c_templates_mcp:latest

# 1c-ssl-mcp
docker run -d -p 8008:8008 --name mcp_ssl_server `
  -e LICENSE_KEY={LICENSE_KEY} `
  -e SSL_VERSION={SSL_VERSION} `
  -v "{DATA_ROOT}\mcp_ssl:/app/chroma_db" `
  comol/mcp_ssl_server:latest

# 1c-syntax — user-managed local server
# Active MCP config entry:
# {
#   "mcpServers": {
#     "1c-syntax": {
#       "command": "C:/Users/Lenovo PC/1c-syntax-mcp-master/venv/Scripts/python.exe",
#       "args": ["C:/Users/Lenovo PC/1c-syntax-mcp-master/server.py"]
#     }
#   }
# }

# rlm-tools-bsl — Windows PowerShell as Administrator
irm https://raw.githubusercontent.com/Dach-Coin/rlm-tools-bsl/master/simple-install-from-pip.ps1 -OutFile simple-install-from-pip.ps1
PowerShell -ExecutionPolicy Bypass -File .\simple-install-from-pip.ps1

# After installation, add / verify the MCP config entry:
# "rlm-tools-bsl": { "type": "http", "url": "http://127.0.0.1:9000/mcp" }

# Linux
curl -LO https://raw.githubusercontent.com/Dach-Coin/rlm-tools-bsl/master/simple-install-from-pip.sh
chmod +x simple-install-from-pip.sh
./simple-install-from-pip.sh

# 1c-lsp-mcp-skill — install by upstream release / quickstart
# 1. Install Java/JVM and download bsl-language-server JAR.
# 2. Start lsp-skill-server and configure the JAR path.
# 3. Add the 1C project in the web UI and enable MCP.
# 4. Put the project id into the active MCP client config:
#    "1c-lsp-diagnostics": { "type": "http", "url": "http://127.0.0.1:9011/mcp", "headers": { "x-project-id": "<project-id>" } }

# 1c-mcp-metacode — separate Neo4j + application Compose setup, see docs
# https://github.com/ROCTUP/1c-mcp-metacode

# onec-buddy-mcp — user-managed 1C Buddy service
# Install and start it by upstream instructions:
# https://github.com/ROCTUP/1c-buddy
# Expected MCP endpoint in this rules catalog:
# "onec-buddy-mcp": { "url": "http://localhost:6002/mcp", "connection_id": "1c_buddy_service_001" }

# 1c-mcp-toolkit — user-managed Toolkit processing
# 1. Open build/MCP_Toolkit.epf in 1C:Enterprise.
# 2. Start embedded-server mode on port 6003, or connect the processing to the Python proxy.
# 3. Enable TOON response format in the processing/proxy settings.
# Expected MCP endpoint in this rules catalog:
# "1c-mcp-toolkit": { "url": "http://localhost:6003/mcp", "connection_id": "1c_mcp_toolkit_service_001" }
```

Exact current commands for each server are on the server-specific documentation page:

- [1c-syntax-mcp](https://github.com/Starik2005/1c-syntax-mcp)
- [rlm-tools-bsl](https://github.com/Dach-Coin/rlm-tools-bsl)
- [1c-lsp-mcp-skill](https://github.com/fserg/1c-lsp-mcp-skill)
- [Graph Metadata Search](https://docs.onerpa.ru/mcp-servery-1c/servery/graph-metadata-search.md)
- [SSLSearchServer](https://docs.onerpa.ru/mcp-servery-1c/servery/ssl-search-server.md)
- [TemplatesSearchServer](https://docs.onerpa.ru/mcp-servery-1c/servery/templates-search-server.md)
- [1C Buddy](https://github.com/ROCTUP/1c-buddy)
- [1C MCP Toolkit](https://github.com/ROCTUP/1c-mcp-toolkit)

### Step 6. After install/start

1. Wait 5-15 seconds (Docker containers and native services need warm-up; indexed servers may need tens of minutes or hours on first indexing. For Docker, monitor `docker logs -f <name>`; for `rlm-tools-bsl`, use `rlm_index(action="info", project="...")` or the service log).
2. Repeat Step 3 (HTTP check); all statuses should become **HTTP_OK**.
3. If the server is absent from the active tool MCP config, add the entry (1c-rules installer should already have rendered it; if installation was not run, add it manually using `adapters/<tool>.yaml → mcp.schema`).
4. Restart the client (Cursor / Claude Code / Codex / OpenCode / Kilo Code) so it reinitializes the MCP session.
5. Run `/checkmcp` again; Step 2 statuses should become **TOOLS_OK**.

## Final report

Summary table for the user:

| Server | Session tools | HTTP | Container | Action |
|---|---|---|---|---|
| `...` | OK / missing | OK / down | running / stopped / missing | none / `docker start` / `docker run` / reconnect client |

For non-Docker servers (`rlm-tools-bsl`, `1c-mcp-toolkit`), use `Service / endpoint` instead of `Container` and report the relevant service, processing, proxy, or endpoint action.

Under the table, list clear next steps with copy-ready commands. Do not list items that already work.

## Limits

- The command does not run `docker run` or upstream service installers without user confirmation. Docker servers need `LICENSE_KEY`, data paths, and consent to download images (several GB). `rlm-tools-bsl` does not use the shared `LICENSE_KEY`, but installing it still downloads and registers a local service.
- Metacode MCP (`1c-mcp-metacode`) requires separate Neo4j setup and indexing. This is a multi-step process; execute it by the GitHub repository instructions, not from this command.
- Indexed servers (`1c-syntax` while it builds `syntax_tree.json`, `1c-mcp-metacode`, `1c-ssl-mcp`, `rlm-tools-bsl` when its SQLite index is building, and `1c-lsp-mcp-skill` while `bsl-language-server` is warming up) may be configured before becoming useful while primary indexing is still running. This is normal; monitor Docker services with `docker logs -f <name>`, `1c-syntax` through the MCP client/server stderr logs, `rlm-tools-bsl` through `rlm_index(action="info", project="...")` / service logs, and `1c-lsp-mcp-skill` through its web UI / service logs.
