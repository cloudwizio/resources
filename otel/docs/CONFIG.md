# OTel Collector — Config Walkthrough

This doc explains *what* each block of `config/otel-collector.mavvrik.yaml` (the default config) does and *why*. If you've never used an OpenTelemetry Collector before, start with the [Concept primer](#concept-primer). If you have, skip to [Walkthrough](#walkthrough).

## Concept primer

An OpenTelemetry Collector is a small server that sits between your applications and your observability backends. It's configured with one YAML file that defines four kinds of things:

- **Receivers** accept telemetry. The most common is `otlp`, which listens for traces, metrics, or logs over HTTP or gRPC. Your application's OTel SDK posts data to a receiver.
- **Processors** transform telemetry as it flows through — drop spans matching a filter, attach resource attributes, batch spans, redact PII.
- **Exporters** send telemetry to a backend. One exporter per destination: `otlp_http` for any OTLP-compatible backend, `datadog` for Datadog, and so on.
- **Pipelines** wire receivers → processors → exporters together. A pipeline is declared under `service.pipelines`. Components are declared once at the top level and referenced from pipelines by name.

**The `forward` connector — the key to fanout.** A *connector* acts as an exporter in one pipeline and a receiver in another. The `forward` connector simply emits whatever it receives. By exporting from an "ingest" pipeline into `forward`, and then declaring two more pipelines that *receive* from `forward`, you get a fanout:

```
[traces]           (ingest pipeline)
   receivers: otlp                ─► accept incoming traces
   processors: limits, filters    ─► clean them up
   exporters: forward             ─► hand off to fanout pipelines
                              │
                ┌─────────────┴─────────────┐
                ▼                           ▼
[traces/mavvrik]              [traces/thirdparty]
   receivers: forward             receivers: forward
   exporters: otlp_http (mvk)     exporters: otlp_http / datadog / …
```

This pattern lets you keep one set of ingest-time processing (auth, filtering, batching) and apply per-destination logic (different endpoints, different headers, different retry settings) without duplicating the receiver or the processors.

## Walkthrough

The default config — `config/otel-collector.mavvrik.yaml` — has eleven distinct blocks. We'll go through each one.

### `receivers.otlp.http`

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
        max_request_body_size: 33554432
        include_metadata: true
```

Listens for OTLP/HTTP on TCP port `4318`. **`0.0.0.0`** (not `localhost`) is required inside Docker — `localhost` would only accept traffic from inside the container, not from the host. `max_request_body_size` caps a single request at 32 MiB; raise it only if you see "request body too large" errors. **`include_metadata: true`** preserves HTTP request headers downstream so the `headers_setter` extension below can read each incoming request's `X-Agent-ID`.

### `processors.memory_limiter`

```yaml
memory_limiter:
  check_interval: 1s
  limit_percentage: 75
  spike_limit_percentage: 10
```

Refuses incoming spans when the collector's RSS exceeds 75 % of the container memory limit; allows a 10 % spike on top of that. This is the **single most important processor for production** — without it, a traffic spike can OOM-kill the collector. The percentage form means the same config works whether the container has 256 MiB or 4 GiB.

### `processors.resourcedetection`

```yaml
resourcedetection:
  detectors: [env, system]
  timeout: 5s
```

Adds resource attributes to every span. `env` reads `OTEL_RESOURCE_ATTRIBUTES` from the collector's environment. `system` adds hostname and OS info. Both are no-ops if their data isn't present, so this is safe everywhere.

### `processors.filter/llm`

```yaml
filter/llm:
  error_mode: ignore
  trace_conditions:
    - >
      span.attributes["gen_ai.system"] == nil and
      span.attributes["gen_ai.provider"] == nil and
      span.attributes["gen_ai.request.model"] == nil and
      span.attributes["gen_ai.response.model"] == nil and
      span.attributes["llm.system"] == nil and
      span.attributes["llm.model_name"] == nil and
      span.attributes["mvk.model_provider"] == nil
```

The `filterprocessor` drops spans for which a condition evaluates to *true*. Our condition is "**none** of these LLM-identifying attributes are present" — so non-LLM spans are dropped, and LLM spans pass through. The attributes cover three conventions:

- **`gen_ai.*`** — the official OpenTelemetry GenAI semantic conventions.
- **`llm.*`** — OpenInference's legacy LLM attributes.
- **`mvk.*`** — attributes set by the Mavvrik SDK auto-instrumentation.

`error_mode: ignore` means a malformed span (missing the attribute structure entirely) is left alone rather than crashing the collector.

**Where this filter runs.** Every shipped config places `filter/llm` in the `traces/mavvrik` pipeline only — i.e., **non-LLM spans are dropped only on the Mavvrik leg, not on the third-party leg**. Customers using this collector for general service telemetry get full traces in their third-party backend (Datadog, Grafana, …) and LLM-only traces in Mavvrik. If instead you want LLM-only spans for *both* destinations, move `filter/llm` from the `traces/mavvrik` pipeline into the ingest `traces` pipeline so it runs before the `forward` connector.

### `connectors.forward`

```yaml
connectors:
  forward: {}
```

Empty config is correct — the `forward` connector takes no parameters. It accepts spans as if it were an exporter and re-emits them as if it were a receiver. Two pipelines can both declare it as their `receivers:` source.

### `exporters.otlp_http/mavvrik`

```yaml
otlp_http/mavvrik:
  endpoint: https://ingest.mavvrik.ai
  compression: gzip
  encoding: json
  timeout: 10s
  auth:
    authenticator: headers_setter
  retry_on_failure:
    enabled: true
    initial_interval: 5s
    max_interval: 30s
    max_elapsed_time: 120s
  sending_queue:
    enabled: true
    num_consumers: 10
    queue_size: 5000
```

The Mavvrik destination.

- **`endpoint`** is a base URL — the exporter appends `/v1/traces`. You'd write `https://ingest.mavvrik.ai/v1/traces` only if you wanted to double up the path (don't).
- **`encoding: json`** — Mavvrik's ingest accepts both JSON and protobuf OTLP. JSON makes packet captures readable.
- **`compression: gzip`** — substantially reduces egress for typical LLM traces (large prompt strings compress well). It's the default for `otlp_http`; we keep it explicit.
- **Headers come from the `headers_setter` extension** (see below) rather than a static `headers:` block. The extension blends operator-managed values (`Authorization`, `X-Tenant-ID` from env) with per-request forwarding (`X-Agent-ID` from incoming OTLP context, falling back to `MVK_AGENT_ID`). This is what lets one collector serve a fleet of agents under a single Mavvrik tenant while keeping secrets out of source SDKs.
- **`retry_on_failure`** retries failed batches for up to 2 minutes with exponential backoff.
- **`sending_queue`** buffers up to 5,000 batches in memory if the exporter is slower than the receiver. Raise `queue_size` for higher throughput; raise `num_consumers` for higher parallelism.

> **Note about deprecated naming.** Older configs used the key `otlphttp` (no underscore). That still works in `0.152.1` but produces a deprecation warning and will be removed. All shipped configs in this folder use `otlp_http`.

### `exporters.otlp_http/thirdparty`

Same shape as the Mavvrik exporter, pointed at a generic OTLP endpoint via env vars. See the [main README's "Switch backends"](../README.md#switch-backends) section for vendor-specific variants (`datadog/thirdparty`, `basicauth/grafana`, etc.).

### `extensions.health_check`

```yaml
extensions:
  health_check:
    endpoint: 0.0.0.0:13133
    path: /health
```

Exposes a `GET /health` endpoint on port `13133`. A healthy collector returns HTTP 200 with body `{"status":"Server available","upSince":"<RFC3339>","uptime":"<duration>"}`. Use it for container-orchestrator probes and ad-hoc smoke tests.

At startup you'll see `Alpha component. May change in the future.` for this extension — it's been labelled Alpha for years but is the de-facto standard. Treat the log line as informational.

### `extensions.headers_setter`

```yaml
extensions:
  headers_setter:
    headers:
      - action: upsert
        key: Authorization
        value: Bearer ${env:MVK_API_KEY}
      - action: upsert
        key: X-Tenant-ID
        value: ${env:MVK_TENANT_ID}
      - action: upsert
        key: X-Agent-ID
        from_context: x-agent-id
        default_value: ${env:MVK_AGENT_ID}
```

Builds the outbound Mavvrik headers per request. Two distinct modes coexist in the same extension:

- **Static (`value:`)** — `Authorization` and `X-Tenant-ID` come from the collector's env. Operator-managed; one place to rotate secrets.
- **Dynamic (`from_context:`) with `default_value:`** — `X-Agent-ID` is forwarded from each incoming OTLP request's `X-Agent-ID` HTTP header. If a particular request doesn't carry that header, `default_value` (`${env:MVK_AGENT_ID}`) is used instead.

**Why this matters.** A single collector deployed behind a fleet of applications can preserve each application's agent identity all the way to Mavvrik — without putting `MVK_API_KEY` on every application's machine. Each application's SDK just needs to set `X-Agent-ID` on its outbound OTLP requests (`OTEL_EXPORTER_OTLP_HEADERS="x-agent-id=<this-app's-uuid>"` for OTel SDKs; automatic in the Mavvrik SDK's `COLLECTOR` mode).

**Batching caveat.** If you add a `batch` processor for higher throughput, configure `metadata_keys: [x-agent-id]` on it. Without that, the batcher can merge spans from different agents into one outbound payload — and that payload can only carry one `X-Agent-ID`, so the others get lost. See [PRODUCTION.md → Explicit batching](./PRODUCTION.md#explicit-batching).

The receiver above sets `include_metadata: true` so the request headers reach this extension. Without that flag `from_context` would always fall back to `default_value`.

### `service.pipelines`

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection]
      exporters: [forward]
    traces/mavvrik:
      receivers: [forward]
      processors: [filter/llm]
      exporters: [otlp_http/mavvrik]
    traces/thirdparty:
      receivers: [forward]
      exporters: [otlp_http/thirdparty]
```

Three pipelines:

- **`traces`** — the ingest pipeline. Everything that arrives goes through memory limiting and resource enrichment, then into `forward`. No filtering here, so the third-party leg downstream gets every span the SDK sent.
- **`traces/mavvrik`** — reads from `forward`, runs the LLM-only filter, exports to Mavvrik. Non-LLM spans are dropped here only — the third-party leg is unaffected.
- **`traces/thirdparty`** — reads from `forward`, exports to your other backend. All spans (LLM and non-LLM) pass through.

Both fanout pipelines see the same spans coming out of `forward` because `forward` is a connector, not a queue. The per-leg processors are what differentiate the two backends' views.

### `service.telemetry.logs`

```yaml
service:
  telemetry:
    logs:
      level: ${env:LOG_LEVEL}
      encoding: json
```

Controls the **collector's own** logs (not the traces it forwards). Set `LOG_LEVEL=debug` in your `.env` while bringing the collector up; drop to `info` or `warn` in production.

## Choosing defaults — why these numbers

- **`limit_percentage: 75 / spike_limit_percentage: 10`** — Industry-standard defaults; leaves enough headroom for spans in flight without making the limiter trigger-happy. On containers >1 GiB you can raise the limit to 80–85 %.
- **`max_request_body_size: 33554432` (32 MiB)** — Big enough to accept large LLM prompts/responses without truncation. Raise only if you see the receiver reject requests with "request body too large".
- **`timeout: 10s` on exporters** — Conservative; gives the backend time to ack without holding up the queue.
- **`queue_size: 5000`** — At a typical batch of ~10 KB, this buffers ~50 MiB of spans. Raise for bursty workloads.
- **`retry max_elapsed_time: 120s`** — Two minutes of retries is a reasonable balance between resilience and not blocking on a dead backend. Raise for backends with known longer outages.
- **No `batch` processor by default** — The per-exporter `sending_queue` already coalesces network requests. An explicit `batch` processor would add latency and risk losing per-request header context (for the multi-tenant variant). See [PRODUCTION.md](./PRODUCTION.md) if you need explicit batching.
