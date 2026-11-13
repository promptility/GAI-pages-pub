# Build and release GAI

Run these instructions from a checkout containing the desired fixes. This branch is not automatically the same as the latest published tag. A build creates a local archive; a pushed release tag publishes through GitHub Actions.

## Prepare a version

You can bump `VERSION` on a release-preparation branch independently of the
currently public release. `website/release.json` selects the archive offered on
the public website and may remain on an older version. Updating it is a separate
publication step after manually uploading the release archive to S3; see the
website publication procedure.

`VERSION` is the version source and must contain three-component SemVer, such as `0.8.0` or `0.8.1`. A prerelease may use `0.9.0-dev.1`; Package Control clients need matching prerelease settings to receive it. The migration from the former `0.8` metadata normalizes that value to `0.8.0` without publishing a new release.

Update `VERSION` only when preparing the actual release. The first-load upgrade pane shows `RELEASE_NOTES.md`, titled with `VERSION`, once per installed version. Major releases must include the **complete changelog between releases**, rather than only selected highlights. Generate the full notes for every release, including major releases:

```bash
git fetch --tags
python3 scripts/generate_release_notes.py
```

Use a full checkout (run `git fetch --unshallow --tags` first for shallow clones).
The generator selects the nearest reachable version tag, excluding `v<VERSION>`
itself, and includes every commit since that tag through `HEAD`, oldest first,
with subjects and commit hashes. It includes merge commits and changes from
merged branches. For an initial release it includes all history. To select an
explicit range, use `--since v0.21.0 --until v0.22.0`. Review and commit the
generated file before tagging, so installations from the source tag also receive
the full notes. Optional highlights may precede the complete list, but must not
replace it. Regenerate after adding further release changes.

Tagged CI builds regenerate this file from the full Git history before packaging
and use the same file as the GitHub Release body. Branch builds retain the
checked-in notes. The builder itself works without Git and requires non-empty
release notes. `package-metadata.json` is generated inside the archive, not
tracked in source. Generated metadata declares Sublime build `>=4213` and Python
`3.14`; keep repository release definitions consistent with those requirements.

## Test before packaging

```bash
GAI_DEV_ENV="$HOME/.virtualenvs/gai-dev"
"$GAI_DEV_ENV/bin/python" -m pytest tests/ --cov --cov-config=.coveragerc \
  --cov-report=term-missing --cov-report=xml
```

Coverage must be at least **95%** for the application, including `plugin.py`
and MCP subprocess code. CI requires that threshold independently on Python 3.12
and 3.14. See [the coverage gate and master protection setup](https://github.com/promptility/GAI-pages-pub/blob/main/help/DEVELOPMENT.md#required-coverage-gate).

For a maintenance release, inspect the diff and confirm that API keys, user settings, private repository credentials, and logs are not included. Test the candidate archive in real Sublime as described under [Development](https://github.com/promptility/GAI-pages-pub/blob/main/help/DEVELOPMENT.md).

## Build the archive

The builder needs Python 3.9+ and only its standard library. It does not need `zip`, vendor SDKs, or root privileges:

```bash
python3 scripts/build_release.py dist/gai.sublime-package
```

The shell wrapper remains available on Linux/macOS:

```bash
bash release_package.sh dist/gai.sublime-package
```

With no output argument it creates `gai.sublime-package` in the current directory. Relative and absolute output paths are accepted; parent directories are created. On Windows, use `py -3 scripts/build_release.py dist/gai.sublime-package` or the selected Python executable.

The archive includes the plugin entry point, `.python-version`, Python sources under `gai`, settings, command palette, `Main.sublime-menu`, library manifest, generated metadata, README/configuration/help documents, `VERSION` and `RELEASE_NOTES.md`, and example JSON/Python files. It excludes tests, CI files, `.git`, chat histories, virtual environments, logs, and bytecode. Libraries are declared rather than vendored; the external MCP SDK is installed separately.

## Inspect and checksum

```bash
python3 -m zipfile -l dist/gai.sublime-package
python3 -m zipfile -t dist/gai.sublime-package
python3 -c "import hashlib,pathlib; print(hashlib.sha256(pathlib.Path('dist/gai.sublime-package').read_bytes()).hexdigest())"
```

Check for `plugin.py`, `gai/async_worker.py`, `.python-version`, `dependencies.json`, `Main.sublime-menu`, and `package-metadata.json` at the archive root. Use the checksum when writing a Package Control repository definition; rebuilds can have different ZIP timestamps, so checksum the exact uploaded artifact.

## GitHub workflow and tag publication

The release workflow builds on pushes to `master`, `dev/*`, and `v*` tags, and on manual dispatch. Branch/manual runs upload a workflow artifact. Tag runs require the tag to equal `v<VERSION>`, then create a GitHub Release and attach `gai.sublime-package`. This check concerns the plugin being built and is independent of the website's selected release. Download and unzip the workflow artifact container to obtain the actual `.sublime-package`.

The release workflow runs the same tests and 95% coverage gate as PR CI on Python
3.12 and 3.14, plus the separate release-tooling suite on Python 3.11:

```bash
python3 -m pip install -r release-tests/requirements.txt
python3 -m pytest release-tests/
```

Packaging, artifact upload, and GitHub release creation depend on all checks
passing; this also applies to branch pushes and manual dispatch. The local
archive builder remains a packaging utility, so run the coverage command above
before distributing a manually built release. Confirm CI and manual editor
validation before pushing the tag. For a new release, after committing the version
and documentation:

```bash
# Substitute the real branch, remote, and new version; do not retag an existing release.
git push gh HEAD
RELEASE_TAG="v$(cat VERSION)"
git tag -a "$RELEASE_TAG" -m "Release ${RELEASE_TAG}"
git push gh "$RELEASE_TAG"
```

These commands are the publication step. Confirm the release job succeeded and download the attached file before updating any repository manifest. Branch protection and GitHub token permissions must allow the workflow to create releases (`contents: write`).

For a release prepared outside GitHub Actions, upload the verified archive using your hosting workflow. See [private S3 distribution](https://github.com/promptility/GAI-pages-pub/blob/main/help/README-private-repo-setup.md) and [public Package Control publication](https://github.com/promptility/GAI-pages-pub/blob/main/help/PUBLISHING.md). Upload the archive first, then update the repository's version, date, URL, and checksum. Use immutable versioned archive paths; do not overwrite a released artifact at the same version.
