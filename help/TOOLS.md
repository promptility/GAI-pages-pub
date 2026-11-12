# HTTP tools and MCP

GAI can offer configured HTTP endpoint functions and discovered MCP functions to the model in one interaction. The model must support Chat Completions function calling. Select the servers to discover with `mcp_use` and the tools to advertise with `tool_use`; merely defining a server or HTTP tool does not select it.

A full combined configuration is [tools_and_mcp.json](https://github.com/promptility/GAI-pages-pub/blob/main/examples/tools_and_mcp.json). Replace its provider credential, external-interpreter paths, and demo-script path before using it.

## HTTP endpoint functions

A tool definition has a URL plus a nested Chat Completions function schema:

```json
{
  "tools": {
    "http_echo": {
      "location": "http://127.0.0.1:8765/echo",
      "type": "function",
      "function": {
        "name": "http_echo",
        "description": "Echo a message through a local service.",
        "parameters": {
          "type": "object",
          "properties": {
            "message": {"type": "string", "description": "Message to echo."}
          },
          "required": ["message"],
          "additionalProperties": false
        },
        "strict": true
      }
    }
  },
  "tool_use": ["http_echo"]
}
```

Merge these fields into a provider configuration; this fragment alone has no model endpoint. The validator requires `location`, `type`, and `function`; function `name`, `description`, and `parameters`; and parameter `type`, `properties`, `required`, and `additionalProperties`. Each handwritten property requires `type` and `description`. `strict` is optional and provider-dependent. Keep dictionary keys and function names identical for readability; filtering uses the dictionary key, while dispatch uses the advertised function name.

Run the supplied loopback service in a terminal:

```bash
python3 examples/http_tool_demo.py --port 8765
```

Select text such as “Use http_echo to repeat hello” and invoke Generate text. GAI strips `location` before advertising the schema, POSTs the model's JSON **object** arguments to that URL, and sends the returned body to the model under `tool_results`. Query strings are preserved. Invalid JSON/non-object arguments are rejected before a request. Non-2xx responses become tool errors; connections and response bodies are closed after each exchange.

HTTP tools send `Content-Type: application/json`; they do not inherit the model API key and do not support arbitrary authentication headers, URL Basic-auth credentials, or built-in tool locations such as `builtin`. Use MCP HTTP headers when an authenticated service requires them, or extend the endpoint transport explicitly. Remote endpoints should use HTTPS; the loopback demo intentionally uses HTTP. `ssl_verify` and `http_timeout` apply to this transport as described in [Configuration](https://github.com/promptility/GAI-pages-pub/blob/main/CONFIGURATION.md).

## Install the external MCP environment

Use Python 3.12 or 3.14, the tested external runtimes:

```bash
python3.14 -m venv /absolute/path/to/mcp-env
/absolute/path/to/mcp-env/bin/python -m pip install 'mcp>=1.28,<2' 'pyzmq==27.1.0'
```

The MCP SDK documents the supported v1 client API. The SDK is pinned below v2 because the bridge uses v1 names and sessions. Sublime's `plugin_host` is not an external Python executable. Python packages installed in this environment do not replace Sublime's Package Control libraries.

Use absolute interpreter/script paths; GUI applications can have a different working directory and PATH from your terminal. On Windows, the interpreter is normally `C:\path\to\mcp-env\Scripts\python.exe` in JSON, with escaped backslashes.

## Stdio server

The stdio bridge launches the server command and maintains its session:

```json
{
  "mcp": {
    "python_executable": "/absolute/path/to/mcp-env/bin/python",
    "request_timeout": 10,
    "discovery_timeout": 10,
    "stdio": [{
      "local": {
        "server_config": {
          "command": "/absolute/path/to/mcp-env/bin/python",
          "args": ["/absolute/path/to/sublime-GAI/examples/mcp_demo.py"],
          "env": {"SERVICE_TOKEN": "REPLACE_IF_YOUR_SERVER_NEEDS_THIS"}
        }
      }
    }]
  },
  "mcp_use": ["local"],
  "tool_use": ["local::echo", "local::add"]
}
```

The demo does not need `SERVICE_TOKEN`; omit `env` for it. For another server, supply its actual command, arguments, and literal environment values. `command` is a string and `args` is an array, not an array inside `command`. GAI does not substitute `${VARIABLE}` in these settings. Keep stdio server protocol output on stdout and diagnostics on stderr.

The external bridge loads GAI from the checkout or installed `.sublime-package` archive; it does not import Sublime APIs. ZeroMQ is the plugin-to-bridge transport, not the MCP server protocol itself. Windows uses loopback TCP endpoints; Unix-like hosts use IPC sockets.

## Streamable HTTP server

Start the same demo in HTTP mode using the external environment:

```bash
/absolute/path/to/mcp-env/bin/python examples/mcp_demo.py \
  --transport streamable-http --host 127.0.0.1 --port 8000
```

Configure its MCP endpoint:

```json
{
  "mcp": {
    "python_executable": "/absolute/path/to/mcp-env/bin/python",
    "request_timeout": 10,
    "discovery_timeout": 10,
    "http": [{
      "demo_http": {"server_config": {"url": "http://127.0.0.1:8000/mcp"}}
    }]
  },
  "mcp_use": ["demo_http"],
  "tool_use": ["demo_http::echo", "demo_http::add"]
}
```

For an authenticated remote server, `server_config` can include SDK-supported headers:

```json
{
  "url": "https://mcp.example.com/mcp",
  "headers": {"Authorization": "Bearer REPLACE_WITH_MCP_TOKEN"}
}
```

Use the server's real Streamable HTTP URL, not an ordinary JSON REST tool route. HTTP and stdio are implemented; legacy `sse` is rejected. GAI does not provide interactive MCP OAuth. The plugin's `ssl_verify` does not configure the external SDK's HTTP TLS stack; use a properly trusted server or external-environment trust configuration supported by that stack.

## Selection and model-specific configuration

- `mcp_use` names must exist in top-level `mcp`. Undefined names are ignored with a warning.
- Server names must be unique across HTTP and stdio and cannot contain `::`.
- `tool_use` matches HTTP dictionary keys and qualified MCP identifiers such as `local::add`. Unix-style glob patterns are case-sensitive: `local::*`, `*::echo`, or `search*`.
- Top-level selection lists are combined with resolved command/model lists and deduplicated. A preset cannot remove a globally selected server/tool by supplying an empty list. Leave the global list empty for per-model allowlists.
- `mcp` definitions cannot be overridden by an alternate. Put server definitions at the top level and select from them in the alternate.
- Only selected routes are advertised and executed. Unmatched patterns produce warnings. A model-requested unadvertised function returns an error without dispatching it.

For separate model tool choices, use flat alternates, an empty global allowlist, and the [actual merge rules](https://github.com/promptility/GAI-pages-pub/blob/main/CONFIGURATION.md#merge-rules-and-model-selection):

```json
{
  "tool_use": [],
  "mcp_use": [],
  "alternates": {
    "default": "with_tools",
    "with_tools": {"mcp_use": ["local"], "tool_use": ["local::add"]},
    "plain": {"mcp_use": [], "tool_use": []}
  }
}
```

Merge this with the provider and top-level server definitions. If `oai` or the command already supplies selection lists, those resolved collisions also follow `__meta__`; an empty **top-level** list alone does not erase them.

## Execution, results, and cleanup

Discovery queries selected servers concurrently. Each tool retains a schema snapshot and route; identifiers remain `server::tool` in user settings, while the model sees bounded deterministic `mcp_...` aliases. Colliding advertised HTTP/model names are rejected. Do not attempt to select an internal hash alias in settings.

The worker appends the assistant's original tool-call message, executes its calls in order, then appends matching tool results. MCP structured content, text/image blocks, and `isError` are serialized as JSON; this preserves data but does not implement a native multimodal chat UI. Handwritten HTTP schemas are validated structurally; full argument-schema enforcement is the provider/server's responsibility.

The loop continues until the model stops calling tools or `max_tool_iterations` is reached (default 10). At the limit, the worker requests a final response without tools. With `response_format`, final formatting occurs after the tool phase. Tools are automatic once selected; there is no per-call confirmation UI. Select tools whose actions you intend to authorize.

`request_timeout` and `discovery_timeout` default to five seconds and must be positive finite values. A discovery failure omits that server while retaining working servers. A failed/cancelled RPC closes the affected client rather than reusing a poisoned request socket. Live routes belong to the interaction and are closed on normal completion or cancellation. Separate interactions sharing a bridge have separate SDK sessions and pending batches.

## Tool logging

Set `operation.log_level` to control tool logging in the Sublime console and
the optional `operation.log_file`:

| Level | Tool information |
|---|---|
| `info` | Start and completion, configured HTTP tool key or MCP `server::tool` name, transport, elapsed seconds, and success status |
| `debug` | All INFO events plus full arguments, full returned output, and exception tracebacks |
| `warning` | Failures (including MCP `isError` results), rejected unselected tools, and cancellations, with elapsed seconds |
| `error` / `critical` | Suppresses the tool lifecycle events above |

Every event includes an `interaction` ID for the generation and a `call` ID
from the model. This distinguishes concurrent buffers and repeated tool calls.
Timing measures the local tool execution, including transport waits, and excludes
model generation and discovery. Tool logs never become part of the editor text.

For complete tool inspection:

```json
"operation": {
  "log_level": "debug",
  "log_file": "/absolute/path/to/gai.log"
}
```

DEBUG logs contain the actual tool arguments and output, which may include
private data. Full API configuration objects are not logged.

## Troubleshooting

| Symptom | Check |
|---|---|
| No MCP tools | Correct `mcp_use`, supported transport, external interpreter, installed SDK/pyzmq |
| Bridge cannot import SDK | Run `EXTERNAL_PYTHON -m pip show mcp pyzmq`; use the v1 requirement |
| Child startup error | Diagnostics include the child traceback/path; verify command, args, environment, and permissions |
| HTTP tool error | Absolute HTTP(S) URL, JSON object args, response status/body, and service availability |
| Model rejects MCP messages | Provider compatibility; Groq documents unsupported message `name` fields |
| Discovery expires | Increase discovery/RPC timeout for slow startup, within the total interaction timeout |
| Stdio protocol errors | No startup banners or ordinary `print()` calls on stdout |
| Unmatched pattern | Match the HTTP dictionary key or `server::tool`, respecting case |
