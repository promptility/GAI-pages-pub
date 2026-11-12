# Development environment

GAI has two Python environments: Sublime's embedded Python 3.14 for editor code, and an external Python interpreter for tests and MCP SDK subprocesses. Installing packages into one does not populate the other. [Installation](https://github.com/promptility/GAI-pages-pub/blob/main/help/INSTALLATION.md) covers Sublime libraries.

## Clone and install with uv

Use Git, Python 3.14 (or let `uv` download it), and uv. Run from a terminal:

```bash
# Use the public URL or authenticated Git remote supplied by the maintainer.
git clone https://github.com/flerim/sublime-GAI.git
cd sublime-GAI
# Use a current integration branch while this work is not in a released tag:
git switch dev/enable_mcp
GAI_DEV_ENV="$HOME/.virtualenvs/gai-dev"
uv venv --python 3.14 "$GAI_DEV_ENV"
uv pip install --python "$GAI_DEV_ENV/bin/python" -r dev_requirements.txt
"$GAI_DEV_ENV/bin/python" -m pytest -q tests/
```

If the branch is not present on the remote yet, use the checkout supplied by the maintainer; local commits are not automatically public. Keep the environment outside Sublime's `Packages` directory so it is not indexed as plugin content. `dev_plugin_requirements.txt` includes the same current development dependencies; it is not a separate Python 3.8 environment.

With an existing interpreter instead of uv:

```bash
GAI_DEV_ENV="$HOME/.virtualenvs/gai-dev"
python3.14 -m venv "$GAI_DEV_ENV"
"$GAI_DEV_ENV/bin/python" -m pip install -r dev_requirements.txt
```

On Windows PowerShell, use the equivalent environment and executable paths:

```powershell
$GaiDevEnv = "$env:LOCALAPPDATA\GAI\venv"
uv venv --python 3.14 $GaiDevEnv
uv pip install --python "$GaiDevEnv\Scripts\python.exe" -r dev_requirements.txt
& "$GaiDevEnv\Scripts\python.exe" -m pytest -q tests/
```

The requirements install Jinja2, Markdown, pytest, pytest-asyncio, pytest-timeout, coverage.py,
pytest-cov, pyzmq, and the supported MCP SDK v1 line. The SDK is constrained to `<2`
because its v2 API differs. No vendor API keys or AWS credentials are required for the tests.

## Required coverage gate

Merges to `master` and GitHub releases require **at least 95% application line
coverage**, independently on Python 3.12 and 3.14. Run the same gate locally:

```bash
"$GAI_DEV_ENV/bin/python" -m pytest tests/ --cov --cov-config=.coveragerc \
  --cov-report=term-missing --cov-report=xml --cov-report=html
```

`.coveragerc` measures every Python source file under `gai/` and the root
`plugin.py`. It collects subprocess execution, including editor lifecycle tests
and the external MCP bridge, and saves data on SIGTERM. It does not omit
application compatibility shims or transport entry points. The target is line
coverage, not branch coverage. Tests, examples, deployment helpers, and packaging
scripts are outside this application coverage denominator.

The command exits nonzero below 95%, or if any test fails. Open
`htmlcov/index.html` for uncovered lines. XML/HTML reports are also uploaded by
CI for each Python version. Focused tests below do not measure aggregate coverage;
run the complete gate before merging or releasing.

PR and merge-queue checks expose the stable required-check name
**`Coverage >= 95%`**. This fails if either Python coverage job fails or is skipped,
or if the separate release-tooling job fails.
The release workflow waits for the same reusable quality workflow before building
or uploading artifacts or creating a release.

### Release-tooling tests

Changelog generation and package construction are tested separately in
`release-tests/`, without application coverage or Sublime stubs. CI runs this
suite as **Release tooling** using Python 3.11, the release builder's interpreter.

```bash
python3 -m pip install -r release-tests/requirements.txt
python3 -m pytest release-tests/
```

These tests create their own Git repositories, so they also run from shallow
checkouts without release tags. The changelog generator itself still requires
full history for actual releases; the suite verifies that a shallow clone is
rejected instead of generating incomplete notes.

### Enable GitHub merge enforcement (repository administrator)

Workflow files alone do not enable GitHub branch protection. Import
master-coverage.json in the repository's
**Settings → Rules → Rulesets** using the import option. The active ruleset targets
`master`, requires the `Coverage >= 95%` check with up-to-date commits, and has no
bypass actors. Keep other existing repository rules in place.

Alternatively, with an administrator-authenticated GitHub CLI, create the ruleset
once from the repository root:

```bash
gh api --method POST repos/flerim/sublime-GAI/rulesets \
  --input .github/rulesets/master-coverage.json
```

If this ruleset already exists, edit/import its update rather than creating
duplicates. Run PR CI once so GitHub can list the named check, then confirm the
active ruleset requires it. See GitHub required status checks.

## Run focused tests

```bash
"$GAI_DEV_ENV/bin/python" -m pytest -q -m 'not integration'
"$GAI_DEV_ENV/bin/python" -m pytest -q -m integration
"$GAI_DEV_ENV/bin/python" -m pytest -q tests/test_core.py tests/test_http_transport.py
```

The integration marker covers real HTTP/TLS and MCP transports, packaged bridge startup, and editor lifecycle simulation. Cold-import unit checks use subprocess isolation. Tests have a 30-second watchdog; runtime/resource warnings fail the suite. They use loopback services and generated TLS certificates, not paid provider accounts. Python 3.12 and 3.14 are the CI targets; the editor itself requires 3.14.

## Run a live checkout in Sublime

Find the directory with **Preferences → Browse Packages...**. Remove/move an existing `GAI.sublime-package` while Sublime is closed, then put the repository at `Packages/GAI` or create a link to your workspace checkout. Avoid installing both packed and unpacked copies of GAI.

For example, after finding the actual Linux path:

```bash
ln -s /absolute/path/to/sublime-GAI /actual/Sublime/Packages/GAI
```

On Windows, an elevated command prompt can create a directory link:

```bat
mklink /D "%APPDATA%\Sublime Text\Packages\GAI" "C:\work\sublime-GAI"
```

Portable installs and alternative platform directories differ; use the path shown by Sublime. Run **Package Control: Satisfy Libraries**, restart, and configure user settings. For MCP, set `mcp.python_executable` to the external development environment's absolute Python path. Do not point it at Sublime's `plugin_host`.

Sublime reloads the plugin entry point when the package is re-enabled after a Package Control upgrade. GAI refreshes its absolute `gai.*` helper imports and ZIP import caches at that point, so updated code takes effect without an application restart. Source changes can also trigger plugin reloads. The lifecycle cancels pending tasks and closes its loop on unload; save your work and inspect the console if a reload fails. Validate generate, edit, presets, a selected HTTP tool, an MCP server, and unload/reload in actual Sublime before a release.

## Code map

| Area | Files |
|---|---|
| Entry point and background-loop lifecycle | `plugin.py`, `gai/async_loop.py` |
| Selection, prompts, polling, and editor insertion | `gai/core.py`, `gai/generators.py`, `gai/commands.py` |
| Settings, merging, and validation | `gai/config.py`, `gai/validate_config.py` |
| HTTPS model calls and HTTP/MCP dispatcher | `gai/async_worker.py` |
| MCP factory, discovery, routing | `gai/mcp.py`, `gai/mcp_tools.py` |
| External SDK sessions and ZeroMQ bridge | `gai/_mcp_impl/` |
| Stable editor stub and bounded fixtures | `tests/sublime_stub.py`, `tests/conftest.py`, `tests/helpers.py` |

Build and publication are separate from tests; continue with [Releasing](https://github.com/promptility/GAI-pages-pub/blob/main/help/RELEASING.md).
