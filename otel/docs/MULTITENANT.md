# Multi-tenant Collector — Shared Collector Across Many Mavvrik Tenants

> **You probably don't need this.** Most teams run one collector per Mavvrik tenant with static credentials (the default `config/otel-collector.mavvrik.yaml`). This doc is for platforms that share one collector across multiple Mavvrik tenants — for example, an internal AI gateway used by multiple business units.

## The pattern

When one collector serves multiple tenants, you can't bake credentials into the config. Instead, the collector forwards the `Authorization`, `X-Tenant-ID`, and `X-Agent-ID` headers it receives from each incoming OTLP request through to Mavvrik on a per-request basis.

This requires three changes from the default config (already applied in `config/otel-collector.multitenant.yaml`):

1. **`include_metadata: true`** on the OTLP receiver. Without this, request headers are dropped before downstream components can see them.
2. **A `headers_setter` extension** that copies named headers from the request context onto the outbound exporter request.
3. **`auth: { authenticator: headers_setter }`** on the Mavvrik exporter (replaces the static `headers:` block).

The third-party exporter is unchanged — only the Mavvrik leg uses header forwarding.

## Diff against the default config

**Receiver** — add `include_metadata: true`:

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
        include_metadata: true   # <-- added
```

**Extensions** — add `headers_setter`:

```yaml
extensions:
  health_check: { endpoint: 0.0.0.0:13133, path: /health }

  headers_setter:
    headers:
      - action: upsert
        key: Authorization
        from_context: authorization
      - action: upsert
        key: X-Tenant-ID
        from_context: x-tenant-id
      - action: upsert
        key: X-Agent-ID
        from_context: x-agent-id
```

Context keys (`from_context:`) are lowercase — HTTP/2 lowercases header names on the wire and gRPC metadata matches case-insensitively. Outbound `key:` preserves whatever case you write.

**Register the extension** in `service.extensions`:

```yaml
service:
  extensions: [health_check, headers_setter]
```

If you forget this line, the extension is defined but not initialized, and the auth call fails at runtime.

**Replace static `headers:`** on the Mavvrik exporter with `headers_setter` auth:

```yaml
exporters:
  otlp_http/mavvrik:
    endpoint: https://ingest.mavvrik.ai
    compression: gzip
    encoding: json
    timeout: 10s
    auth:
      authenticator: headers_setter   # <-- replaces the static headers: block
    retry_on_failure: { enabled: true, initial_interval: 5s, max_interval: 30s, max_elapsed_time: 120s }
    sending_queue: { enabled: true, num_consumers: 10, queue_size: 5000 }
```

## The SDK side

Your application's OTel SDK must send the three Mavvrik headers on every OTLP request. For the official OpenTelemetry SDKs, set `OTEL_EXPORTER_OTLP_HEADERS`:

```bash
export OTEL_EXPORTER_OTLP_HEADERS="authorization=Bearer ${MVK_API_KEY},x-tenant-id=${MVK_TENANT_ID},x-agent-id=${MVK_AGENT_ID}"
```

The Mavvrik SDK in COLLECTOR mode handles this automatically — see [agentic/INTEGRATION.md](../../agentic/INTEGRATION.md) § "COLLECTOR Mode".

## Failure mode

If your SDK omits any of the three headers, the collector still POSTs to Mavvrik — but with empty header values — and Mavvrik responds `MISSING_CREDENTIALS`. **The collector itself doesn't surface this; the failure is silent on the collector side and only visible in Mavvrik's response body.** Two things make this easier to diagnose:

1. **Turn on debug logging first.** Set `LOG_LEVEL=debug` in your `.env` and restart. The per-batch log lines from the `otlp_http/mavvrik` exporter will show the response body Mavvrik returned, including the `MISSING_CREDENTIALS` error.
2. **Verify SDK side first.** Before pointing your SDK at the collector, confirm `OTEL_EXPORTER_OTLP_HEADERS` is set in the SDK's environment with all three keys. The minimum is:

   ```bash
   export OTEL_EXPORTER_OTLP_HEADERS="authorization=Bearer <key>,x-tenant-id=<id>,x-agent-id=<id>"
   ```

   For the Mavvrik SDK in COLLECTOR mode, this happens automatically — but if your SDK doesn't, this is the most common multi-tenant onboarding failure.

If you want the collector itself to enforce the contract (drop spans missing required headers rather than forward empties), add an `attributes/extract` processor that lifts the headers to span attributes, followed by a `filter` processor that drops spans where the attributes are missing. That's heavier than the default and adds latency — use it only if SDK discipline is genuinely a problem.

## Per-tenant fan-out (advanced)

Want to route specific tenants to specific backends — e.g., only tenant A's traces should go to a regulated archive? Promote the tenant header to a span attribute and filter on it in a per-tenant pipeline.

```yaml
processors:
  attributes/extract_tenant:
    actions:
      - key: auth.tenant_id
        from_context: x-tenant-id
        action: insert

  filter/only_tenant_a:
    error_mode: ignore
    trace_conditions:
      - span.attributes["auth.tenant_id"] != "tenant-a-id"

  attributes/cleanup_tenant:
    actions:
      - key: auth.tenant_id
        action: delete
```

The `filterprocessor` drops spans whose expressions evaluate to true, so `attributes["auth.tenant_id"] != "tenant-a-id"` keeps only tenant-a spans in that pipeline. The Mavvrik pipeline is unaffected because it doesn't include `filter/only_tenant_a`.

Add a per-tenant pipeline:

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, attributes/extract_tenant, filter/llm]
      exporters: [forward]

    traces/mavvrik:
      receivers: [forward]
      processors: [attributes/cleanup_tenant]
      exporters: [otlp_http/mavvrik]

    traces/tenant_a_archive:
      receivers: [forward]
      processors: [filter/only_tenant_a, attributes/cleanup_tenant]
      exporters: [otlp_http/tenant_a_archive]
```

`attributes/extract_tenant` runs in the ingest pipeline (header is in context there). `attributes/cleanup_tenant` runs at the end of each downstream pipeline so the auth attribute doesn't leak into your backends.
