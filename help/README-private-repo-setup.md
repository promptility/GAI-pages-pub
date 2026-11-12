# Private GAI distribution: S3 and authenticated repositories

Build and validate the archive with [Releasing](https://github.com/promptility/GAI-pages-pub/blob/main/help/RELEASING.md) first. A Package Control **repository** is a JSON list of packages/releases. A **channel** is a list of repositories. For one private package, put the manifest URL in `repositories`; you do not need a custom channel. Use schema `4.0.0` and release `python_versions: ["3.14"]`. See Package Control's repository example.

There are two practical private-S3 paths:

| Delivery | Sublime access | Updates |
|---|---|---|
| Presigned S3 manifest and archive | Full HTTPS capability URLs | Refresh both links before expiry |
| Authenticated HTTPS gateway reading private S3 | Stable URL plus per-host HTTP Basic credentials | Replace the manifest when adding releases |

Package Control does not natively sign S3 requests with AWS credentials. HTTP Basic credentials cannot turn an unsigned S3 object URL into a SigV4 request. Keep the bucket private and choose one of the paths below.

## 1. Create private storage and upload an archive

Install/configure AWS CLI v2 with an identity permitted to create/configure the bucket and publish objects. Substitute a globally unique bucket name. This example chooses a region outside `us-east-1`, where a location constraint is required:

```bash
GAI_BUCKET="YOUR_UNIQUE_PRIVATE_BUCKET"
GAI_REGION="eu-central-1"
aws s3api create-bucket --bucket "$GAI_BUCKET" --region "$GAI_REGION" \
  --create-bucket-configuration "LocationConstraint=$GAI_REGION"
aws s3api put-public-access-block --bucket "$GAI_BUCKET" \
  --public-access-block-configuration \
  'BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true'
GAI_VERSION=$(cat VERSION)
aws s3 cp dist/gai.sublime-package \
  "s3://$GAI_BUCKET/gai/releases/$GAI_VERSION/gai.sublime-package" \
  --region "$GAI_REGION" --content-type application/octet-stream
```

For `us-east-1`, omit `--create-bucket-configuration`. If the bucket already exists, skip creation and verify its region and policy instead. AWS defaults and ownership settings are described in create-bucket and Block Public Access.

The publisher needs object write permission under `gai/`. A downloader/gateway identity needs `s3:GetObject` for that prefix. For an IAM role/user policy granting reads:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::YOUR_UNIQUE_PRIVATE_BUCKET/gai/*"
  }]
}
```

Use your deployment's role or credential mechanism; do not put AWS access keys in Sublime settings. With SSE-KMS, the reader also needs the applicable key's decrypt permission.

## 2A. Short-lived distribution with presigned URLs

Generate the archive URL, then a matching manifest from the exact local archive:

```bash
GAI_ARCHIVE_URL=$(aws s3 presign \
  "s3://$GAI_BUCKET/gai/releases/$GAI_VERSION/gai.sublime-package" \
  --region "$GAI_REGION" --expires-in 3600)
python3 scripts/make_repository.py --package dist/gai.sublime-package \
  --url "$GAI_ARCHIVE_URL" --output dist/private-repository.json \
  --author YOUR_ORGANIZATION
aws s3 cp dist/private-repository.json "s3://$GAI_BUCKET/gai/repository.json" \
  --region "$GAI_REGION" --content-type application/json --cache-control max-age=60
aws s3 presign "s3://$GAI_BUCKET/gai/repository.json" \
  --region "$GAI_REGION" --expires-in 3600
```

Give authorized clients the final signed manifest URL. In **Package Control: Settings – User**:

```json
{
  "repositories": ["PASTE_THE_COMPLETE_HTTPS_PRESIGNED_REPOSITORY_URL"]
}
```

Keep the entire query string and treat it as a bearer credential. The generated manifest contains a separate signed archive URL; signing only the manifest is insufficient. Its checksum must match the uploaded archive.

A signing identity needs `s3:GetObject`; temporary credential expiry can shorten the URL lifetime. AWS CLI supports up to seven days, but a URL cannot outlive its signing credentials. See AWS presign and presigned URL lifetime. Refresh both URLs and distribute the new manifest URL. This approach is suitable for temporary access, not unattended long-term updates.

Alternatively, distribute the generated manifest as a local file with a signed archive URL:

```json
{
  "repositories": ["file:///absolute/path/to/private-repository.json"]
}
```

On Windows use a URI such as `file:///C:/GAI/private-repository.json`. The archive URL still expires; replacing the local manifest is necessary. Use `file://` only in Package Control settings, not a browser-only `s3://` URI.

## 2B. Stable authenticated HTTPS backed by private S3

Expose the same object keys through a trusted HTTPS host, for example:

```text
https://packages.example.com/gai/repository.json
https://packages.example.com/gai/releases/0.8.0/gai.sublime-package
```

examples/s3_repository_proxy.py is a small reference gateway: it verifies HTTP Basic credentials, restricts reads to a configured S3 prefix, then streams objects using the server's AWS identity. It does not grant bucket public access. Install its optional dependencies in a separate environment:

```bash
python3.14 -m venv /absolute/path/to/gai-repository-env
/absolute/path/to/gai-repository-env/bin/python -m pip install fastapi boto3 uvicorn
export GAI_REPO_BUCKET="YOUR_UNIQUE_PRIVATE_BUCKET"
export GAI_REPO_PREFIX="gai/"
export GAI_REPO_USERNAME="gai-reader"
read -r -s -p 'Repository password: ' GAI_REPO_PASSWORD
export GAI_REPO_PASSWORD
# Set AWS_PROFILE if using a configured profile; deployed hosts can use an IAM role.
/absolute/path/to/gai-repository-env/bin/python -m uvicorn \
  examples.s3_repository_proxy:app_factory --factory --host 127.0.0.1 --port 8080
```

Run from the source checkout. Keep the service under your normal supervisor/container mechanism, supply secrets through that mechanism, and put a trusted TLS proxy in front. For example, a Caddyfile on a host with the matching public DNS name:

```caddyfile
packages.example.com {
    reverse_proxy 127.0.0.1:8080
}
```

Configure DNS and ports according to Caddy HTTPS requirements. The reference gateway is single-account; use your organization's authentication and access controls for multi-user deployments. See FastAPI HTTP Basic and S3 GetObject.

Create and upload a manifest whose release URL uses the stable gateway, not a presigned S3 URL:

```bash
python3 scripts/make_repository.py --package dist/gai.sublime-package \
  --url "https://packages.example.com/gai/releases/$GAI_VERSION/gai.sublime-package" \
  --output dist/repository.json --author YOUR_ORGANIZATION
aws s3 cp dist/repository.json "s3://$GAI_BUCKET/gai/repository.json" \
  --region "$GAI_REGION" --content-type application/json --cache-control max-age=60
```

Sublime user settings:

```json
{
  "repositories": ["https://packages.example.com/gai/repository.json"],
  "http_basic_auth": {
    "packages.example.com": ["gai-reader", "REPLACE_WITH_REPOSITORY_PASSWORD"]
  }
}
```

Package Control's per-host authentication setting applies to downloads from that host. Keep the manifest and archive on the same host; do not redirect authenticated requests to an unrelated domain. Install **GAI**, run **Satisfy Libraries**, and restart. Basic authentication is protected by HTTPS; AWS credentials stay on the gateway host.

CloudFront with a private S3 origin is another hosting option, but origin access control protects only the S3 origin, not viewer access by itself. Add viewer authentication or signed delivery and ensure Package Control can fetch both objects. See CloudFront private S3 origins.

## Upgrades and troubleshooting

Upload the new immutable archive first; regenerate the manifest with its real version and checksum, then replace `gai/repository.json`. The generator emits one release; retain older entries manually if you need a release history or compatibility ranges. Keep repository credentials separate from API keys, and keep `dist/private-repository.json` out of source control because it can contain capability URLs.

| Failure | Likely cause |
|---|---|
| S3 403 on an unsigned URL | Private object without signed delivery; Basic auth is not AWS SigV4 |
| Signed URL expires early | Temporary signing credentials expired |
| Gateway 401 | Missing/wrong host-specific Basic credentials |
| Gateway 502 | Reader role/prefix/region/KMS access failure; inspect server logs |
| Download rejected | Manifest checksum does not match the exact uploaded artifact |
| Release not shown | Version/build/Python selectors, prerelease preferences, or stale repository cache |
| Libraries fail | Keep default Package Control library channels; inspect **Satisfy Libraries** output |

For a private Git repository, Git clone/SSH remains a valid development installation. GitHub token download settings are provider-specific and should not be confused with AWS access. A managed archive repository behind HTTPS avoids reliance on browser-generated GitHub download links or assumptions about their expiration.
