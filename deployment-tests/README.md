# GAI deployment tests

Reproduce the tool setup used in the installed Sublime Text instance, without
requiring model requests to check the servers themselves.

| File | Purpose |
|---|---|
| `http_server.py` | Current deployed HTTP implementation: `POST /echo` and `POST /add` |
| `mcp_server.py` | MCP SDK v1 `echo` and `add` tools over stdio or Streamable HTTP |
| `setup.sh` | Create an external Python 3.14 environment and install MCP/ZeroMQ |
| `run_http.sh` | Start the HTTP endpoint server on loopback port 8765 |
| `run_mcp_http.sh` | Start an MCP HTTP server at `http://127.0.0.1:8000/mcp` |
| `run_mcp_stdio.sh` | Start MCP stdio; normally GAI starts this transport itself |
| `check_tools.py` | Check HTTP and MCP discovery/execution without a model API key |
| `GAI.sublime-settings.example` | Credential-free template for two reasoning models and tool presets |
| `make_settings.py` | Generate settings containing absolute paths and an optional key from the environment |

The HTTP server is a snapshot of the deployed service, including addition.
The MCP implementation is the repository demo server. SondaHub is the public
MCP service used in the current deployment; its source is external to this repo.
The deployment configuration does not require a public MCP service: local stdio
and HTTP alternatives are included.

## 1. Install the environment

Run from this directory:

```bash
./setup.sh
```

With `uv` installed, setup obtains Python 3.14 if necessary. Otherwise it expects
`python3.14` with `venv` and `pip` support already installed. The environment lives
outside the Git worktree at `$HOME/tmp/gai-deployment-tests-venv` (on this
container, `/home/ubuntu/tmp/gai-deployment-tests-venv`). Local settings and logs
live in the ignored `.runtime/` directory. The scripts resolve their own
directory and can also be invoked by absolute path.

To use another environment location, export `GAI_DEPLOYMENT_VENV` before setup
and keep it set when running the launchers or generating settings:

```bash
export GAI_DEPLOYMENT_VENV="$HOME/tmp/gai-deployment-tests-venv"
./setup.sh
```

For the Python commands below, select that environment once:

```bash
DEPLOYMENT_PYTHON="${GAI_DEPLOYMENT_VENV:-$HOME/tmp/gai-deployment-tests-venv}/bin/python"
```

## 2. Start the servers

In one terminal:

```bash
./run_http.sh
```

In another terminal:

```bash
./run_mcp_http.sh
```

Both commands run in the foreground; Ctrl-C stops that server. The stdio server
is started and stopped by GAI for each interaction. Do not launch it separately
unless you are connecting a stdio MCP inspector: stdout carries MCP protocol
messages, so it is not an interactive shell.

The HTTP endpoints accept JSON:

```bash
curl --fail-with-body http://127.0.0.1:8765/add \
  -H 'Content-Type: application/json' -d '{"a":17,"b":25}'
# {"sum": 42}

curl --fail-with-body http://127.0.0.1:8765/echo \
  -H 'Content-Type: application/json' -d '{"message":"hello"}'
# {"message": "hello"}
```

Addition accepts finite numbers and rejects booleans or missing operands with
HTTP 400. Neither local server requires authentication. They bind to loopback.

On the already configured container, `gai-start-tools` runs a separate deployed
HTTP copy on port 8765. Reuse it, or run these copies on unused ports:

```bash
./run_http.sh --port 18765
./run_mcp_http.sh --port 18000
```

Those two foreground commands need separate terminals. If you use custom ports
with GAI, update the endpoint URLs in the generated settings too.

## 3. Check the implementations

With the default servers running:

```bash
"$DEPLOYMENT_PYTHON" check_tools.py
```

This checks addition, echo, invalid HTTP arguments, MCP HTTP discovery/execution,
and MCP stdio discovery/execution. It launches its own stdio server and closes
its MCP sessions when finished. It exits nonzero on failure. No paid model
requests are made.

To include the public server:

```bash
"$DEPLOYMENT_PYTHON" check_tools.py --public
```

To check servers on alternate ports:

```bash
"$DEPLOYMENT_PYTHON" check_tools.py \
  --http-url http://127.0.0.1:18765 \
  --mcp-url http://127.0.0.1:18000/mcp
```

The public service requires internet access and its availability may vary.

## 4. Generate Sublime settings

Generate settings with a placeholder API key:

```bash
"$DEPLOYMENT_PYTHON" make_settings.py
```

Or read your OpenRouter key without putting it in shell history:

```bash
read -rs -p 'OpenRouter API key: ' OPENROUTER_API_KEY
printf '\n'
export OPENROUTER_API_KEY
"$DEPLOYMENT_PYTHON" make_settings.py --log-level debug
unset OPENROUTER_API_KEY
```

The output is `.runtime/GAI.sublime-settings`, readable only by its owner.
The generator refuses to overwrite an existing output unless `--force` is given.
It never prints the credential. To use another destination, pass `--output PATH`.
If generated without a key, replace `REPLACE_WITH_OPENROUTER_API_KEY` manually.
Moving this directory requires regenerating the settings because paths are absolute.

In Sublime, open **Preferences → Package Settings → GAI → Settings**. Back up your
current user settings, then merge or paste the generated file into that user
settings pane. The generator does not alter the running editor's settings.

The template includes **GPT-OSS 120B** and **Gemini 2.5 Flash**, each with:

- Thinking with no tools.
- HTTP endpoints: `http_echo`, `http_add`.
- Public MCP: `public_test::echo`, `public_test::add`.
- HTTP endpoints plus public MCP.
- Local MCP stdio: `local_stdio::echo`, `local_stdio::add`.
- Local MCP HTTP: `local_http::echo`, `local_http::add`.

Select a preset in the generation/edit picker. There is no automatic alternate
selection in the template; the picker’s `default` entry uses the base GPT-OSS
configuration without tools. The model requires your own OpenRouter credit or
applicable account allowance. An API call is made when generating in Sublime.

## 5. Exercise GAI in Sublime

Select a prompt and invoke **GAI: Generate text**. Example prompts:

| Preset | Selected prompt |
|---|---|
| HTTP tool | `Use http_add to calculate 17 plus 25. Report the result.` |
| HTTP tool | `Use http_echo with message GAI_HTTP_READY. Report the returned message.` |
| MCP local_stdio | `Use the available MCP add tool to calculate 17 plus 25.` |
| MCP local_http | `Use the available MCP echo tool with message GAI_MCP_READY.` |
| MCP tools (public) | `Use the MCP add tool to calculate 17 plus 25.` |

The model chooses whether to call a tool; verify execution in the tool logs.
Generate preserves the selected prompt and puts its answer below it. Edit
replaces the selection. Thinking is logged separately from the final answer.

`operation.log_level` controls the console and `.runtime/gai.log`:

- INFO: thinking returned by the model, tool name, transport, start/completion,
  duration, generation ID, and call ID.
- DEBUG: also full tool arguments/output and exception tracebacks.
- WARNING: tool failures, rejected calls, and cancellations with duration.

For a concurrency check, trigger generation in three separate buffers before
any completes, using distinct input numbers or echo markers. Check that results
land in the matching buffers and log entries have distinct interaction IDs.
These are manual editor checks; `check_tools.py` checks servers, not model-driven
routing, editor placement, reasoning, or editor concurrency.

## Building the plugin from this checkout

From the repository root:

```bash
python3 scripts/build_release.py /tmp/gai-release/GAI.sublime-package
```

Install that archive using the [manual installation instructions](https://github.com/promptility/GAI-pages-pub/blob/main/help/INSTALLATION.md).
The deployment-test directory is checkout tooling and is not included in the
plugin archive. See [Development](https://github.com/promptility/GAI-pages-pub/blob/main/help/DEVELOPMENT.md),
[Configuration](https://github.com/promptility/GAI-pages-pub/blob/main/CONFIGURATION.md), and [Tools and MCP](https://github.com/promptility/GAI-pages-pub/blob/main/help/TOOLS.md)
for the full workflows.
