# Production Hardening

`config/otel-collector.mavvrik.yaml` is production-shaped but conservative. Tune these knobs once you understand your traffic.

## Memory: how the three layers fit together

OTel-recommended practice is to layer three independent memory controls so the collector never OOMs under load. Each layer protects against a different failure mode:

| Layer | Where it lives | What it does |
| --- | --- | --- |
| 1. Container hard memory limit | Orchestrator (`docker run --memory=1g`, K8s `resources.limits.memory: 1Gi`, Cloud Run `memory: 1Gi`) | The kernel kills the container if RSS exceeds this. The other two layers exist to stop the kernel from ever needing to. |
| 2. `GOMEMLIMIT` env var | Set in your `.env` (see `.env.example`) | Go runtime soft ceiling. **Set to ~80 % of layer 1.** Tells the Go GC to work harder before approaching the container limit. The OTel project [strongly recommends](https://github.com/open-telemetry/opentelemetry-collector/blob/main/processor/memorylimiterprocessor/README.md) configuring this on every production collector. |
| 3. `memory_limiter` processor | First processor in every pipeline (already in every shipped config) | Application-level backpressure. Refuses new spans when process memory crosses the configured threshold, giving exporters time to drain. |

Worked example for a 1 GiB container:

| Setting | Value | Rationale |
| --- | --- | --- |
| Container memory limit | `1g` | Set by your orchestrator (Compose, K8s, Cloud Run). |
| `GOMEMLIMIT` | `820MiB` | ~80 % of 1024 MiB. Set as a regular env var via `.env`. |
| `memory_limiter.limit_percentage` | `75` | Refuse new spans when process RSS ≈ 615 MiB (75 % of GOMEMLIMIT). |
| `memory_limiter.spike_limit_percentage` | `10` | Allow a brief +82 MiB spike before refusing. |

`GOMEMLIMIT` is set per-environment (in `.env`), not in the Dockerfile, because the right value depends on the container size your orchestrator gives the collector.

## Memory limiter (processor) — knob detail

```yaml
memory_limiter:
  check_interval: 1s
  limit_percentage: 75
  spike_limit_percentage: 10
```

- Raise `limit_percentage` to 80–85 % if your container has > 1 GiB of memory and you want more headroom for spans in flight.
- Lower `check_interval` to `500ms` if you see brief OOM events between checks.
- The collector **refuses new spans** when memory is high; it does not kill in-flight batches.

## Retries

```yaml
retry_on_failure:
  enabled: true
  initial_interval: 5s
  max_interval: 30s
  max_elapsed_time: 120s
```

- Increase `max_elapsed_time` (e.g. `1800s` = 30 min) for backends with occasional long outages.
- Exponential backoff starts at `initial_interval`, capped at `max_interval`.

**Datadog caveat:** the Datadog exporter accepts the `retry_on_failure` block but internally **disables retries for traces** (to avoid APM-event de-duplication issues). It's parsed without error but has no effect. If you need retry-like behavior, rely on the sending queue.

## Sending queue

```yaml
sending_queue:
  enabled: true
  num_consumers: 10
  queue_size: 5000
```

- `queue_size` is the number of **batches** buffered in memory, not the number of spans. With typical batches of ~10 KB, 5,000 ≈ 50 MiB.
- Raise `num_consumers` if your backend is fast and the queue depth grows under load.
- The queue is in-memory; it does not survive a restart unless you also enable disk persistence (next section).

**Datadog 413 errors:** if Datadog rejects payloads as too large, switch from the plain `sending_queue` to the Datadog-specific batched form:

```yaml
exporters:
  datadog/thirdparty:
    api: { key: ${env:DD_API_KEY}, site: ${env:DD_SITE} }
    sending_queue:
      enabled: true
      batch:
        min_size: 10
        max_size: 100
        flush_timeout: 10s
```

This caps payload sizes at the exporter level.

## Persistent queue (survive restarts and outages)

Add the `file_storage` extension and reference it from the exporter's `sending_queue`:

```yaml
extensions:
  file_storage/mavvrik_queue:
    directory: /var/lib/otelcol/mavvrik
    timeout: 5s
    create_directory: true   # let the extension create the dir if missing

exporters:
  otlp_http/mavvrik:
    sending_queue:
      enabled: true
      num_consumers: 10
      queue_size: 5000
      storage: file_storage/mavvrik_queue
```

Register the extension:

```yaml
service:
  extensions: [health_check, file_storage/mavvrik_queue]
```

**Mount a host directory** at `/var/lib/otelcol/mavvrik` so the queue survives container restarts. The container runs as **UID 10001** (non-root) — the host directory must be writable by that UID or the extension fails:

```bash
mkdir -p ./queue
chown 10001:10001 ./queue
```

Then add to `docker-compose.yml`:

```yaml
services:
  otel-collector:
    volumes:
      - ${COLLECTOR_CONFIG:-./config/otel-collector.mavvrik.yaml}:/etc/otelcol-contrib/config.yaml:ro
      - ./queue:/var/lib/otelcol/mavvrik
```

`./queue` is `.gitignore`-d by default in this folder.

## TLS

`https://ingest.mavvrik.ai` requires no extra TLS configuration — the Go HTTP client in the collector uses the system trust store. If you're behind a corporate proxy with a custom CA, mount your CA bundle and point at it:

```yaml
exporters:
  otlp_http/mavvrik:
    tls:
      ca_file: /etc/ssl/certs/corporate-ca.pem
```

Add a Compose volume to mount the CA bundle into the container.

## Explicit batching

The default config does **not** include a `batch` processor — the per-exporter `sending_queue` already coalesces network requests. If you want explicit batching (e.g., to amortize compression or enforce a maximum batch size), add a `batch` processor:

```yaml
processors:
  batch:
    send_batch_size: 2000
    send_batch_max_size: 5000
    timeout: 3s
    # CRITICAL when X-Agent-ID is forwarded per-request — see below.
    metadata_keys: [x-agent-id]
```

Insert `batch` AFTER `filter/llm` in the ingest pipeline:

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, filter/llm, batch]
      exporters: [forward]
```

### Why `metadata_keys: [x-agent-id]` matters

Every shipped config forwards `X-Agent-ID` from each incoming OTLP request through to Mavvrik (so one collector can serve a fleet of agents). The `batch` processor coalesces spans from multiple incoming requests into one outbound batch — but each outbound batch only carries one set of headers. Without `metadata_keys`, spans from agent A and agent B can land in the same batch and only **one** `X-Agent-ID` value will be sent. The other agent's identity is silently lost.

Setting `metadata_keys: [x-agent-id]` tells the batch processor to split batches by `X-Agent-ID` value — so spans from different source agents never share an outbound payload. Add `authorization` and `x-tenant-id` to the list if you also use the `multitenant.yaml` variant.

### When NOT to add explicit batching

For low-to-moderate throughput, the per-exporter `sending_queue` already does enough coalescing. Explicit `batch` adds latency. Don't reach for it until you see real evidence (network call rate, payload sizes) that it would help.

## Image debugging

The collector image is `FROM scratch` — there is no shell. `docker exec mvk-otel sh` will fail. To debug:

- `docker logs -f mvk-otel` — follow collector logs. Set `LOG_LEVEL=debug` in `.env` to see per-batch detail (number of spans, exporter target, response code).
- `curl http://localhost:13133/health` — confirm the collector is alive.
- `curl http://localhost:4318/v1/traces` with empty `resourceSpans` — confirms the receiver responds (`{"partialSuccess":{}}`).
