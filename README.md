# GAI for Sublime Text

GAI sends selected text regions to an OpenAI-compatible Chat Completions endpoint and inserts the answer into the editor. Use it to generate text or code, or edit a selection with an instruction. Requests run on a background event loop; selected HTTP endpoint tools and MCP tools can participate in the same interaction.

This checkout includes the MCP integration and test repairs on `dev/enable_mcp`. Installing an older tagged release does not necessarily include these changes. The code has automated coverage on Python 3.12 and 3.14; the editor plugin targets Sublime's Python 3.14 host. The automated editor tests use Sublime stubs, so validate a release in Sublime before distributing it.

## Buffer models and shortcuts

Use **GAI: Select model ...** to choose a model, then choose **This buffer** or
**This window** in the second palette. Subsequent Generate, Edit, Instruct,
Combined Edit, and Question calls use that choice without reopening the model
picker. A buffer choice overrides the window choice. The complete model preset
includes its provider, tools, MCP servers, and thinking configuration.

Use **GAI: Unset model ...** and choose the buffer or window scope to clear its
selection. Clearing a buffer choice falls back to the window choice, if present;
clearing both restores normal configured defaults or request-time model picking.
The status bar shows each buffer’s effective model, provider, and Multi Generation
state, for example **GAI: model-id | Provider: OpenRouter | Multi: on (3)**. Without an explicit selection, the
existing model-picking behavior is unchanged. Use **GAI: Toggle status information**
to hide/show the persistent summary. Hidden information still appears briefly on
each GAI command; `status_information.fully_disabled` suppresses it entirely.
See [status visibility settings](https://github.com/promptility/GAI-pages-pub/blob/main/CONFIGURATION.md#status-information-visibility).

Buffer and window model choices are stored in Sublime's cache and survive
restarts. Named buffers are matched by file path; restored unsaved buffers and
windows use session identity tokens. Cloned views share a buffer choice.
**GAI: Unset model ...** removes the cached choice. Clearing Sublime's cache is
safe and restores normal defaults after reload. Credentials are never cached.

Default shortcuts use **Ctrl+Shift+Y**, release it, then press:

| Key | Command |
|---|---|
| G | Generate |
| E | Edit |
| I | Instruct |
| B | Combined Edit |
| Q | Ask |
| M | Select model |
| T | Set Thinking |
| X | Cancel task |
| S | Show Ask |
| H | Collapse Ask |
| C | Close Ask |

Ctrl+M is already Sublime's **Jump to Matching Bracket** on all three platforms.
Ctrl+Shift+Y is unused in the shipped default keymaps checked in Sublime build
4215. Other installed packages or operating-system bindings may differ.
See [shortcut configuration](https://github.com/promptility/GAI-pages-pub/blob/main/CONFIGURATION.md#keyboard-shortcuts) to change
sequences in **GAI: Settings**, including using Ctrl+M if desired.

## Requirements

- Sublime Text with the Python 3.14 plugin host: use stable build 4213 or newer. The package's `.python-version` is `3.14`; older Python 3.8-only builds cannot run this checkout. See Sublime API environments.
- Current Package Control 4 with Python 3.14 library support, preferably 4.2.8 or newer. Use Sublime's **Install Package Control** command or its official installation instructions.
- An API key and a model accepting GAI's Chat Completions payload. Provider accounts and API usage are separate from this plugin.
- For MCP: an external Python environment with `mcp>=1.28,<2` and `pyzmq==27.1.0`. It is separate from Sublime's embedded interpreter.

## Start here

1. Follow [installation instructions](https://github.com/promptility/GAI-pages-pub/blob/main/help/INSTALLATION.md): public repository, authenticated private repository, S3 distribution, or manual archive/source installation.
2. Open **GAI: Settings** from the command palette. Put a JSON **object** in the user settings pane, using [example_config.json](https://github.com/promptility/GAI-pages-pub/blob/main/example_config.json) as a starting point.
3. Replace the API-key placeholder. Select a provider from the [vendor examples](https://github.com/promptility/GAI-pages-pub/blob/main/help/PROVIDERS.md).
4. Select one or more non-empty text regions. Run **GAI: Generate text**, **GAI: Instruct ...**, or **GAI: Edit ...**.

With multiple selections, **Edit** applies the same instruction independently to
each region and replaces it with its answer. **Generate** starts an independent
request for every region; by default each answer is appended after its region.
Regions track changes in output length, and requests run concurrently.
`keep_prompt_text` controls whether the source text is preserved.

**GAI: Instruct ...** accepts an instruction and applies it independently to each
selected region. Each answer is appended below its selection with a blank line
separator, preserving the source text. Requests run concurrently and can be
cancelled separately. Configure its persona and prompt in `command_instruct`.

Instruct can also generate new text with no selection: place the cursor where
output belongs, run **GAI: Instruct ...**, and enter the instruction. The response
is inserted directly at the cursor, without added blank lines. Generate and
Instruct use the entire buffer as context when all selections are empty carets;
each caret receives an independent response. With any selected text, existing
selection behavior is unchanged: Generate uses each selected region, and Instruct
continues to append its response below each selection. Mixed Instruct selections
and empty carets retain their individual contexts.

With no text selected, **GAI: Edit ...** edits the entire buffer in one request,
even with multiple empty carets. Its response replaces the whole buffer under
the normal Edit settings (`keep_prompt_text: false`) and can be undone as one
replacement. If any text is selected, existing per-selection Edit behavior and
validation remain unchanged. Combined Edit still requires selected source text.

**GAI: Set Thinking** lets you select a configured model and its thinking level.
The preference is saved in your user settings and applies across that model's
profiles and commands, including subsequent Question follow-ups. Choose
**Default** to restore its existing configuration. See
[thinking configuration](https://github.com/promptility/GAI-pages-pub/blob/main/CONFIGURATION.md#thinking-effort-and-logging) for options
and provider support.

**GAI: Ask ...** (formerly Question) asks about the highlighted text, or the whole buffer when
nothing is highlighted. Multiple highlighted regions are joined in buffer order.
The question palette has up to nine recent questions from that buffer only,
with the same highlighted arrow-key recall and long-prompt preview.
Answers are rendered as Markdown in a bottom panel with an input box for
follow-up questions. Headings, emphasis, lists, links, and code blocks are
formatted; tables use aligned monospace blocks. Expand the folded source beneath
an answer to read or copy the original Markdown.
Highlight different text while the panel stays open, then type a follow-up in
its input box and press Enter, or run **GAI: Ask ...** again.
Each follow-up includes fresh context from the active buffer and the preceding
conversation. The source buffer is never changed. One request at a time runs in
each conversation; separate windows can ask questions concurrently. The status
bar shows elapsed time and the request timeout while waiting, including when
the panel is collapsed, and reports completion, cancellation, or failure.
The current model is also displayed persistently as **GAI Ask: model** in
the status bar. Use **GAI: Change Ask Model ...** to select a configured
alternate for subsequent questions without discarding the conversation. The
picker's **default** entry uses the base Question configuration; named entries
apply their complete alternate profile, including tool and reasoning settings.
Finish or cancel any pending answer before changing models.

Use `/collapse` or **GAI: Collapse Ask** to hide the panel without losing
the conversation. **GAI: Show Ask** restores it. Switching to the console
or another panel also preserves the conversation; pending answers finish in the
background without bringing Questions to the front.

Close the conversation explicitly with **GAI: Close Ask**, `/close` in its
input, or Escape while Questions is displayed (outside another palette). This
discards the conversation and cancels any pending answer. Escape in the console
does not close a hidden Questions conversation. A later question starts fresh. Recent question text stays in the buffer’s settings, independently
of the discarded conversation. Configure the assistant using `command_question`;
the selected model and tools remain in use until the conversation closes.

**GAI: Combined Edit ...** sends the selected texts together, joined with blank
lines in buffer order, and applies one instruction to that combined input.
Its single answer is appended after the last selected region, preserving all
selected text and the text between the selections. You may add an empty caret
as the final selection to designate the output location; only the non-empty
selections are sent to the model. At least one source selection is required.
It uses `command_edits`
settings and the same window’s recent Edit instructions. Saving user settings affects subsequent requests.

**GAI: Edit ...** and **GAI: Instruct ...** each show up to nine recent
instructions beneath the input box,
newest first, as text without links. Arrow navigation highlights the active entry
and selects the recalled text in the input so you can replace it by typing, or
move the caret to edit it. Type a new
instruction or use **↑/↓** to recall a recent instruction into the input.
**Tab** cycles forward through the recent entries, wrapping to the first;
**Shift+Tab** cycles backward. Edit it directly, then press Enter
to submit. Moving up past the first entry restores your original draft. Selecting a history entry does not execute it.
Long instructions show a wrapped three-line preview below the single-line
input, above the recent list. Its wrapping follows the palette input width,
using the same font measurements as the displayed preview. It updates while typing and recalling history;
for longer text it shows the last three wrapped lines with an ellipsis.
Overflow is detected from the rendered input, including the command label.
Recent entries stay on one line so they do not push the preview behind the input.
Trailing blank lines are omitted from the preview only.
The full instruction is submitted, including text outside the preview.
The list stays visible while typing and is not filtered. Submitted instructions
are deduplicated and stored in each window’s settings (`gai_edit_history_prompts`
for Edit/Combined Edit, `gai_instruct_history_prompts` for Instruct).
Buffers in the same window share history; separate windows have independent
lists, even when they open the same project or file. The old global history file
is no longer used. Drafts and cancelled input are not added.


Run **GAI: Cancel task ...** to list pending requests in the current window,
including requests in other buffers or multiple requests in the same buffer.
Each single-line title begins with the selected prompt. Select a task to cancel
it, or press Escape to leave it running. While this picker is open, completed
results for its listed tasks wait until you select or dismiss it. A selected
task's result is discarded even if the response arrived while you were choosing.
Other completed results are then applied. Results already inserted before opening
the picker are not undone. Request regions follow intervening buffer edits.

## Documentation

| Task | Guide |
|---|---|
| Install from public/private repositories, S3, or manually | [Installation](https://github.com/promptility/GAI-pages-pub/blob/main/help/INSTALLATION.md) |
| Configure endpoints, model presets, prompts, timeouts, and output | [Configuration](https://github.com/promptility/GAI-pages-pub/blob/main/CONFIGURATION.md) |
| OpenAI, OpenRouter, Azure, Gemini, Groq, Claude, and local HTTPS | [Provider examples](https://github.com/promptility/GAI-pages-pub/blob/main/help/PROVIDERS.md) |
| HTTP tools, MCP servers, filtering, and execution | [Tools and MCP](https://github.com/promptility/GAI-pages-pub/blob/main/help/TOOLS.md) |
| Run HTTP addition/echo and MCP deployment checks with sample settings | [Deployment tests](https://github.com/promptility/GAI-pages-pub/blob/main/deployment-tests/README.md) |
| Set up Python, tests, and a live development checkout | [Development](https://github.com/promptility/GAI-pages-pub/blob/main/help/DEVELOPMENT.md) |
| Enforce 95% coverage for master merges and releases | [Coverage gate](https://github.com/promptility/GAI-pages-pub/blob/main/help/DEVELOPMENT.md#required-coverage-gate) |
| Build an archive, validate it, and create a release | [Releases](https://github.com/promptility/GAI-pages-pub/blob/main/help/RELEASING.md) |
| Host on private S3 and configure Sublime clients | [Private distribution](https://github.com/promptility/GAI-pages-pub/blob/main/help/README-private-repo-setup.md) |
| Publish a public repository or submit to Package Control | [Public distribution](https://github.com/promptility/GAI-pages-pub/blob/main/help/PUBLISHING.md) |

## Commands and behavior

The command palette exposes **Generate text**, **Instruct ...**, **Edit ...**,
**Combined Edit ...**, **Question ...**, **Show Questions**, **Collapse Questions**,
**Close Questions**, **Cancel task ...**,
and **Settings**. See the [command settings table](https://github.com/promptility/GAI-pages-pub/blob/main/CONFIGURATION.md#command-sections).
The code also contains `write_code_generator`, `complete_code_generator`, and
`whiten_code_generator`; these are not exposed by the shipped palette.
“Whitening” means identifier renaming. The shipped keymaps provide arrow-key
recall while an instruction palette is active.

GAI provides non-streaming text output. Question conversations remain available
while the panel is collapsed and end when explicitly closed or when the
window/plugin shuts down; other generation commands start independent
interactions.
The plugin does not automatically gather project files. Selected HTTP tools and
MCP tools may participate in either kind of interaction.

## Compatibility and data flow

GAI uses Chat Completions, not the Responses API or native Anthropic Messages API. The generated payload includes `model`, `messages`, `max_tokens`, and `temperature`; some newer reasoning models reject this combination. Choose a compatible model rather than assuming every model on an endpoint works. `top_p`, penalties, `max_completion_tokens`, custom model-request headers, environment-variable substitution in settings, and OAuth for model requests are not implemented.

Model requests use HTTPS and send the configured key as both `Authorization: Bearer ...` and `api-key`. HTTP endpoint tools use their configured HTTP or HTTPS URL without these model credentials. MCP transports have their own server configuration. Selected text and tool results go to the chosen provider; tools can execute server-defined actions automatically. API keys are not written to the HTTP debug-header log, but prompts, settings, results, and child-process diagnostics can appear in logs. Keep real credentials in user settings and use logging appropriate to the data you handle.

## Upgrades


When GAI is installed or its version changes, the current window opens a
read-only **GAI — Release Notes** tab describing the latest release. Closing
it requires no save. Upgrades reload GAI's runtime code without restarting
Sublime. Pending requests are cancelled and Question conversations close during
the reload; settings and saved thinking preferences are preserved. The notice
is shown once per version, tracked separately in
`Packages/User/GAI-updates.sublime-settings`.

## Project status

The repository documents public and private publication paths; that does not assert that this checkout has been released or accepted into Package Control's default catalog. Refer to the actual release/tag and channel listing when selecting a version. Development and publishing commands in these guides are instructions, not actions performed automatically by the plugin.

Licensed under [Apache 2.0](https://github.com/promptility/GAI-pages-pub/blob/main/LICENSE). See [Code of Conduct](https://github.com/promptility/GAI-pages-pub/blob/main/CODE_OF_CONDUCT.md).

Multi Generation samples several alternatives for Generate, Edit, Combined Edit,
and Instruct. Run **GAI: Enable Multi Generation** for the current buffer; the
preference survives restarts in Sublime's cache. **GAI: Set Multi Generation count ...**
chooses 2–16 sequences for this buffer or restores the configured number; the
choice also survives restarts. Use **GAI: Choose Generation ...**,
or Ctrl+Shift+Y then V, to choose an alternative. Ctrl+Shift+Y then N/P cycles
forward/backward through alternatives near the cursor. **GAI: Disable Multi
Generation** restores one response per request. Ask conversations are unaffected.
See [configuration](https://github.com/promptility/GAI-pages-pub/blob/main/CONFIGURATION.md#multi-generation) for sampling and provider fallback.
