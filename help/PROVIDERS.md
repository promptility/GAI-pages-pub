# Provider configuration examples

Start with a complete JSON file below, replace its credential/resource/model placeholders, and paste it into the **user** pane of **GAI: Settings**. Each example uses flat presets and the merge rules in [Configuration](https://github.com/promptility/GAI-pages-pub/blob/main/CONFIGURATION.md). The filenames are templates, not credentials; do not commit your edited user settings.

These endpoints were checked against provider documentation. They have not been exercised against paid vendor accounts as part of this repository's offline tests. Availability, permissions, token limits, tool support, and JSON modes remain model-specific.

## Endpoint mapping

GAI sends non-streaming Chat Completions over HTTPS, with `model`, `messages`, `max_tokens`, and `temperature`. A compatible endpoint does not guarantee compatibility with every model. Unsupported extra settings cannot add missing request fields.

| Provider | `open_ai_base` | `open_ai_endpoint` | Example |
|---|---|---|---|
| OpenAI | `api.openai.com` | `/v1/chat/completions` | [openai.json](https://github.com/promptility/GAI-pages-pub/blob/main/examples/openai.json) |
| OpenRouter | `openrouter.ai` | `/api/v1/chat/completions` | [openrouter.json](https://github.com/promptility/GAI-pages-pub/blob/main/examples/openrouter.json) |
| Azure OpenAI v1 | `YOUR_RESOURCE.openai.azure.com` | `/openai/v1/chat/completions` | [azure.json](https://github.com/promptility/GAI-pages-pub/blob/main/examples/azure.json) |
| Gemini OpenAI compatibility | `generativelanguage.googleapis.com` | `/v1beta/openai/chat/completions` | [gemini.json](https://github.com/promptility/GAI-pages-pub/blob/main/examples/gemini.json) |
| Groq | `api.groq.com` | `/openai/v1/chat/completions` | [groq.json](https://github.com/promptility/GAI-pages-pub/blob/main/examples/groq.json) |
| Claude compatibility endpoint | `api.anthropic.com` | `/v1/chat/completions` | [anthropic.json](https://github.com/promptility/GAI-pages-pub/blob/main/examples/anthropic.json) |
| Local compatible server behind TLS | `localhost:8443` | `/v1/chat/completions` | [local_https.json](https://github.com/promptility/GAI-pages-pub/blob/main/examples/local_https.json) |

A hostname may contain a port, but not a scheme or path. Preserve query strings in the endpoint path when a provider requires them.

## OpenAI

The example selects `gpt-4.1-mini`, which documents Chat Completions support. GAI currently sends `max_tokens` rather than `max_completion_tokens`; this prevents treating newer reasoning models as interchangeable replacements. See the model page and Chat Completions reference.

Use an OpenAI API key in `open_ai_key`. ChatGPT subscription credentials do not configure this API client. JSON output can be requested with `response_format` when supported, but Responses API tools such as hosted web search are not exposed by GAI's configuration.

## OpenRouter

Use an OpenRouter key and a provider-qualified model slug, such as `openai/gpt-4.1-mini`. To access Claude through OpenRouter, choose a supported `anthropic/...` slug from your account's model catalog. The endpoint is shared across providers; GAI itself does not need a separate native Anthropic client. See OpenRouter's quickstart and models endpoint.

Provider routing preferences, fallback lists, and optional referral headers are not currently forwarded from arbitrary GAI settings. Pick a model that accepts the fixed request fields and selected tool schemas.

## Azure OpenAI

Replace `YOUR_RESOURCE` and set `model` to your **deployment name**, which need not equal the underlying model name. The example uses Azure's v1 path without an API-version query. GAI sends both bearer and `api-key` headers; use an API key, since automatic Entra token acquisition/renewal is not implemented. See Microsoft's endpoint-switching guide.

For a deployment-based endpoint instead, set the path to `/openai/deployments/YOUR_DEPLOYMENT/chat/completions?api-version=2024-10-21` and select an API version supported by the deployment. Keep the entire query string. This route is documented in Microsoft's deployment example. Do not combine a deployment-based URL with the v1 path.

## Gemini

Use a Gemini API key and a current model identifier available in your account. The example intentionally leaves `YOUR_GEMINI_MODEL` for you to replace. GAI uses Google's OpenAI-compatible route, not the native `generateContent` API. See Google's compatibility documentation.

GAI does not forward Google's `extra_body`, thinking controls, or native multimodal configuration. Check the chosen model's support for legacy `max_tokens`, `temperature`, function calls, and the desired JSON mode.

## Groq

Replace `YOUR_GROQ_CHAT_MODEL` with a model in your Groq account. The host/path follows Groq's OpenAI compatibility guide.

That guide lists `messages[].name` as unsupported. GAI's MCP result messages currently include a `name` field, so MCP rounds on this endpoint may need a normalizing gateway; plain generation and HTTP endpoint tools do not add this field. The offline tests validate protocol mechanics, not vendor-specific acceptance.

## Claude / Anthropic

This template targets Anthropic's OpenAI compatibility endpoint. Use a Claude API key and replace `YOUR_CLAUDE_MODEL`. Anthropic describes the layer as useful for evaluation and documents limitations such as ignored strict function schemas; it is not the native Messages API. See Anthropic's compatibility guide.

Keys spanning multiple workspaces can require `anthropic-workspace-id`, which GAI cannot add. Use a suitable single-workspace credential or an authenticated compatibility gateway. Native thinking, citations, prompt caching, and workspace selection are not exposed. OpenRouter is another route to Claude models using the normal provider configuration above.

## Local models

GAI's model connection uses `HTTPSConnection`. Setting `ssl_verify: false` disables certificate checks; it does **not** make an HTTP-only URL work. Put an OpenAI-compatible local server behind HTTPS, then use [local_https.json](https://github.com/promptility/GAI-pages-pub/blob/main/examples/local_https.json). `open_ai_key` remains a literal string; use the server's token, or a non-empty dummy value if that server accepts unauthenticated compatible requests.

A Caddy reverse proxy can terminate local TLS:

```caddyfile
localhost:8443 {
    reverse_proxy 127.0.0.1:11434
}
```

Set `ssl_verify` to an absolute PEM path for the proxy's CA. Locate the CA through your Caddy installation rather than assuming a cross-platform filesystem path. Caddy's local certificates must be trusted by the client; see automatic HTTPS. The upstream must expose `/v1/chat/completions`; substitute its real port and model identifier.

## Switch providers at runtime

Copy [multiple_providers.json](https://github.com/promptility/GAI-pages-pub/blob/main/examples/multiple_providers.json) and supply both keys. Set `alternates.default` to `openai` or `openrouter` for automatic selection, or remove `default` to show the chooser. Alternate objects contain endpoint/model keys directly, not a nested `oai`.

The example intentionally leaves `input_prio_keys` empty. In this checkout, including endpoint or model keys there makes the resolved base value win **over** the selected alternate. See the phase-dependent [merge rules](https://github.com/promptility/GAI-pages-pub/blob/main/CONFIGURATION.md#merge-rules-and-model-selection).

## Provider model discovery

Use **GAI: Enable provider models ...** to choose one credentialed connection for
this window. Discovery reads existing `oai` and `alternates` connections, plus
`provider_models.connections`; the [complete example](https://github.com/promptility/GAI-pages-pub/blob/main/examples/provider_models.json)
shows the additional connections. Only replace credentials for services you use.
The generated model choices are cached by connection under Sublime's cache
folder and never written into your User settings. See
[configuration](https://github.com/promptility/GAI-pages-pub/blob/main/CONFIGURATION.md#provider-model-discovery) for window behavior,
refresh and filtering rules. Discovery makes read-only catalog requests, not
paid inference requests. The integrations below have offline contract coverage;
account-specific access and vendor inference still require validation.

### Gateways

OpenRouter, Vercel and Requesty use their catalog endpoints and bearer API keys.
Requesty uses the configured regional host, including `router.eu.requesty.ai`.
References: OpenRouter models,
Vercel model discovery,
Requesty API index.

Portkey uses `x-portkey-api-key` for both discovery and inference. Its model IDs
may include an integration slug such as `@my-openai/model`. If your account uses
explicit routing, add an `open_ai_headers` object to the connection, for example
`{"x-portkey-virtual-key": "YOUR_VIRTUAL_KEY"}` or the routing headers specified
by your Portkey integration. The same headers are used for catalog requests.
See Portkey's API specifications. Custom
`open_ai_headers` are also forwarded on other model connections; they must be
string names/values without newlines and should contain only trusted settings.

### Azure / Microsoft Foundry

Azure inference expects a deployment alias in `model`. Discovery lists the
Cognitive Services account's deployed models using Azure Resource Manager,
including pagination. It reads `model_discovery.subscription_id`,
`resource_group`, and `access_token`; `account` defaults to the inference host's
first component. The token needs account deployment read permissions. This is a
management bearer token, distinct from the inference API key in `open_ai_key`.
GAI does not acquire or renew Entra tokens. Refresh the configured token when it
expires. The API-key-only inference endpoint does not itself supply this
management permission.

Use `/openai/v1/chat/completions` for the inference path. Legacy URLs containing
a fixed `/deployments/NAME/` path are unsuitable for changing model deployments.
References: list deployments,
inference endpoints.

### AWS Bedrock

Use `bedrock-mantle.REGION.api.aws` and a Bedrock API key. The Mantle catalog is
always `/v1/models`, even for models whose inference path is `/openai/v1`.
Generated profiles choose `/openai/v1/chat/completions` for proprietary OpenAI GPT
and Google Gemma models, and `/v1/chat/completions` for other compatible models.
Families requiring native Bedrock/Anthropic Messages APIs are excluded. Model
availability and API compatibility remain region-specific.

Runtime's model discovery requires AWS-native APIs and IAM request signing;
GAI's API-key discovery adapter explicitly directs Runtime connections to
Mantle. It does not reinterpret AWS access keys as Bedrock API keys or implement
SigV4, Converse or Messages. References:
Bedrock discovery,
API compatibility.

### Google Cloud / Vertex AI and Gemini

For Vertex AI, configure the project's regional OpenAI-compatible inference
path and put a Google OAuth access token in `open_ai_key`. The discovery adapter
uses the same regional host's publisher Model Garden API, defaulting to publisher
`google`; Gemini text models receive IDs such as `google/gemini-...`. It follows
`nextPageToken`. The token must have permission to list publisher models. GAI
does not acquire or renew Google tokens. Discovery reports catalog candidates,
not a proof of project enablement or quota.

`model_discovery.publisher` can select another publisher, but its models must
support your configured OpenAI-compatible inference endpoint. Native Anthropic,
rawPredict-only models, and arbitrary self-deployed endpoints are not converted
into Chat Completions by discovery. References:
publisher model listing,
OpenAI compatibility,
authentication.

For the separate Gemini Developer API, use
`generativelanguage.googleapis.com`, the `/v1beta/openai/chat/completions` path,
and a Gemini API key with `model_discovery.auth` set to `api_key`. Discovery uses
`/v1beta/models`, selects `generateContent` models, and removes the `models/`
prefix for inference. Google API-key auth uses `x-goog-api-key` for discovery and
inference. See Gemini compatibility.
