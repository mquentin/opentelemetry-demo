# collector-config-demo.sh

Point the demo collector at your own OTLP/HTTP endpoints and stamp custom
resource attributes on the telemetry of one service, without editing any
tracked file.

```sh
./collector-config-demo.sh \
  --resource-attribute <key>=<value> \
  --resource-attribute <key>=<value> \
  --traces-url https://example.com/v1/traces?token=... \
  --logs-url   https://example.com/v1/logs?token=...
```

## Options

| Option | Description |
| --- | --- |
| `--traces-url <url>` | OTLP/HTTP URL for traces. Required. |
| `--logs-url <url>` | OTLP/HTTP URL for logs. Required. |
| `--resource-attribute key=value` | Resource attribute to set. Repeat for as many attributes as needed. |
| `--service <name>` | Service whose telemetry gets the attributes. Defaults to `frontend-web`, the browser app. `all` applies them to every service. |
| `--out <path>` | Where to write the generated config. Defaults to `.collector-config-demo/otelcol-config-generated.yml`. |
| `-- <compose args>` | Run `docker compose` with these arguments instead of recreating the collector, for example `-- up -d`. |

Both URLs are used verbatim through the OTLP/HTTP exporter's `traces_endpoint`
and `logs_endpoint` settings, so a token embedded in the path or the query
string is preserved. The script rejects anything that is not an absolute
`http(s)` URL.

## What it generates

The script writes a collector extras config and points
`OTEL_COLLECTOR_CONFIG_EXTRAS` at it in `.env.override`, which is the seam the
collector already loads last. The generated file contains:

* an OTLP/HTTP exporter per signal, pointing at the two URLs
* a `transform` processor setting the resource attributes, scoped to the
  selected service
* the `traces` and `logs` pipelines, repeating the processors declared in
  `src/otel-collector/otelcol-config.yml` and the exporters declared in
  `src/otel-collector/otelcol-config-observability.yml`, plus the new ones

The pipelines are regenerated from those two files on every run because the
collector *replaces* lists instead of merging them: a pipeline that omits
`debug`, `otlp_grpc/jaeger`, `span_metrics` or `opensearch` silently turns the
local observability stack off. Reading them back means the generated config
follows the upstream files instead of a stale copy.

With no `--resource-attribute` the processor is omitted entirely and only the
exporters are added.

## The generated file must never be committed

It holds your endpoints, including any token embedded in them.
`.collector-config-demo/` is listed in `.gitignore` for that reason. Keep
`--out` inside that directory, or add your own path to `.gitignore` first.

The script also sets `OTEL_COLLECTOR_CONFIG_EXTRAS` in `.env.override`, which
is tracked upstream as a local overrides file and carries its own "do not push"
header. Only the path to the generated file is written there, never an endpoint
or a token, but the file still shows up as modified in `git status`, so leave it
out of your commits. Because the `Makefile` passes `--env-file .env.override`,
`make start` and `make restart` pick the generated config up with no further
setup.

The attributes are applied by the collector, so nothing is shipped to the
browser bundle and no frontend rebuild is needed.

## Verifying it works

```sh
# the exporters are wired and sending
docker compose logs otel-collector | grep -i "your-endpoint-host"

# the attributes landed on the browser telemetry only
curl -s "http://localhost:8080/jaeger/ui/search?service=frontend-web&lookback=15m"
```

A deliberately unroutable host is a quick smoke test: the collector logs
`Exporting failed. Will retry` naming the full URL, which proves the exporter
is live and the token survived.
