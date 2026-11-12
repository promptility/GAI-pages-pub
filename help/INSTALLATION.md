# Install GAI 1.0.0

The package name is **GAI**, its author is **Promptility**, and its description
is **General AI**. Use **Sublime Text build 4213 or newer**, with the Python 3.14
plugin host, and **Package Control 4.2.8 or newer**. The downloadable archive's
metadata requires build 4213 even though the catalog submission originally used
`>=4000`.

Open the command palette with **Ctrl+Shift+P** (Windows/Linux) or **Cmd+Shift+P**
(macOS). If Package Control is not installed, run **Install Package Control** first.

## Option 1: Install by name through Package Control

1. Open the command palette and run **Package Control: Install Package**.
2. Search for **GAI** and select the entry by **Promptility**, described as
   **General AI**.
3. Wait for installation and any library downloads to finish.
4. Run **GAI: Settings** and configure your provider as described below.

This route becomes available when the Package Control submission has been
accepted and indexed. If **GAI** is not listed, use Option 2. Adding the Pages
Git repository as a Package Control repository will not install the plugin;
that repository contains the website and documentation.

## Option 2: Install using a local repository manifest linked to S3

This route works independently of catalog acceptance. Package Control reads a
local JSON file, then downloads **GAI 1.0.0** from the S3 URL in that file.

1. Download the ready-made [GAI repository.json](https://promptility.github.io/GAI-pages-pub/repository.json).
   Save the JSON file itself, not the browser's HTML page. It contains
   `schema_version: "4.0.0"`, package **GAI**, release **1.0.0**, its complete
   signed S3 URL, and the Python/library requirements.
2. In Sublime, open **View → Show Console** and run:

   ```python
   sublime.packages_path() + "/User"
   ```

   Save the downloaded manifest in that directory as **GAI-repository.json**.
   On Windows, ensure the filename is not `GAI-repository.json.txt`.
3. In the same console, run:

   ```python
   __import__('pathlib').Path(sublime.packages_path(), 'User', 'GAI-repository.json').as_uri()
   ```

   Copy the resulting `file:///.../User/GAI-repository.json` URI without its
   surrounding quotes. This command provides the exact URI for your installation
   on Windows, macOS, or Linux.
4. Run **Package Control: Add Repository** from the command palette. Paste that
   `file:///...` URI and press Enter. The repository is the local JSON file;
   the package URL inside it points to S3.
5. Run **Package Control: Install Package**, search for **GAI**, and select it.
   Wait for the package and its libraries to install.
6. Run **GAI: Settings** and configure your provider below.

Keep **GAI-repository.json** in that location. Package Control needs it to check
for updates. To refresh the release or an expired S3 link, download the current
manifest again and replace that file. Preserve other repositories in Package
Control settings. The signed download link has an expiry; an unsigned S3 URL
will return Access Denied.

## Direct archive download and installation

For a direct download, open the GAI homepage and click **Download GAI 1.0.0**.
The button downloads **gai.sublime-package** from the same signed S3 URL used by
the manifest.

1. In the Sublime console, run:

   ```python
   sublime.installed_packages_path()
   ```

2. For a new installation, copy the downloaded archive into that exact directory
   and rename it **GAI.sublime-package**. If GAI is already running, first use
   **Package Control: Disable Package → GAI**, replace the archive, then use
   **Package Control: Enable Package → GAI**.
3. Run **Package Control: Satisfy Libraries**. Allow the library downloads to
   complete; restart Sublime if Package Control requests it.
4. Run **GAI: Settings** and configure your provider below.

Avoid a conflicting unpacked `Packages/GAI` installation. Preserve your user
settings in `Packages/User`. Direct archive copying does not configure an update
source; use Option 2 for Package Control to manage this S3 release.

## Configure and verify GAI

1. Run **GAI: Settings** from the command palette.
2. Copy [example_config.json](https://github.com/promptility/GAI-pages-pub/blob/main/example_config.json) into the **user** settings
   pane. Replace the API key and choose your endpoint and model using the
   [provider guide](https://github.com/promptility/GAI-pages-pub/blob/main/help/PROVIDERS.md). Model API usage is separate from installation.
3. Select some text and run **GAI: Ask …**. A Markdown answer appears in the
   conversation panel. **GAI: Edit …** replaces selected text;
   **GAI: Generate text** appends generated output.

The package declares **Jinja2**, **MarkupSafe**, **regex**, **pyzmq**, and
**Markdown** as Sublime libraries. Installing these with a system `pip` does not
install them into Sublime. If commands are missing, inspect **View → Show
Console** for import errors and run **Package Control: Satisfy Libraries**.
MCP use additionally needs an external Python environment; see
[Tools and MCP](https://github.com/promptility/GAI-pages-pub/blob/main/help/TOOLS.md).
