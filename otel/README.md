# Mavvrik OTel Collector — Fanout Setup

Run an OpenTelemetry Collector in Docker that fans out LLM traces to **Mavvrik** and your existing observability backend (Datadog, Grafana Cloud, or any OTLP-compatible service). Three commands and you're done.

> Only sending to Mavvrik? You don't need this folder — use the [Mavvrik SDK in DIRECT mode](../agentic/INTEGRATION.md) instead. It's simpler.

## Architecture

```
                                          ┌─────────────────────────┐
                                     ┌───► │ Mavvrik                 │
                                     │     │ https://ingest.mavvrik.ai│
   ┌──────────────┐    OTLP/HTTP     │     └─────────────────────────┘
   │ Your LLM app ├──────────────┐   │
   │  + OTel SDK  │              │   │     ┌─────────────────────────┐
   └──────────────┘              ▼   │     │ Third-party backend     │
                          ┌──────────┴┐    │  (Datadog, Grafana,     │
                          │  OTel     ├───►│   any OTLP endpoint)    │
                          │  Collector│    └─────────────────────────┘
                          │ (contrib) │
                          └───────────┘
```

The collector receives OTLP traces, applies an LLM-only filter, then fans the same trace out to Mavvrik **and** your third-party backend in parallel.

## Get your Mavvrik credentials

**Read this before the Quick Start.** Every config in this folder (except `otel-collector.multitenant.yaml`) needs three Mavvrik values in your `.env` — without them, the collector starts but ingest rejects every trace with `MISSING_CREDENTIALS` or `INVALID_API_KEY`.

| Env var | What it is | Treat as |
| --- | --- | --- |
| `MVK_AGENT_ID` | UUID identifying this collector's default agent. Used as a **fallback** when an incoming OTLP request doesn't carry an `X-Agent-ID` header. | Public identifier |
| `MVK_API_KEY` | Bearer token authorizing this collector to send traces to Mavvrik. Sent on every outbound request. | **Secret — like a password. Never commit.** |
| `MVK_TENANT_ID` | Your organization's identifier; shared across every agent in your org. Sent on every outbound request. | Public identifier |

> 💡 **Fleet of agents in one tenant?** One collector can serve many SDKs under the same Mavvrik tenant. Each SDK should send its own `X-Agent-ID` header on every OTLP request; the collector forwards it through to Mavvrik so each source's identity is preserved. `MVK_API_KEY` and `MVK_TENANT_ID` stay in the collector's `.env` (operator-managed); `MVK_AGENT_ID` becomes the fallback for any request that doesn't carry the header. See [Multiple agents → one collector](#multiple-agents--one-collector) below for SDK-side setup.

### Register an agent and copy the values

1. **Sign in** to the [Mavvrik Dashboard](https://app.mavvrik.ai). No account yet? [Sign up](https://app.mavvrik.ai/signup) first.

2. **Open the Agents page** — **Admin → Accounts → Agents** — then click **+ Agent**. A right-side drawer opens with two tabs: **Setup** and **Connect**.

3. **Setup tab** — fill in:
   - **Agent ID** — Mavvrik generates a UUID automatically. You can keep it or edit it now. **It cannot be changed after you click Register**, so pick a stable identifier if you're customizing.
   - **Agent Name** — a name that identifies where this collector will run (for example, `prod-otel-eu-west`).
   - **Description** — optional.

   Click **Next**.

4. **Connect tab** — under "Step 2 — Set environment variables" you'll see your three values:
   ```env
   MVK_AGENT_ID=<the UUID from the Setup tab>
   MVK_API_KEY=mvk-agent-…
   MVK_TENANT_ID=<your tenant id>
   ```
   Copy them into `.env` (you'll create that file in the Quick Start by copying from `.env.example`).

   The Connect tab also has a **Download SDK** button — that's for the [Mavvrik SDK integration path](../agentic/INTEGRATION.md). **OTel-collector customers can ignore it.**

5. **Click Register** to save the agent. Until you do this, the API key is inactive and ingest will reject traces with `INVALID_API_KEY`.

6. **(Optional) Verify the credentials** before bringing up the collector:
   ```bash
   set -a; . ./.env; set +a    # load .env into shell (one-shot)
   curl -sS -X POST https://ingest.mavvrik.ai/v1/traces \
     -H 'Content-Type: application/json' \
     -H "Authorization: Bearer ${MVK_API_KEY}" \
     -H "X-Tenant-ID: ${MVK_TENANT_ID}" \
     -H "X-Agent-ID: ${MVK_AGENT_ID}" \
     -d '{"resourceSpans":[]}'
   ```
   Expected: `HTTP 200` + body `{"partialSuccess":{}}`. If you see `MISSING_CREDENTIALS` one header is empty; `INVALID_API_KEY` means the key value is wrong or the agent wasn't Registered yet.

> 💡 **Already have an agent?** Same path — **Admin → Accounts → Agents** — click the 🖍️ icon next to it, open the **Connect** tab, copy the env vars.

> ⚠️ **Never commit `.env`.** The repo's `otel/.gitignore` already excludes it; only `.env.example` (placeholders) is tracked.

**Third-party backend credentials** depend on which backend you're fanning out to — see the per-backend subsections in [Switch backends](#switch-backends) below.

## Quick Start

With your three `MVK_*` values from the previous section in hand:

```bash
cd otel
cp .env.example .env
$EDITOR .env          # paste in MVK_* and your third-party backend's creds
docker compose up
```

That's it. The collector listens on port `4318` for OTLP/HTTP. Health check is on `:13133/health`.

**Expected startup log line:** `Everything is ready. Begin running and processing data.`

**Verify:**

```bash
curl -sS http://localhost:13133/health
```

Expected: `{"status":"Server available","upSince":"...","uptime":"..."}`

**Send a synthetic trace** (in a second terminal):

```bash
curl -sS -X POST http://localhost:4318/v1/traces \
  -H 'Content-Type: application/json' \
  -d '{
    "resourceSpans": [{
      "resource": {"attributes":[{"key":"service.name","value":{"stringValue":"hello-llm"}}]},
      "scopeSpans": [{"scope":{"name":"manual"},"spans":[{
        "traceId":"5b8aa5a2d2c872e8321cf37308d69df2",
        "spanId":"051581bf3cb55c13",
        "name":"hello-completion",
        "kind":1,
        "startTimeUnixNano":"1700000000000000000",
        "endTimeUnixNano":"1700000001000000000",
        "attributes":[
          {"key":"gen_ai.system","value":{"stringValue":"openai"}},
          {"key":"gen_ai.request.model","value":{"stringValue":"gpt-4o"}}
        ]
      }]}]
    }]
  }'
```

Expected: `{"partialSuccess":{}}` (empty `partialSuccess` = full success).

The trace should appear in both backends within a few seconds. In the Mavvrik dashboard, look under **Home → Agentic → Cost** (spend over time) or **Home → Agentic → Sessions** (search by `session_id`) under the agent you registered.

## Run without Docker Compose

Compose is the easy path. For Cloud Run, Kubernetes, or any raw Docker host, use `docker run` directly. Two patterns, depending on whether you want to ship a self-contained image or stay on the upstream one.

### Pattern A: build a Mavvrik-branded image (recommended for production)

This uses the [`Dockerfile`](./Dockerfile) in this folder, which bakes a chosen config into the image and applies OCI labels + `GOMEMLIMIT` guidance.

```bash
# Build (default config = Mavvrik + generic OTLP)
docker build -t mvk-otel:0.152.1 .

# Or build a different backend variant without editing the Dockerfile
docker build --build-arg CONFIG=config/otel-collector.datadog.yaml -t mvk-otel:datadog .

# Run
docker run --rm \
  --name mvk-otel \
  -p 4318:4318 -p 13133:13133 \
  --env-file .env \
  mvk-otel:0.152.1
```

For production deployments, set `GOMEMLIMIT` to ~80% of the container's hard memory limit (per the [official OTel guidance](https://github.com/open-telemetry/opentelemetry-collector/blob/main/processor/memorylimiterprocessor/README.md)). Either uncomment `GOMEMLIMIT=` in `.env` or pass it inline:

```bash
docker run --rm \
  --name mvk-otel \
  --memory 1g \
  -e GOMEMLIMIT=820MiB \
  -p 4318:4318 -p 13133:13133 \
  --env-file .env \
  mvk-otel:0.152.1
```

### Pattern B: run the upstream image with a bind-mounted config

No image to build — bind-mount the config file into the upstream `otel/opentelemetry-collector-contrib:0.152.1` image directly. Faster iteration if you're tweaking the config.

```bash
docker run --rm \
  --name mvk-otel \
  -p 4318:4318 -p 13133:13133 \
  --env-file .env \
  -v "$(pwd)/config/otel-collector.mavvrik.yaml:/etc/otelcol-contrib/config.yaml:ro" \
  otel/opentelemetry-collector-contrib:0.152.1
```

To switch backends, change the source path of the `-v` mount to `config/otel-collector.datadog.yaml`, `.grafana.yaml`, or `.multitenant.yaml` and re-run.

### Verifying it works

Same checks as the Compose Quick Start:

```bash
curl -sS http://localhost:13133/health             # {"status":"Server available", ...}
docker logs mvk-otel | grep "Begin running"        # startup banner
docker logs -f mvk-otel                            # follow live
```

To stop:

```bash
docker stop mvk-otel
```

## Switch backends

To send to a different third-party backend, set `COLLECTOR_CONFIG` in `.env` to point at the matching config file, fill in that backend's credentials, then restart.

| Backend | Config file | Required env vars |
| --- | --- | --- |
| Generic OTLP (default) | `./config/otel-collector.mavvrik.yaml` | `THIRDPARTY_OTLP_ENDPOINT`, `THIRDPARTY_OTLP_AUTH` |
| Datadog | `./config/otel-collector.datadog.yaml` | `DD_API_KEY`, `DD_SITE` |
| Grafana Cloud | `./config/otel-collector.grafana.yaml` | `GRAFANA_CLOUD_OTLP_ENDPOINT`, `GRAFANA_CLOUD_INSTANCE_ID`, `GRAFANA_CLOUD_API_KEY` |
| Multi-tenant (shared collector) | `./config/otel-collector.multitenant.yaml` | `THIRDPARTY_OTLP_ENDPOINT`, `THIRDPARTY_OTLP_AUTH` (Mavvrik creds come from incoming SDK request headers) |

### Datadog

Set `DD_API_KEY` and `DD_SITE` in `.env`. Available sites:

```
datadoghq.com (US1, default) | us3.datadoghq.com | us5.datadoghq.com
datadoghq.eu (EU)            | ap1.datadoghq.com (Japan) | ap2.datadoghq.com (Australia)
ddog-gov.com (US1-FED)       | us2.ddog-gov.com (US2-FED)
```

Traces land in **Datadog APM → Traces**. The `gen_ai.*` attributes are preserved as span tags. **Note:** the dedicated **Datadog LLM Observability** product UI (prompt/response display, cost tracking, model groupings) requires Datadog's own LLM Observability SDK — the OTel exporter path doesn't route into LLM Observability automatically. Also set `service.name` on the SDK side (via `OTEL_RESOURCE_ATTRIBUTES`) — otherwise spans appear in APM under `otlpresourcenoservicename` and are hard to find.

### Grafana Cloud

- Find your `GRAFANA_CLOUD_OTLP_ENDPOINT` and `GRAFANA_CLOUD_INSTANCE_ID`: **Grafana Cloud Portal → your stack → OpenTelemetry tile → Configure**.
- Create an Access Policy token with the `traces:write` scope: **Grafana Cloud Portal → Security → Access Policies → Create access policy → Add token**. Use that as `GRAFANA_CLOUD_API_KEY`.

The endpoint MUST be the **base** `/otlp` path (for example `https://otlp-gateway-prod-us-east-0.grafana.net/otlp`). The collector appends `/v1/traces` automatically — if you include it yourself, you'll get a double-path error.

Traces land in **Explore → Tempo** in your Grafana stack.

### Multiple agents → one collector

If several applications (each registered as its own Mavvrik agent) all send to the same collector under **one** Mavvrik tenant — no extra config swap needed. The default `mavvrik`, `datadog`, and `grafana` configs already forward each request's `X-Agent-ID` header through to Mavvrik (with `MVK_AGENT_ID` from `.env` as a fallback). `MVK_API_KEY` and `MVK_TENANT_ID` stay static in the collector's `.env`.

Each application's SDK just needs to set `X-Agent-ID` on outbound OTLP. For OTel SDKs:

```bash
# In each application's environment
export OTEL_EXPORTER_OTLP_HEADERS="x-agent-id=<this-app's-agent-uuid>"
export OTEL_EXPORTER_OTLP_ENDPOINT=http://your-collector:4318
```

For the Mavvrik SDK in COLLECTOR mode, the agent_id configured on the SDK is sent as `X-Agent-ID` automatically.

**What you'll see at the Mavvrik side:** traces from each application show up under their own agent. If an SDK forgets to set the header, those traces fall back to whatever `MVK_AGENT_ID` is configured on the collector — usable, but they'll all be grouped under that one fallback agent.

**Heavy-traffic note.** If you add a `batch` processor for production throughput, set `metadata_keys: [x-agent-id]` on it — otherwise batching can merge spans from different SDKs into one outbound payload and only one `X-Agent-ID` gets sent. See [docs/PRODUCTION.md → Explicit batching](./docs/PRODUCTION.md#explicit-batching).

### Multi-tenant (shared collector)

Use this **only** when one collector serves multiple Mavvrik tenants — for example, an internal AI gateway used by multiple business units, each with their own Mavvrik account. Mavvrik credentials don't come from `.env`; they come from incoming OTLP request headers. Your SDK sets them via `OTEL_EXPORTER_OTLP_HEADERS`:

```bash
export OTEL_EXPORTER_OTLP_HEADERS="authorization=Bearer ${MVK_API_KEY},x-tenant-id=${MVK_TENANT_ID},x-agent-id=${MVK_AGENT_ID}"
```

If your SDK omits any of the three headers, the collector still POSTs to Mavvrik — but with empty values — and Mavvrik responds `MISSING_CREDENTIALS`.

See [docs/MULTITENANT.md](./docs/MULTITENANT.md) for the full pattern.

## Troubleshoot

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Collector exits immediately with `cannot load configuration` | Typo or unknown component in the config | Run `docker run --rm -v $(pwd)/config:/etc/otelcol-contrib otel/opentelemetry-collector-contrib:0.152.1 validate --config=/etc/otelcol-contrib/$(basename your-config.yaml)`. The error names the bad key. |
| Collector exits with `at least one endpoint must be specified` | Required env var is unset in `.env` | Set the missing variable. The collector won't start without it. |
| No traces in Mavvrik dashboard | Wrong creds, or `filter/llm` is dropping your spans | (1) Check the dashboard agent ID matches `MVK_AGENT_ID`. (2) Confirm at least one of `gen_ai.system`, `llm.system`, or `mvk.model_provider` is set on your spans. |
| Mavvrik returns `MISSING_CREDENTIALS` in collector logs (multi-tenant variant) | An incoming OTLP request didn't carry the required headers; `headers_setter` silently forwards an empty header value. The collector itself doesn't reject the request — Mavvrik does. | Set `LOG_LEVEL=debug` to see the failing requests; on the SDK side, set `OTEL_EXPORTER_OTLP_HEADERS="authorization=Bearer <key>,x-tenant-id=<id>,x-agent-id=<id>"` with all three keys. |
| Mavvrik returns `MISSING_CREDENTIALS` only for some spans (single-tenant variants) | A specific SDK is sending without `X-Agent-ID` AND your collector's `MVK_AGENT_ID` env var is unset, so the fallback is empty too. | Set `MVK_AGENT_ID` in `.env`, OR have the SDK set `OTEL_EXPORTER_OTLP_HEADERS="x-agent-id=<this-app's-uuid>"`. |
| `X-Agent-ID` not appearing in outbound requests despite the SDK sending it | Header-case mismatch between SDK and `headers_setter`. OTel normalizes header names to lowercase in the metadata map, so `from_context: x-agent-id` should match regardless of case — but if you've customized `from_context:` to something other than the canonical lowercase, that breaks. | Keep `from_context: x-agent-id` (all lowercase). Set `LOG_LEVEL=debug` and look at the per-batch log lines to confirm the header name the collector received. |
| Mavvrik returns `INVALID_API_KEY` | Bad `MVK_API_KEY` or `MVK_TENANT_ID` | Re-copy from the Mavvrik Dashboard Connect tab. |
| Traces in Mavvrik but not in Datadog | 413 Payload Too Large from Datadog ingest | Add `sending_queue.batch: { min_size: 10, max_size: 100, flush_timeout: 10s }` to the `datadog/thirdparty` exporter to bound payload size. |
| Grafana returns 404 with `path: /v1/traces/v1/traces` | You included `/v1/traces` in `GRAFANA_CLOUD_OTLP_ENDPOINT` | The endpoint must be the `/otlp` base only — the collector appends `/v1/traces`. |
| `docker exec mvk-otel sh` fails | The image is `FROM scratch` and has no shell | Use `docker logs mvk-otel`. Set `LOG_LEVEL=debug` in `.env` for per-batch diagnostics. |
| `file_storage` extension errors on disk write | Container runs as UID `10001`; mount directory is owned by root | `chown 10001:10001 ./queue` on the host, then `docker compose up`. |
| Startup log: `Alpha component. May change in the future.` for `health_check` | Expected — the `health_check` extension is still labeled Alpha but is widely used in production | No action needed. |
| Startup warning about `otlphttp` deprecation | You're using the old `otlphttp:` key in a customized config | Rename to `otlp_http:` (and `otlphttp/<name>` to `otlp_http/<name>`). The old name still works in 0.152.1 but will be removed. |

## Going deeper

- [docs/CONFIG.md](./docs/CONFIG.md) — concept primer and annotated walkthrough of the default config.
- [docs/MULTITENANT.md](./docs/MULTITENANT.md) — running one shared collector across multiple Mavvrik tenants.
- [docs/PRODUCTION.md](./docs/PRODUCTION.md) — memory tuning, persistent queue, TLS, batching.
- [OpenTelemetry Collector — Install with Docker](https://opentelemetry.io/docs/collector/install/docker/)
- [OpenTelemetry Collector — Configuration reference](https://opentelemetry.io/docs/collector/configuration/)
- [Mavvrik SDK integration (DIRECT mode)](../agentic/INTEGRATION.md)
- [Mavvrik Help Center](https://help.mavvrik.ai)

---

Need help? Visit the [Mavvrik Help Center](https://help.mavvrik.ai) or contact your Mavvrik representative.
