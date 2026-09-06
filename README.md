# v1 — Metrics (Prometheus + Grafana)

First stage of a four-stage teaching progression:

1. **v1-metrics** (this folder) — Prometheus + Grafana, metrics only
2. v2-traces — adds OTel + Tempo *(not yet built)*
3. v3-logs — adds Loki + Promtail *(not yet built)*
4. v4-rca — adds SLO burn-rate alerting + the RCA agent *(not yet built)*

Each stage is its own standalone `docker-compose.yml`, deliberately not a
toggle on one shared stack — bring one stage down before starting the next
so the story stays "here's what we just added," not "here's a flag."

## What's here, and why only this

The app repos (`../expense-backend-v1`, `../expense-frontend-v1`,
`../expense-mysql-v1` — siblings of this folder inside
`expense-app-stages/`, same content as `../../expense-app/expense-*-v1`)
are shared across every stage in this folder, unmodified per stage — this
stage doesn't fork the app, it just stands up a smaller slice of the
observability stack around it:

- **Prometheus** — scrapes the backend's own `/metrics`, `mysqld-exporter`
  (a sidecar process reading MySQL's internals), `nginx-exporter` (reading
  nginx's `stub_status`), `node-exporter` (host CPU/mem/disk via `/proc`,
  `/sys`), and `cadvisor` (per-container CPU/mem)
- **Grafana** — one dashboard, `Expense Tracker — v1 Metrics`, provisioned
  automatically on startup

**Deliberately not here yet:**
- No traces (no `otel-collector`, no `tempo`) — the backend's OTel SDK is
  wired in already (a later phase of the full build) but is turned off here
  via `ENABLE_TRACING: "false"` in `docker-compose.yml`, so there's nothing
  trying to export spans to a collector that doesn't exist. Arrives in
  v2-traces.
- No log shipping (no `loki`, no `promtail`) — the app still logs structured
  JSON to stdout regardless (`docker compose logs backend` shows it), just
  nothing ships those lines anywhere queryable yet. Arrives in v3-logs.
- No alerting rules, no SLOs, no RCA agent — Prometheus's alerting story
  (Inactive → Pending → Firing) is worth teaching once burn-rate math and
  error budgets are on the table, bundled with the RCA agent in v4-rca
  rather than introduced here as an isolated rule.

## Before you start

This stack uses the same container names and host ports (80, 3000, 9090)
as `../../expense-app/observability` — fine as long as that stack isn't
also running on this host. If you ever do bring both up side by side,
either stack's containers/ports will collide with the other's.

## Quick start

```bash
cd expense-app-stages/v1-metrics
docker compose up -d --build
```

- App: `http://<host>/` (port 80, via nginx)
- Prometheus: `http://<host>:9090`
- Grafana: `http://<host>:3000` (`admin` / `admin`)

Generate some traffic first (sign up, log in, add a few expenses) — every
panel on the dashboard is empty/flat on a cold start, which is itself a
fine thing to point out ("no data yet" is a different, better state than
"broken").

## Verification checklist (maps to `../../teaching/prom.md`)

- [ ] `curl http://<host>:4000/metrics` from inside the `backend` container
  (or `docker compose exec backend wget -qO- localhost:4000/metrics`) shows
  raw Prometheus text — read it before ever opening a UI (Topic 1)
- [ ] Prometheus `/targets` shows all six jobs `UP`: `expense-backend`,
  `mysql`, `nginx-frontend`, `node`, `cadvisor`, `prometheus` (Topic 4)
- [ ] In Prometheus's **Explore/Graph** tab, build up
  `sum(rate(http_requests_total[5m])) by (route)` one function at a time —
  raw counter first, then `rate()`, then `sum() by (...)` (Topic 5)
- [ ] Confirm `node_cpu_seconds_total` is a counter (`rate()` makes sense,
  raw value doesn't) and `node_memory_MemAvailable_bytes` is a gauge (raw
  value makes sense, `rate()` doesn't) (Topic 6)
- [ ] Grafana → Explore → Prometheus: run a query before ever building a
  panel from it (Topic 7)
- [ ] Open the provisioned dashboard, use the `$tier` template variable to
  filter panels that support it (Topic 8)
- [ ] Stop the `backend` container's health without stopping the process —
  e.g. `docker compose exec mysql mysqladmin shutdown -u root -prootpass`
  (backend stays up, its DB calls start failing) — watch `up{job="expense-backend"}`
  stay `1` the whole time while the 5xx panel spikes. **This is the single
  most important demo in this stage**: `up` only means "Prometheus could
  scrape it," never "the app is healthy" (Topic 11)
- [ ] Restart mysql afterwards (`docker compose up -d mysql`) and confirm
  the error panel recovers

## Explicitly out of scope for this stage
- Alerting rules (arrives in v4-rca, alongside SLOs)
- Anything trace- or log-specific (v2-traces, v3-logs)
- Failure-injection tooling (`../../expense-app/observability/inject.sh`) —
  built against `ENABLE_DEBUG_ROUTES`/the `/debug` API, which this stage
  leaves unset (same as `ENABLE_TRACING`, off unless set) since there's no
  reason to teach deliberate failure injection before SLOs/alerting exist
  to react to it; not ported here, arrives conceptually in v4-rca
