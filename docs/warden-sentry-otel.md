# Warden Telemetry to Sentry

The shared Warden GitHub Action exports OpenTelemetry environment variables so Warden runs can send metrics, logs, and traces through the Warden OTLP integration.

## Setup

Set these GitHub Actions values at the org level:

- Secret: `WARDEN_OTLP_ENDPOINT`, for example `https://s.fields.app/api/54/integration/otlp`
- Secret: `WARDEN_OTLP_HEADER`, for example `Authorization=Bearer TOKEN`

The Warden workflow maps those to the variables Warden reads:

```yaml
OTEL_METRICS_EXPORTER: otlp
OTEL_LOGS_EXPORTER: otlp
OTEL_TRACES_EXPORTER: otlp
OTEL_EXPORTER_OTLP_PROTOCOL: http/protobuf
OTEL_EXPORTER_OTLP_ENDPOINT: ${{ secrets.WARDEN_OTLP_ENDPOINT }}
OTEL_EXPORTER_OTLP_HEADERS: ${{ secrets.WARDEN_OTLP_HEADER }}
```

`WARDEN_OTLP_ENDPOINT` is also exported directly into the action environment so Warden itself can use the same integration endpoint if needed.

## What It Enables

- `OTEL_METRICS_EXPORTER=otlp`
- `OTEL_LOGS_EXPORTER=otlp`
- `OTEL_TRACES_EXPORTER=otlp`
- Warden OTLP endpoint for telemetry export
- Warden OTLP auth through `OTEL_EXPORTER_OTLP_HEADERS`
- `OTEL_LOG_TOOL_DETAILS=1` for useful tool and skill audit data

Keep the most sensitive content flags disabled by default:

- `OTEL_LOG_USER_PROMPTS=0`
- `OTEL_LOG_TOOL_CONTENT=0`
- `OTEL_LOG_RAW_API_BODIES=0`

If you explicitly want prompt text or full tool input/output content in Sentry, change the workflow env to:

```sh
export OTEL_LOG_USER_PROMPTS=1
export OTEL_LOG_TOOL_CONTENT=1
```

## Custom Attributes

The workflow sets resource attributes for filtering in Sentry:

```yaml
OTEL_RESOURCE_ATTRIBUTES: service.name=warden,service.namespace=github-actions,deployment.environment=ci,github.repository=${{ github.repository }},github.workflow=${{ github.workflow }}
```

Recommended defaults:

- `deployment.environment=local`
- `service.namespace=github-actions`
- `telemetry.destination=sentry`
