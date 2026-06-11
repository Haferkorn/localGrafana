# local-observability

Local Tempo + Loki + Prometheus + Grafana stack for testing OpenTelemetry-instrumented services during development. Mirrors the Grafana Cloud setup we run in tst/prd, so what you see locally is close to what you get there.

Single OTLP entry point (the OTel Collector on `localhost:4318`) fans out to:

| Backend | Signal | Grafana datasource |
|---|---|---|
| Tempo | Traces | `Tempo` |
| Loki | Logs | `Loki` |
| Prometheus | Metrics | `Prometheus` |

## Start the stack

```bash
docker compose up -d
```

Containers persist across reboots; use `docker compose start` to resume after a stop.

## Point a service at it

In each service's `.env`:

```
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
```

## View data

Open http://localhost:3000 — anonymous-admin login, no credentials.

- **Traces**: Explore → datasource `Tempo` → search e.g. `{ resource.service.name = "control-plane" }`
- **Logs**: Explore → datasource `Loki` → query `{service_name="control-plane"}`
- **Metrics**: Explore → datasource `Prometheus`

Service map and trace-to-logs/metrics jumps are pre-wired between the datasources, so you can pivot from a span to its logs and back.

## Multi-service tracing

Set `OTEL_EXPORTER_OTLP_ENDPOINT` on every service that should appear in the trace, then exercise a flow that crosses them. Each service emits spans to the same Tempo instance, all stitched together by the W3C `traceparent` header that the Go SDK propagates automatically. In Grafana you'll see one connected trace covering every hop.

## Dashboards

Local Grafana auto-loads any JSON dashboard placed under `grafana/dashboards/`. Same JSON file works in production: each service repo (e.g. `control-plane/dashboards/`) already commits dashboard JSON and applies it to Grafana Cloud via Terraform's `grafana_dashboard` resource. So one JSON = both local Grafana and prod Grafana.

### Workflow: design once, ship to prod

1. `docker compose up -d` (or already running).
2. Open http://localhost:3000 → build/edit your dashboard in the UI.
3. **Share** → **Export** → **Save to file** → you get a JSON file.
4. Commit it to the right repo:
   - **Cross-cutting** (e.g. "distributed-trace overview" that spans services): drop into `grafana/dashboards/` here in `local-observability`.
   - **Service-specific** (e.g. CP-only metrics): drop into that service's existing `dashboards/` directory next to its Terraform.
5. **Locally**: appears in Grafana within ~5 seconds (live reload).
6. **In prod**: the existing `grafana_dashboard` Terraform resource picks it up on next apply.

### Editing existing dashboards

Edits made in the local UI persist back to the JSON file on disk (`allowUiUpdates: true`). So tweak in the UI, then `git diff grafana/dashboards/` to see what changed. Don't forget to commit.

### Subdirectories become folders

The provisioning config respects directory structure: `grafana/dashboards/observability/foo.json` lands in a Grafana folder called `observability`. Useful if you accumulate many dashboards.

## Stop / wipe

```bash
docker compose stop          # pause, keep all data
docker compose down          # remove containers, keep data volumes
docker compose down -v       # nuke everything including data
```

## Stack reference

| Service | Host port | Purpose |
|---|---|---|
| otel-collector | 4317 (gRPC), 4318 (HTTP) | OTLP receiver, fans out |
| tempo | (internal) | Traces backend |
| loki | (internal) | Logs backend |
| prometheus | (internal) | Metrics backend |
| grafana | 3000 | UI |

Backends only listen on the internal compose network — services never talk to them directly, only to the collector. That mirrors the production topology where services export to a sidecar/extension collector and only the collector talks to Grafana Cloud.

## Troubleshooting

- **`docker compose up` fails on port 4317/4318/3000**: another OTel/Grafana stack is running on the host. `lsof -i :4318` to find it.
- **Service starts but Grafana shows nothing**: check the service logs for `OTEL_EXPORTER_OTLP_ENDPOINT not set; OpenTelemetry export is disabled` — the env var didn't propagate.
- **Collector logs `connection refused` to a backend**: the backend container may still be starting. `docker compose logs <backend>` to confirm; usually self-resolves within seconds.
- **`docker compose ps` shows Loki as unhealthy / `/ready` returns 503**: harmless — Loki's compactor reports unready for ~10 minutes after start while the ring stabilises in single-instance mode. Writes and queries work normally during this window.
