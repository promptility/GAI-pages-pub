# Public publication and Package Control

Public source hosting, a GitHub Release, a custom Package Control repository, and inclusion in the default Package Control catalog are separate steps. Building or pushing a branch does not submit the plugin to a catalog. This guide documents the process; it does not assert that GAI's current integration branch has been published.

## Publish the source and release asset

Create or select a **public** repository accessible without login. The repository root must be the package root, with `plugin.py`, `.python-version`, and `dependencies.json` directly under it. The installed package name is `GAI`, even if the source repository is named `sublime-GAI`.

Prepare a new semantic version in `VERSION`, run tests and actual Sublime validation, build the archive, and publish a matching tag/release using [Releasing](https://github.com/promptility/GAI-pages-pub/blob/main/help/RELEASING.md). Keep generated `package-metadata.json`, caches, credentials, logs, and development environments out of the source repository. The current builder generates installation metadata only inside its archive.

For code-hosting-only distribution, tell users to add the public Git repository URL in **Package Control: Add Repository**. They may need `package_name_map` to map `sublime-GAI` to `GAI`. This works independently of catalog approval; see [Installation](https://github.com/promptility/GAI-pages-pub/blob/main/help/INSTALLATION.md).

## Publish a custom public repository manifest

For a built archive hosted on GitHub Releases or another HTTPS host:

```bash
GAI_VERSION=$(cat VERSION)
python3 scripts/make_repository.py --package dist/gai.sublime-package \
  --url "https://github.com/YOUR_ACCOUNT/sublime-GAI/releases/download/v$GAI_VERSION/gai.sublime-package" \
  --output dist/repository.json --author YOUR_ACCOUNT \
  --homepage https://github.com/YOUR_ACCOUNT/sublime-GAI
```

The helper derives the version, minimum Sublime build, Python version, library list, and SHA-256 from the exact archive. Upload `repository.json` to a stable, publicly readable HTTPS URL. Add that URL to user `repositories`, then install **GAI**. The JSON format follows Package Control's schema-4 example.

The generator writes one release. Append earlier releases if you need history or separate compatibility ranges. Use a new version and immutable archive URL for each release. Upload the archive before updating the manifest. A public catalog crawler must be able to read both without a token, password, expiring capability URL, or interactive login.

## Public S3 delivery

You can reuse the manifest/archive structure from the [S3 guide](https://github.com/promptility/GAI-pages-pub/blob/main/help/README-private-repo-setup.md) with an HTTPS delivery host. A private S3 origin behind CloudFront Origin Access Control keeps direct bucket access private while allowing **public** viewer downloads through the distribution. Do not add viewer authentication when the files are intended for the default public catalog. See CloudFront private origins.

A directly public S3 prefix is another option. Use a separate public-distribution bucket/prefix, HTTPS object URLs rather than an HTTP-only website endpoint, and a policy allowing anonymous `s3:GetObject` for only the release/manifest prefix. The bucket/account Block Public Access settings must permit that policy. Do not make the existing private bucket public as a shortcut. AWS documents the alternatives in public access guidance.

Example policy for a dedicated public bucket (apply only after replacing the bucket name and choosing this public-delivery design):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::YOUR_PUBLIC_RELEASE_BUCKET/gai/*"
  }]
}
```

Upload with `aws s3 cp` as in the private guide, generate the release URL as `https://YOUR_PUBLIC_RELEASE_BUCKET.s3.YOUR_REGION.amazonaws.com/gai/releases/VERSION/gai.sublime-package`, and host the manifest at the corresponding `/gai/repository.json`. Test anonymous downloads and checksums before distributing the URL.

## Submit to the default Package Control catalog

The public homepage source lives in `website/` and is exported to a separate
Pages repository on `v*` tag pushes or manual runs. See
website setup and publishing for credentials, build-only
testing, and destination configuration.

Package Control's package ecosystem uses `packagecontrol.io` and the newer `packages.sublimetext.com` browsing frontend. Submission is through the maintained sublimehq/package_control_channel repository, not an upload of a ZIP to a generic “package.io” service. Follow its current README/contribution rules and the submission guide.

Choose one submission route:

### Public Git tags

Fork the channel repository and add GAI in alphabetical order to the appropriate file under `repository/` (normally `g.json`). A tag-based entry can look like:

```json
{
  "name": "GAI",
  "details": "https://github.com/YOUR_ACCOUNT/sublime-GAI",
  "releases": [{
    "sublime_text": ">=4213",
    "python_versions": ["3.14"],
    "platforms": ["*"],
    "tags": true,
    "libraries": ["Jinja2", "MarkupSafe", "regex", "pyzmq"]
  }]
}
```

This route packages the source tag; it does not automatically use the clean archive produced by this project's builder. Review the tagged tree for development-only files and follow the channel's packaging rules. Use asset-based release definitions if the current channel schema/review policy permits the desired GitHub Release asset; consult its live examples rather than assuming branch archives and release artifacts are interchangeable.

The minimum build/Python selectors must be correct for every selected tag. If older tags exist with different host requirements, use tag prefixes or explicit compatibility ranges according to the channel's schema. Do not label old Python 3.8 releases as Python 3.14 builds merely by changing the new release entry. Branch-based releases are not the normal submission path for new default-channel packages.

### Public self-hosted manifest

Host the schema-4 `repository.json` and clean archive on stable HTTPS URLs. In the channel fork, add the public manifest URL to the channel's repository list as directed by its current contribution process. This keeps explicit `version`, `url`, `date`, and checksum fields in your own manifest instead of placing fixed-download entries in the centralized tag-based repository.

### Validate and submit

Validate JSON, the package's selectors, download URLs, declared libraries, and release contents. Run the channel repository's checks and respond to maintainer review. Submit a pull request to the channel repository. Once accepted and crawled, confirm GAI is installable through **Package Control: Install Package** on a compatible Sublime installation without a custom repository setting.

Check the catalog for existing entries/name conflicts before submitting. Keep the release URL reachable after acceptance; private S3 presigned links and HTTP Basic-protected repositories belong in private/custom client settings, not the default public catalog. Subsequent tag-based versions are discovered by the crawler; self-hosted versions require updating your manifest.
