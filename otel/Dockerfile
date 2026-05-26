# Mavvrik OTel Collector — production image.
# Pinned to otel/opentelemetry-collector-contrib v0.152.1.
# Bakes in the default config (Mavvrik + generic OTLP). To bake a different
# variant without editing this file:
#   docker build --build-arg CONFIG=config/otel-collector.datadog.yaml -t mvk-otel:datadog .
# For per-deploy config changes, prefer docker-compose.yml (it bind-mounts
# whichever file COLLECTOR_CONFIG points at).

FROM otel/opentelemetry-collector-contrib:0.152.1

# OCI image metadata (https://github.com/opencontainers/image-spec/blob/main/annotations.md)
LABEL org.opencontainers.image.title="Mavvrik OTel Collector"
LABEL org.opencontainers.image.description="OpenTelemetry Collector that fans out LLM traces to Mavvrik and a third-party backend"
LABEL org.opencontainers.image.vendor="Mavvrik"
LABEL org.opencontainers.image.version="0.152.1"
LABEL org.opencontainers.image.source="https://github.com/cloudwizio/resources"
LABEL org.opencontainers.image.documentation="https://github.com/cloudwizio/resources/blob/master/otel/README.md"
LABEL org.opencontainers.image.licenses="Apache-2.0"

# Build-time variant selector. Default is the generic-OTLP config; override via:
#   --build-arg CONFIG=config/otel-collector.datadog.yaml
ARG CONFIG=config/otel-collector.mavvrik.yaml
COPY ${CONFIG} /etc/otelcol-contrib/config.yaml

# LOG_LEVEL Dockerfile-internal default; docker-compose.yml env_file overrides it.
ENV LOG_LEVEL=info

# GOMEMLIMIT is intentionally NOT set here. Official OTel guidance is to set it
# to 80% of the container's hard memory limit at runtime (see
# https://github.com/open-telemetry/opentelemetry-collector/blob/main/processor/memorylimiterprocessor/README.md).
# Set GOMEMLIMIT in your .env (see .env.example) once you know your container size.

# Ports we actively use:
#   4318   OTLP HTTP receiver (primary traffic)
#   13133  health check endpoint (orchestrator probes)
EXPOSE 4318 13133

# Image is FROM scratch — no shell, no curl, so a Dockerfile HEALTHCHECK that
# exec'd a binary would fail. Probe :13133/health from your orchestrator.
HEALTHCHECK NONE

# Base image already sets ENTRYPOINT=["/otelcol-contrib"], CMD with the config
# path, and runs as non-root UID 10001. We inherit all three.
