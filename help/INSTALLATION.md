# Install GAI

Use one installation method at a time, and name the installed package **GAI**. The current checkout targets Sublime's Python 3.14 host; use stable build 4213 or newer and current Package Control 4 (4.2.8+ supports the Python 3.14 repository schema). An older published tag may contain older functionality.

Install Package Control through Sublime's **Install Package Control** command; consult the official instructions if that command is unavailable. Model API credentials, repository-download credentials, and AWS publisher credentials are separate.

## Public source repository

A public Git repository is usable through **Package Control: Add Repository** even before it is accepted into the public catalog:

1. Add the **public** URL supplied by the maintainer, for example `https://github.com/YOUR_ACCOUNT/sublime-GAI`. Use `https://github.com/flerim/sublime-GAI` only if that repository is publicly accessible to your client.
2. Open **Package Control: Settings – User**. Because the repository name differs from the package name, use the mapping below when needed:

```json
{
  "package_name_map": {"sublime-GAI": "GAI"}
}
```

3. Run **Package Control: Install Package** and choose **GAI**. A source/tag installer selects its available release, not your unpublished local branch.
4. Run **Package Control: Satisfy Libraries** if needed, then restart Sublime.

For current unreleased code, a source checkout under `Packages/GAI` is the more explicit development method. Do not assume that an old GitHub tag includes this branch's MCP fixes. If **GAI** is present in Package Control's default catalog after maintainer submission, use **Install Package** directly without adding a custom repository. This documentation does not assert that submission has happened.

## Public custom repository manifest

A maintainer may supply an HTTPS `repository.json` containing release URLs:

```json
{
  "repositories": ["https://packages.example.com/gai/repository.json"]
}
```

Add that URL through **Add Repository** or merge it into `repositories` in user Package Control settings. Keep existing entries and default channels. Install **GAI** and configure its model. Package Control upgrades reload GAI automatically without restarting Sublime. The [publishing guide](https://github.com/promptility/GAI-pages-pub/blob/main/help/PUBLISHING.md) explains how the maintainer creates this manifest.

## Private repository / S3 distribution

Use the URL and authentication supplied by the maintainer. For a stable HTTPS gateway backed by a private S3 bucket:

```json
{
  "repositories": ["https://packages.example.com/gai/repository.json"],
  "http_basic_auth": {
    "packages.example.com": ["gai-reader", "REPLACE_WITH_REPOSITORY_PASSWORD"]
  }
}
```

Authentication is keyed by the host, and the same host should serve the manifest and release asset. Install **GAI** through Package Control. These credentials authenticate downloads; they do not configure model or MCP authentication.

Direct S3 presigned URLs are another option, but both the manifest URL and the release URL inside it expire. Copy the full signed manifest URL into `repositories`, with its query string intact. Refresh both links when they expire; a normal `s3://...` URL or an unsigned private bucket URL will not work. The [private distribution guide](https://github.com/promptility/GAI-pages-pub/blob/main/help/README-private-repo-setup.md) gives publisher and client procedures, including a local manifest alternative.

For a private GitHub checkout, use SSH or your organization's Git credential manager:

```bash
git clone git@github.com:YOUR_ORGANIZATION/YOUR_PRIVATE_REPOSITORY.git /actual/Sublime/Packages/GAI
```

This is an unpacked development/manual installation with Git-based updates. Package Control HTTP authentication does not configure Git SSH. For managed private updates, distribute an archive through the authenticated HTTPS/S3 repository rather than relying on an unauthenticated GitHub download URL.

## Manual archive installation

1. Obtain the verified `gai.sublime-package` built by the maintainer, or [build it yourself](https://github.com/promptility/GAI-pages-pub/blob/main/help/RELEASING.md).
2. In Sublime's console (**View → Show Console**), run:

```python
sublime.installed_packages_path()
```

3. Close Sublime. Copy the archive into that directory and **rename it to `GAI.sublime-package`**. The lower-case build filename is an asset name; the installed name must match GAI's hard-coded settings paths.
4. Ensure there is no conflicting `Packages/GAI` checkout or override from an older installation. Keep your user settings in `Packages/User`.
5. Start Sublime, run **Package Control: Satisfy Libraries**, then restart. The archive includes `dependencies.json`; Package Control scans installed packages when satisfying libraries. Command implementation.

Typical installed-package directories are `~/.config/sublime-text/Installed Packages` on Linux, `~/Library/Application Support/Sublime Text/Installed Packages` on macOS, and `%APPDATA%\Sublime Text\Installed Packages` on Windows. Portable installs differ; trust the console path.

Manual copies do not establish a managed update source. For a manual upgrade while Sublime stays open, use **Package Control: Disable Package** to disable GAI, replace the archive, then use **Package Control: Enable Package**. This is the same lifecycle Package Control performs automatically for managed upgrades; it refreshes package resources and runtime code without restarting the application. A raw file replacement alone may leave Sublime's package-resource cache unchanged. You can also move to a Package Control repository for automatic upgrades. Offline installation also requires preprovisioning compatible Jinja2/MarkupSafe/pyzmq/regex/Markdown libraries into Sublime; system pip is not a substitute.

## Manual unpacked source installation

Use **Preferences → Browse Packages...** to locate `Packages`, then place the repository root at `Packages/GAI`. `plugin.py` and `.python-version` must be directly inside that folder, not under an extra `sublime-GAI` subfolder. A symlink to a workspace checkout also works. Follow [Development](https://github.com/promptility/GAI-pages-pub/blob/main/help/DEVELOPMENT.md) for an external test/MCP environment.

Run **Satisfy Libraries** and restart. The source intentionally has no tracked `package-metadata.json`; the library manifest and `.python-version` supply the needed information.

## Verify and configure

Open **GAI: Settings** and paste an object from [example_config.json](https://github.com/promptility/GAI-pages-pub/blob/main/example_config.json). Replace the key and choose a [provider](https://github.com/promptility/GAI-pages-pub/blob/main/help/PROVIDERS.md). Select one or more non-empty regions and run Generate text, Instruct, or Edit. Instruct accepts an instruction and appends an answer after each selected region; when all selections are empty carets, Generate and Instruct use the entire buffer as context and generate directly at each cursor. Without selected text, Edit edits the entire buffer once, even with multiple carets. Mixed selections keep their existing behavior. Use Combined Edit to send all selected regions as one request and append its answer after the last region. MCP needs an additional external interpreter and server selection; see [Tools and MCP](https://github.com/promptility/GAI-pages-pub/blob/main/help/TOOLS.md).

If commands are missing, inspect the console for import errors before troubleshooting the provider. Verify `jinja2` or `zmq` from a Python 3.14-hosted console context; a legacy console context can show a different library set. Restart after installing/upgrading libraries. A missing Markdown library shows question answers as source text; a missing Jinja2 affects templating only; a missing pyzmq affects MCP when it is imported.
