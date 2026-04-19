# cvt-plugin-rest

A REST schema-registry backend for [CVT](https://github.com/sahina/cvt).
Implements the `RegistryProvider` plugin contract so CVT can fetch
OpenAPI schemas from any HTTP registry and report consumer usage back
to it.

## What it does

CVT's built-in schema loader handles files and raw URLs, but many
organizations run a **Central API Registry** — a service that hosts
OpenAPI specs with versioning, access control, and consumer tracking
(Apicurio, Backstage, SwaggerHub, or a homegrown REST API). This plugin
bridges CVT to that registry over HTTP.

Concretely:

- When `cvt validate` needs a schema by ID, CVT calls this plugin's
  `FetchSchema`, which translates the call into `GET {base_url}/schemas/{id}/versions/{version}/spec`
  and returns the raw OpenAPI bytes. The plugin respects the registry's
  `latest` version alias and surfaces the resolved version via the
  `X-Schema-Version` response header.
- When a consumer runs `cvt validate` in CI, CVT calls `RegisterConsumerUsage`,
  which translates to `POST {base_url}/schemas/{id}/consumers` with a
  JSON body describing which consumer hit which endpoints. Your
  registry can use that feed to power "who depends on this schema?"
  answers, gate deployments, or fill out a contract map.

The plugin is a thin, opinionated HTTP client. No auth flows beyond an
optional bearer token. No caching. Retry policy is one try on 5xx, then
surface the failure.

## Install

```sh
git clone https://github.com/sahina/cvt-plugin-rest
cd cvt-plugin-rest
go build -o cvt-plugin-rest .
cvt plugins install ./cvt-plugin-rest
```

This copies the binary to `~/.cvt/plugins/` and records its SHA256 in
`~/.cvt/plugins/state.json`.

## Config

Add to `~/.cvt/config.yaml`:

```yaml
plugins:
  registry:
    binary: ~/.cvt/plugins/cvt-plugin-rest
    timeout: 5s
    on_error: fail_closed
    secrets: [token]
    config:
      base_url: https://registry.example.com/api/v1
      token: ${CVT_REGISTRY_TOKEN}
hooks:
  fetch_schema: registry
  register_consumer_usage: registry
```

| Config key | Required | Description |
|---|---|---|
| `base_url` | yes | Registry API base URL. |
| `token` | no | Bearer token sent as `Authorization: Bearer <token>`. Declare in `secrets:` so it's delivered via gRPC (not env) and redacted from logs. |

Then run `CVT_REGISTRY_TOKEN=... cvt serve` (or `cvt validate --schema
<id>` — the plugin forks at CLI startup too).

## REST contract your registry must implement

- `GET {base_url}/schemas/{schemaId}/versions/{version}/spec`
  - `version = "latest"` must resolve to the current active version.
  - Response body: raw OpenAPI (JSON or YAML). Content-Type respected.
  - Optional response header `X-Schema-Version`: the resolved version
    string (useful when `latest` was requested).
  - `404` → plugin returns gRPC `NotFound`. CVT treats as schema
    resolution failure.

- `POST {base_url}/schemas/{schemaId}/consumers`
  - JSON body:
    ```json
    {
      "consumerId": "order-service",
      "schemaVersion": "2.1.0",
      "environment": "ci",
      "endpoints": [{"method": "GET", "path": "/pets/{id}"}]
    }
    ```
  - MUST be idempotent upsert. Same `(schemaId, consumerId, environment)`
    tuple replaces the prior record rather than stacking duplicates.
  - `2xx` → success. Anything else is surfaced per the status-code
    table below.

## Quick test against a fake registry

Don't have a real registry yet? A tiny Python HTTP server is enough to
prove the wiring works end-to-end:

```sh
# Terminal 1: toy registry that returns a minimal OpenAPI spec
python3 -c '
import http.server
class H(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200); self.send_header("X-Schema-Version","1.0.0"); self.end_headers()
        self.wfile.write(b"openapi: 3.0.0\ninfo: {title: demo, version: 1.0.0}\npaths: {}\n")
    def do_POST(self):
        self.send_response(204); self.end_headers()
http.server.HTTPServer(("127.0.0.1", 8765), H).serve_forever()
'

# Terminal 2: point CVT at it
# (edit ~/.cvt/config.yaml -> base_url: http://127.0.0.1:8765)
export CVT_REGISTRY_TOKEN=dev
cvt serve
```

The registry terminal logs each `GET /schemas/.../spec` and
`POST /schemas/.../consumers` call as CVT exercises the hooks.

## gRPC status codes

| Code | When |
|---|---|
| `NotFound` | registry returned 404 |
| `InvalidArgument` | registry returned 4xx other than 404; or request missing required fields |
| `Unavailable` | registry unreachable or 5xx |
| `FailedPrecondition` | `base_url` not configured |

## Releases

Tagged releases track a compatible CVT version range. `v0.1.x` targets
CVT `v0.6.x`. Breaking changes bump minor.

## See also

- [CVT plugin system overview](https://github.com/sahina/cvt/blob/main/docs/plugins/README.md)
- [Plugin config schema](https://github.com/sahina/cvt/blob/main/docs/plugins/config.md)
- [Authoring a plugin](https://github.com/sahina/cvt/blob/main/docs/plugins/authoring-go.md)
- Sibling plugin: [cvt-plugin-slack](https://github.com/sahina/cvt-plugin-slack)

---
Extracted from github.com/sahina/cvt monorepo at commit 628f1d9924bb897faa20b4022cb2836e084300a1 on 2026-04-19.
