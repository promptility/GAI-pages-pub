# Configuring GAI

Open **GAI: Settings** and edit the user file at `Packages/User/GAI.sublime-settings`. The command palette also opens the packaged defaults for reference. The user file must be a JSON object, not an array. Sublime settings allow comments and trailing commas; the downloadable example files are strict JSON.

Copy [example_config.json](https://github.com/promptility/GAI-pages-pub/blob/main/example_config.json), replace its API key, and adjust the model. Complete alternatives are in [examples](https://github.com/promptility/GAI-pages-pub/tree/main/examples) and [the provider guide](https://github.com/promptility/GAI-pages-pub/blob/main/help/PROVIDERS.md). Settings contain literal strings: `${OPENAI_API_KEY}` and `$OPENAI_API_KEY` are not expanded by GAI.

## Small working configuration

```json
{
  "operation": {"log_level": "info"},
  "oai": {
    "open_ai_base": "api.openai.com",
    "open_ai_endpoint": "/v1/chat/completions",
    "open_ai_key": "REPLACE_WITH_OPENAI_API_KEY",
    "model": "gpt-4.1-mini",
    "temperature": 0.2,
    "max_tokens": 2000,
    "max_seconds": 120,
    "http_timeout": 60
  },
  "alternates": {"default": ""},
  "command_generate": {"keep_prompt_text": true},
  "command_edits": {"keep_prompt_text": false}
}
```

`open_ai_base` is the hostname and optional port, without `https://` or a path. `open_ai_endpoint` is the request path, including any query string. Model requests always use HTTPS. A hosted URL such as `https://openrouter.ai/api/v1/chat/completions` should be split into `openrouter.ai` and `/api/v1/chat/completions`.

## Settings actually used by this checkout

Values below are code fallbacks when a key is absent. Packaged defaults can supply different values, so check both panes of **GAI: Settings**.

| Key | Location | Fallback / meaning |
|---|---|---|
| `operation.log_level` | Top level | `debug` in logger setup; use `info`, `warning`, `error`, or `critical` to reduce output |
| `operation.log_file` | Top level | No file; optional absolute log path in addition to console output |
| `open_ai_base` | `oai`, command, alternate | Required hostname for model requests |
| `open_ai_endpoint` | Same | Required Chat Completions path |
| `open_ai_key` | Same | Required provider credential; no automatic environment lookup |
| `model` | Same | `gpt-4`; select a model supported by your account and payload |
| `persona` | Same | `You are a helpful AI Assistant`; system-message text |
| `prompt` | Same | Empty string; precedes the instruction and selected text |
| `max_tokens` | Same | 100; sent as the legacy `max_tokens` field |
| `temperature` | Same | 0; always sent in the model payload |
| `reasoning_effort` | Same | Absent; optional Chat Completions effort string, such as `low`, `medium`, or `high` |
| `reasoning` | Same | Absent; optional OpenRouter reasoning object, e.g. `{"effort": "high", "exclude": false}` |
| `keep_prompt_text` | Same | False in the worker; Generate defaults to true and preserves the selection above the answer with a blank line. Edit defaults to false and replaces the selection. |
| `max_seconds` | Same | 20 in editor task management; total interaction timeout, including tools |
| `http_timeout` | Same | 30 seconds; positive finite connection/socket timeout for model and HTTP-tool requests |
| `max_tool_iterations` | Same | 10; positive integer; maximum model/tool rounds before a final request without tools |
| `ssl_verify` | Same | Normal certificate verification; CA-file path, `true`, or `false` / legacy string `"False"` |
| `response_format` | Same | Absent; forwarded to the final model request |
| `response_encoding` | Same | Absent; optional template for a JSON-object response |
| `tools` | Top level or resolved settings | HTTP function definitions; see [tools](https://github.com/promptility/GAI-pages-pub/blob/main/help/TOOLS.md) |
| `tool_use` | Top level plus resolved settings | Combined selection patterns for HTTP keys and MCP `server::tool` identifiers |
| `mcp` | Top level | Server definitions, external interpreter, discovery/request timeouts |
| `mcp_use` | Top level plus resolved settings | Combined list of MCP server names to discover |
| `alternates` | Top level or command | Named provider/model presets; `default` chooses one automatically |
| `__meta__` | Top level | Scalar and string merge rules |

`system_prompt` and `instructions` are not prompt fields consumed by the worker; use `persona`, `prompt`, and the Edit command's instruction. `top_p` is read during payload construction but not sent. `frequency_penalty`, `presence_penalty`, `max_completion_tokens`, `stream`, and arbitrary extra settings are not automatically forwarded. This is why a provider supporting Chat Completions can still reject a particular model/configuration.

## Thinking effort and logging

Run **GAI: Set Thinking** from the Command Palette. Select a configured model,
then choose Default, Off, Minimal, Low, Medium, High, Extra high, or Maximum.
The model picker lists unique model IDs from `oai`, command sections, and
`alternates`. The saved preference applies to that model across all profiles
and commands, including the next follow-up in an existing Question conversation.
Requests already running keep their original thinking settings.

Preferences are persisted in **Packages/User/GAI.sublime-settings**, for example:

```json
{
  "model_thinking": {
    "openai/gpt-oss-120b": "high",
    "google/gemini-2.5-flash": "low"
  }
}
```

The palette preference overrides the profile's `reasoning_effort` or `reasoning`
setting. **Default** removes that model's override and restores the existing
configuration; it does not disable thinking. **Off** saves `none`. On OpenRouter,
GAI uses `reasoning.enabled: false` for Off, or a `reasoning.effort` object for
other levels. Other Chat Completions providers use `reasoning_effort`, unless
the profile already uses a `reasoning` object. An effort override replaces any
`reasoning.max_tokens` budget, while preserving `reasoning.exclude` and other
reasoning fields. If `exclude` was not configured, reasoning remains readable
for logging. Supported levels and whether thinking can be disabled depend on
the model/provider; the palette does not query a live capability catalog.

For a Chat Completions provider that accepts `reasoning_effort`, add
`"reasoning_effort": "high"` to `oai`, a command section, or an alternate.
For OpenRouter, use its reasoning object instead:

```json
{
  "operation": {"log_level": "info"},
  "oai": {
    "reasoning": {"effort": "high", "exclude": false}
  }
}
```

Merge this fragment into your existing provider configuration. Configure only
one of `reasoning_effort` and `reasoning`. Neither is sent by default. Supported
effort values depend on the provider and model; GAI forwards the configured
string without translating it. Both settings remain in intermediate tool requests
and the final structured-output request.

Returned readable reasoning (`reasoning`, `reasoning_content`, and text/summary
entries in `reasoning_details`) appears in INFO messages in the Sublime console
and configured log file. Only the response's final `content` goes into the editor
or response template. Encrypted reasoning is not logged; tool conversations retain
the provider's reasoning fields. A provider may return no readable reasoning even
when effort is configured. No reasoning text is invented in that case.

Reasoning can consume the model's output-token budget. Allow enough `max_tokens`
for both reasoning and the final answer. The provider/model must also accept
GAI's token and temperature parameters.

References: OpenAI Chat Completions
and OpenRouter reasoning.

## Command sections

| Settings section | Command | Shipped palette entry |
|---|---|---|
| `command_generate` | `generate_code_generator` | Generate text |
| `command_edits` | `edit_code_generator` | Edit ... |
| `command_edits` | `combined_edit_code_generator` | Combined Edit ... |
| `command_instruct` | `instruct` | Instruct ... |
| `command_question` | `ask` (`question` alias) | Ask ... |
| `command_write` | `write_code_generator` | None |
| `command_completions` | `complete_code_generator` | None |
| `command_whiten` | `whiten_code_generator` | None; identifier renaming |

Select one or more regions (Generate, Edit, and Instruct also accept empty carets). Edit, Generate, and Instruct submit an independent request for each region, using one alternate selection for the batch. Combined Edit sends the region texts together, separated by blank lines in buffer order, and appends one answer after the last region. Its final selection may be an empty caret marking the output location; earlier selections must contain text. The caret is excluded from the collated input. It uses `command_edits` settings and always preserves the source selections.

Each request contains a system message from `persona` and a user message consisting of `prompt`, the optional Edit or Instruct instruction, and the selection, separated by newlines. `keep_prompt_text: true` preserves the selection. Generate appends its answer with a blank line separator; Edit normally replaces the selection (`keep_prompt_text: false`). With selected text, Instruct appends its answer with a blank line separator, preserving the selection even if `keep_prompt_text` is false. When all selections are empty carets, Generate and Instruct use one snapshot of the entire buffer as context and insert independent answers directly at each caret without adding a separator or duplicating the buffer, even when `keep_prompt_text` is true. If any text is selected, this fallback does not apply; existing per-region validation and behavior remain unchanged, including mixed Instruct selections and carets. Multiple carets and mixed selections create independent requests. It uses `command_instruct` settings and an independent per-window recent-instruction list (up to nine), with the same arrow-key recall and Enter-to-submit behavior as Edit. In Edit, Combined Edit, Instruct, and Question palettes, **Tab** cycles forward through recent prompts and **Shift+Tab** cycles backward, wrapping around. Long-prompt previews wrap to the input width and retain the three-line limit.

When every selection is an empty caret, **Edit** uses and targets the entire
buffer with one request, regardless of the number of carets. Its response
replaces the whole buffer as one undoable replacement under the default
`keep_prompt_text: false`; the existing `keep_prompt_text: true` option still
preserves the original text. Multi Generation samples alternatives for that one
whole-buffer target. If any text is selected, Edit keeps its existing per-region
behavior and mixed-selection validation. Combined Edit continues to require
selected source text and keeps its final-caret output-anchor behavior.

Each request tracks its region as other outputs shift the buffer, and can be cancelled independently with **GAI: Cancel task ...**.

Ask uses the existing `command_question` settings section for compatibility. The old `question` command remains an alias, including custom shortcuts; new bindings should use `ask`. With no highlighted text, the whole
buffer becomes context; multiple highlights are joined with blank lines in
buffer order. Its recent questions are stored per buffer in
`gai_question_history_prompts`, independently of Edit/Instruct history.
Responses are rendered as Markdown in a read-only output panel with a separate
follow-up input box. The original Markdown remains in the transcript as folded
source for copying. Rendering uses the Package Control `Markdown` library; if it
is not installed yet, the answer is shown as source text until **Package Control:
Satisfy Libraries** installs it and Sublime is restarted.
Each follow-up includes the prior questions/answers and the current selection or
whole buffer. The initial model/alternate and tool configuration are reused until
you choose **GAI: Change Ask Model ...**. This command reads current settings
and offers the base Question configuration (**default**) plus all named
alternates, even if an automatic default is configured. The chosen alternate's
model, tools, reasoning, and request limits apply to subsequent questions while
the existing conversation history is retained. Cancelling the picker keeps the
current model. Finish or cancel a pending answer before switching. The status bar
shows **GAI Ask: model** for the open conversation, and clears it when the
conversation closes. `keep_prompt_text` has no effect on the source buffer.

Use `/collapse` or **GAI: Collapse Ask** to hide the panel, and
**GAI: Show Ask** to restore it with the same transcript and model. Switching
to the console or another panel also preserves it. Pending responses are added
in the background without reopening Questions.

Use **GAI: Close Ask**, `/close`, or Escape while Questions is displayed
to discard the transcript and cancel outstanding work. Escape in a different
panel does not discard Questions. The next conversation uses current
configuration and starts with no previous answers. Buffer-local recent question
text is retained. Closing the window or unloading the plugin also ends its
conversations.

```json
{
  "command_question": {
    "persona": "Explain the supplied code clearly. Answer the question directly.",
    "max_tokens": 3000
  }
}
```

## Merge rules and model selection

GAI builds resolved settings from `oai`, the command section, then the selected alternate. The direction of merging matters:

- During global → command construction, unlisted colliding scalar keys retain the global value. Command keys absent from the global settings are added.
- During alternate application, the arguments are reversed: unlisted colliding scalar keys take the alternate value.
- `input_prio_keys` makes command values override global values during construction, but makes the resolved global/command value override the alternate during alternate application. Use this deliberately; it does not mean “alternate wins.”
- `target_prio_str_keys` concatenates strings in global/command order during initial construction; alternate application places alternate text before the resolved text.
- `input_prio_str_keys` concatenates in the opposite order. Both concatenation rules insert two newlines and require string values.
- Dictionaries merge recursively. A dictionary/scalar collision retains the dictionary and stores the scalar under the colliding key; avoid mixing types for the same setting.

These rules are implemented in `gai/config.py`; neither “most specific always wins” nor a universal concatenation order describes them correctly. The provider-switching examples leave scalar override lists empty so the chosen alternate can replace the endpoint, credential, and model. Avoid conflicting global and command scalars unless you intend the documented priority.

Alternates are **flat resolved setting dictionaries**, not objects containing another `oai` block. Copy [multiple_providers.json](https://github.com/promptility/GAI-pages-pub/blob/main/examples/multiple_providers.json):

```json
{
  "__meta__": {
    "input_prio_keys": []
  },
  "oai": {
    "open_ai_base": "api.openai.com",
    "open_ai_endpoint": "/v1/chat/completions",
    "open_ai_key": "REPLACE_WITH_OPENAI_API_KEY",
    "model": "gpt-4.1-mini"
  },
  "alternates": {
    "default": "openai",
    "openai": {"model": "gpt-4.1-mini"},
    "router": {
      "open_ai_base": "openrouter.ai",
      "open_ai_endpoint": "/api/v1/chat/completions",
      "open_ai_key": "REPLACE_WITH_OPENROUTER_API_KEY",
      "model": "openai/gpt-4.1-mini"
    }
  }
}
```

Set `default` to a preset name to skip the chooser, set it to `""` to use the base configuration, or omit it to show the runtime chooser. Command-specific alternates can add presets; conflicting preset fields follow the same construction merge rules. Top-level `tool_use` and `mcp_use` are always combined with resolved selections, with duplicates removed; use an empty top-level list if each model should have a separate tool allowlist. `mcp` definitions are stripped from alternates and cannot be overridden there.

## JSON output and templates

Choose a model supporting JSON output and ask for JSON in the prompt:

```json
{
  "command_generate": {
    "keep_prompt_text": false,
    "prompt": "Return a JSON object with keys answer and source.",
    "response_format": {"type": "json_object"},
    "response_encoding": "Answer: {{ answer }}\nSource: {{ source }}"
  }
}
```

Without tools, the first request includes `response_format`. With tools, it is applied after tool execution in a final request without tool definitions. If the final content parses as a JSON object, a template containing `{{` is rendered using Jinja2; otherwise the legacy `{answer}` form is evaluated as a Python f-string. Use only trusted f-string templates: the current implementation evaluates Python expressions. Jinja rendering uses the model's object keys as template variables. Invalid JSON or template errors fall back to the raw response. Parsed arrays and scalar values are converted back to JSON text before insertion. Responses marked `length` (truncated) or `content_filter` are rejected so incomplete output cannot replace selected text.

Jinja2 is loaded only when needed. A missing library leaves GAI operational and returns raw content for that template, with a recovery message. Run **Package Control: Satisfy Libraries**, then restart Sublime. Installing Jinja2 with system `pip` does not install it into Sublime's embedded runtime.

## Timeouts, TLS, and logging

Set `max_seconds` above the expected total of discovery, model calls, and tool calls; `http_timeout` only bounds individual socket operations, not the entire interaction. MCP `request_timeout` and `discovery_timeout` are separate; see [Tools and MCP](https://github.com/promptility/GAI-pages-pub/blob/main/help/TOOLS.md).

For a local HTTPS endpoint, set `ssl_verify` to an absolute CA PEM path. `false` / `"False"` disables model and HTTPS endpoint-tool certificate verification; it does not change MCP HTTP client TLS settings. Prefer a trusted certificate or explicit CA file.

Every console and file log line starts with `GAI [LEVEL]`, with padding so message text aligns across all levels. Multiline reasoning, tool output, and tracebacks receive the same prefix on every line.

Use an absolute `operation.log_file` path when file output is needed. Console logs are visible through **View → Show Console**. Startup debug logging includes settings; prompt construction and diagnostics can include selected text, credentials, or tool data. Keep user settings and logs out of releases.

## Troubleshooting

| Symptom | Check |
|---|---|
| Commands missing after update | Python 3.14-capable Sublime, current Package Control 4, installed libraries, console traceback |
| Preset appears to do nothing | Flat alternate syntax and the phase-dependent `__meta__` priority rules |
| Provider rejects token/temperature fields | Use a compatible model; changing unsupported fields in settings does not add request support |
| Certificate verification fails | Correct host, trusted certificate chain, or absolute CA-file path |
| HTTP 401/403 | Provider key and endpoint; repository-download credentials are separate from model credentials |
| Tool never advertised | Matching `tool_use`; MCP additionally needs a defined server selected in `mcp_use` |
| Deadline reached | Increase total/per-request timeouts as appropriate and inspect server diagnostics |
| No text inserted | One non-empty selection and a non-empty final response; inspect the console error |

## Keyboard shortcuts

Shortcuts are disabled on first installation. Run **GAI: Enable keybindings ...**
to enable them, or **GAI: Disable keybindings** to disable them. This saves
`keybindings_enabled` in your user GAI settings (default: `false`) and takes
effect immediately. Your custom sequences are preserved when disabling them.

Use **GAI: Edit Keybindings ...** to open the GAI defaults and user settings side
by side. Edit the `shortcuts` section in the user pane; opening the editor does
not enable shortcuts.

Set `shortcuts` at the root of your **user** `GAI.sublime-settings` object. Keys
are GAI command names or dropdown paging actions; values are lists of sequential key presses. An empty
list disables that command's shortcut. Unspecified commands retain the defaults
listed in the README. Changes apply immediately without restarting Sublime:

```json
{
    "keybindings_enabled": true,
    "shortcuts": {
        "generate_code_generator": ["ctrl+m", "g"],
        "edit_code_generator": ["ctrl+m", "e"],
        "instruct": ["ctrl+m", "i"],
        "combined_edit_code_generator": ["ctrl+m", "b"],
        "ask": ["ctrl+m", "q"],
        "gai_select_model": ["f8"],
        "gai_cancel_task": []
    }
}
```

Ctrl+M is already assigned to Sublime's matching-bracket command. Using it as a
prefix can interfere with that command; the defaults therefore use Ctrl+Shift+Y.
Press and release the prefix before the second key. GAI shortcuts apply in editor
buffers and do not intercept typing in palettes, console inputs, or panel inputs.
Edit, Instruct, Combined Edit, and Question shortcuts open their instruction
palettes, preserving recent-prompt navigation.

When keybindings are enabled, **Page Up** and **Page Down** in GAI dropdowns move
the highlight by 10 rows in the current filtered result order. Set `quick_panel_page_size` at the
root of GAI settings to change that distance (1–100 rows). Remap or disable the
keys using `shortcuts.gai_quick_panel_page_up` and
`shortcuts.gai_quick_panel_page_down`, with defaults `["pageup"]` and
`["pagedown"]`. These bindings apply only while a GAI dropdown has focus;
editor navigation and other packages' dropdowns retain their normal bindings.
Enter accepts the highlighted result and Escape cancels as usual.

GAI manages `Packages/GAI Shortcuts/Default.sublime-keymap` from these settings.
Do not edit that generated file; it is removed when GAI unloads. Your existing
User keymap stays untouched. You may also use Sublime's **Preferences: Key
Bindings** for native bindings; later User bindings can override GAI bindings.

## Buffer and window model selection

**GAI: Select model ...** first lists the base `oai` model and named alternates
(including discovered models when provider discovery is enabled). After choosing
a model, a second palette offers **This buffer** and **This window**. Cancelling
either palette leaves all current selections unchanged. The shortcut is
**Ctrl+Shift+Y**, then **M**, configurable as `shortcuts.gai_select_model`.

An explicit selection suppresses the model picker for subsequent Generate, Edit,
Instruct, Combined Edit and Question calls in that scope. Buffer selections take
precedence over window selections, which also apply to new buffers in the same
window. The selected model and connection override command-specific model and
connection fields; prompt and explicit HTTP/MCP tool settings retain their normal
merge behavior. Even selecting the base model pins its model ID instead of
using a command-specific model. Existing requests continue with their snapshots.
Question follow-ups retain conversation history and use the scoped selection on
the next call, even in an existing conversation.

**GAI: Unset model ...** offers buffer or window scope. Unsetting a buffer choice
reveals the window choice, if one exists. Unsetting a window choice leaves
explicit buffer choices intact. Once no choice applies, normal configured
defaults or request-time model picking resume, including for the next Question
follow-up after a scoped choice is unset. With no use of Select model, existing
model selection and Question conversation behavior are unchanged.

Selections are stored in `sublime.cache_path()/GAI/model-selections.json` and
restored across GAI reloads and Sublime restarts. Saved buffers use their file
path; unsaved buffers and windows use opaque identity tokens in Sublime's session
metadata. Session metadata contains no model selection or credential. Cloned
views share a buffer choice, and saving an unsaved buffer also caches its choice
under the new file path. Closing a buffer/window clears its runtime entry, while
leaving its cached choice available for restoration. Use **GAI: Unset model ...**
to remove a choice persistently.

The cache contains only scope identities and model/preset names. Missing or
invalid cache files fall back to normal defaults. It is safe to externally clear
Sublime's cache; cached choices then disappear on the next reload/restart. If the
cache cannot be written, choices still work for the current session. The status
bar shows the effective default model, provider, and Multi Generation state for
that buffer, for example **GAI: model-id | Provider: OpenRouter | Multi: on (3)**.
Multi Generation shows **off** when disabled and the sample count when enabled.
A provider is shown by its platform name or endpoint hostname. When models or
providers vary by request, the corresponding field shows **choose per request**.
The display updates when selecting/unsetting a buffer or window model, switching
providers, toggling Multi Generation, changing settings, and activating a buffer.
Cloned views show the same buffer preference; window defaults still apply in
each view’s own window. Command-specific overrides and an active Ask conversation
can use a different model from the buffer’s displayed default.
Removing a selected manual alternate falls back to the next applicable default.
Switching provider discovery to a different connection temporarily suspends
choices absent from that connection's model list.

The former `gai_set_buffer_model` command remains available to existing custom
bindings, with its former buffer-only behavior. Its default shortcut is now
empty; new bindings should use `gai_select_model`. It affects new Question
conversations, while Select model applies to subsequent calls in existing ones.

### Provider model discovery

Run **GAI: Enable provider models ...**, then select a configured connection.
The picker offers supported endpoints with a non-empty literal `open_ai_key`;
shipped placeholders and environment-variable placeholders are excluded. GAI
looks in `oai`, flat `alternates`, and optional `provider_models.connections`.
Connections with the same endpoint, credentials and discovery options are shown
once. The configured default connection is highlighted initially.

Enablement and the selected connection are stored only in the current window's
settings (`gai_provider_models_source`). Other windows and buffer defaults are
independent. **GAI: Disable provider models** restores that window's normal
configured model choices. There is no global enable setting.

While enabled, model pickers include discovered models and manual presets for
only the selected connection. GAI uses that connection's complete profile,
including its endpoint, credentials and explicitly configured tools. Command-specific
connection fields are replaced by this window selection; prompt and tool
settings remain configured. The model
picker can be opened using **GAI: Select model ...** or **GAI: Change Question
Model ...**; commands configured to ask for a model also use the discovered list.
An existing Question conversation keeps its selected model unless a scoped
Select model choice applies or its Question model is changed explicitly. Enabling/disabling does not change requests already in progress.
Buffer model preferences remain shared across cloned views; a preference for a
different connection is ignored while the window's provider filter is enabled.

Catalogs live under `sublime.cache_path()/GAI/provider-models/`, separately keyed
by connection/account, region, project and discovery options. The cache stores
model IDs and display metadata, never API keys, access tokens or HTTP headers.
Profiles are reconstructed using the current configuration. Sublime may delete
this cache at any time. Catalogs refresh in the background when enabled or when
an enabled window is activated, after `provider_models.cache_hours` (default 24).
A failed refresh retains cached models and is retried with **GAI: Refresh provider
models**. Requests to different connections can discover models concurrently.

Additional connections can be configured without creating manual model presets:

```json
{
  "provider_models": {
    "cache_hours": 24,
    "connections": {
      "My Vercel account": {
        "open_ai_base": "ai-gateway.vercel.sh",
        "open_ai_endpoint": "/v1/chat/completions",
        "open_ai_key": "YOUR_VERCEL_API_KEY",
        "model": "YOUR_CHAT_MODEL"
      }
    }
  }
}
```

Replace the placeholders before using this example. When a connection changes
host, configure its own key and inference path; credentials and custom headers
from another host are never borrowed. See
[provider_models.json](https://github.com/promptility/GAI-pages-pub/blob/main/examples/provider_models.json) for all supported platforms
and [the provider guide](https://github.com/promptility/GAI-pages-pub/blob/main/help/PROVIDERS.md#provider-model-discovery) for cloud setup.

| Platform | Discovery | Additional configuration |
|---|---|---|
| OpenRouter | `/api/v1/models` | Existing OpenRouter connection |
| Vercel AI Gateway | `/v1/models` | Gateway API key |
| Requesty | `/v1/models` on the configured regional router | Requesty API key |
| Portkey | `/v1/models` | Portkey key; routing headers in `open_ai_headers` when needed |
| Azure / Foundry | Azure Resource Manager account deployments | `model_discovery.subscription_id`, `resource_group`, `access_token`; optional `account` |
| AWS Bedrock | Mantle regional `/v1/models` | Bedrock API key and Mantle connection |
| Google Vertex AI | Publisher Model Garden catalog | Google access token; optional `model_discovery.publisher` (default `google`) |
| Google Gemini API | `/v1beta/models` | Gemini API key; `model_discovery.auth: "api_key"` |

Discovery filters out explicitly advertised non-chat/non-text models. A catalog
listing is not a guarantee of model access, quota or acceptance of GAI's request
parameters. HTTP and MCP access remains explicit; advertised capabilities never
grant tools. Generated presets cannot override settings using downloaded fields.
The old OpenRouter-only enable setting and User catalog file are no longer read;
use the window commands above to opt in to provider discovery.

## Multi Generation

Enable or disable sampling for the current buffer with **GAI: Enable Multi
Generation** / **GAI: Disable Multi Generation**. Cloned views share the preference.
It persists in Sublime's disposable cache alongside model selections, including
for unsaved buffers restored by Sublime's session. Clearing the cache resets the
preference after reload. No credentials or generated text are saved there.

```json
"multi_generation": {
    "samples": 3,
    "mode": "auto",
    "parallel_requests": 3
}
```

Run **GAI: Set Multi Generation count ...** to choose 2–16 sequences for the
current buffer. **Use configured number** removes the buffer override and returns
to the selected profile's configured `samples`. The override is shared by cloned
views and persists in Sublime's disposable cache across restarts, including when
an unsaved buffer is restored or saved to a file. It applies to subsequent
requests, preserves request mode and concurrency, and does not enable or disable
Multi Generation. Existing alternatives and in-flight requests keep their count.
The status bar shows the effective count when Multi Generation is enabled.

`samples` accepts 2–16; `parallel_requests` accepts 1–16 and limits concurrent
requests within each generation. Sampling uses the selected profile's normal
`temperature` and other generation settings. Set a nonzero temperature when the
model supports it if you want more varied results; identical alternatives remain
possible. Sampling increases token usage and cost.

In `auto` mode GAI first requests `n` choices. If the endpoint rejects `n` with a
specific unsupported-parameter error, GAI retries using separate requests. If it
ignores `n` or returns fewer choices, GAI keeps those choices and requests only the
missing ones. Detection is remembered for that endpoint/account/model during the
current runtime. Authentication, rate-limit, and unrelated errors are reported
without a sampling retry. A rejected probe may add one unsuccessful API request.
`multiple` always uses separate requests; `single` requires all choices in one
response and reports an error otherwise. When HTTP or MCP tools are selected,
all modes use separate conversations so every alternative completes its own tool
calls. Tool permissions remain those of the selected profile.

Generate, Edit, Combined Edit, and Instruct insert the first alternative once all
samples succeed. Ask conversations remain unchanged. Use **GAI: Choose Generation
...** (Ctrl+Shift+Y then V), **GAI: Next Generation** (then N), or **GAI: Previous
Generation** (then P). These shortcuts are configurable under `shortcuts` as
`gai_choose_generation`, `gai_next_generation`, and `gai_previous_generation`.
The palette supports the usual filtering and PageUp/PageDown navigation.

Selection applies to the generated region nearest the cursor, independently for
each generated selection. Surrounding text and appended prompts are preserved.
Regions track edits elsewhere; if generated text itself has been edited, GAI
refuses to overwrite it when choosing an alternative. Alternatives stay in memory
(up to 20 regions per buffer) until the buffer closes or GAI reloads; only the
enabled/disabled preference survives restart. Disable stops sampling for future
requests and leaves existing alternatives available.

## Status information visibility

The settings summary and Ask model label are visible by default. Configure them
in `GAI.sublime-settings`:

```json
"status_information": {
    "enabled": true,
    "fully_disabled": false,
    "command_seconds": 8
}
```

- `enabled`: show information persistently. Set `false` to hide it between GAI commands.
- `command_seconds`: how long to reveal information after selecting a GAI command,
  even when `enabled` is `false`. Defaults to eight seconds; accepts a positive
  number up to 3600. Repeated commands restart the timer for the source buffer;
  its cloned views share the temporary display.
- `fully_disabled`: set `true` to suppress status information entirely, including
  command-triggered reveals. This override takes precedence over `enabled` and
  the palette toggle. Normal progress and error messages are unaffected.

**GAI: Toggle status information** switches `enabled` and saves the choice in user
settings. Turning it off still reveals the information briefly because the toggle
is itself a GAI command. The toggle does not change `fully_disabled`; clear that
option in the configuration file to permit status information again.
