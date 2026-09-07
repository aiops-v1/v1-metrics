# Metrics & Alerting (Prometheus + Alertmanager + Grafana)

Prometheus scraping the app + a few sidecar exporters, Alertmanager routing
firing alerts to email + Slack, and two Grafana dashboards built from that
data. No traces, no log shipping — just: where does a metric come from,
what shape is it, how do you turn it into a number worth looking at, and
what does it take to get paged about it.

## What's running

The app repos (`../expense-backend-v1.1`, `../expense-frontend-v1.1`,
`../expense-mysql-v1` — siblings of this folder) are unmodified; this
folder only adds the observability layer around them.

- **Prometheus** — scrapes the backend's own `/metrics`, `mysqld-exporter`
  (MySQL internals), `nginx-exporter` (nginx's `stub_status`),
  `nginx-module-vts` (request-duration histograms straight from nginx),
  `node-exporter` (host CPU/mem/disk), and `cadvisor` (per-container
  CPU/mem)
- **Alertmanager** — receives firing alerts from Prometheus, routes them to
  email + Slack (`alertmanager/alertmanager.yml`; one-time credential setup
  in `alertmanager/secrets/README.md`)
- **Grafana** — two dashboards, provisioned automatically on startup:
  `Expense Tracker — v1 Metrics` (RED/business/USE panels + a live
  all-alerts table) and `Expense Tracker — v1 SLO & Burn Rate` (error
  budget, burn rate vs. thresholds)

Two layers of alerting:
- **Plain thresholds** (`prometheus/alert-rules.yml`) — `up==0`, a flat 5%
  error rate, p95 latency over 1s, host CPU over 80%, host memory over 70%.
- **SLO burn-rate alerts** (`prometheus/slo-rules.yml` +
  `slo-burn-rate-alerts.yml`) — error budgets and burn-rate math instead of
  a fixed line. Covered in its own section below.

## Quick start

```bash
cd expense-app-stages/v1-metrics
docker compose up -d --build
```

- App: `http://<host>/` (port 80, via nginx)
- Prometheus: `http://<host>:9090`
- Alertmanager: `http://<host>:9093`
- Grafana: `http://<host>:3000` (`admin` / `admin`)

Alerting needs a one-time credential setup before it works — see
`alertmanager/secrets/README.md`. Until `smtp_password` and
`slack_webhook_url` exist there, the `alertmanager` container exits right
after starting (`docker compose logs alertmanager` shows why). Prometheus
itself is unaffected — it still evaluates rules and shows
Inactive/Pending/Firing on its own `/alerts` page regardless.

Generate some traffic first — every panel is empty on a cold start:

```bash
./scripts/healthy-load.sh   # random signups + expenses, through nginx only
./scripts/fault-load.sh     # real 4xx/5xx traffic
```

See [`scripts/README.md`](scripts/README.md) for details.

---

## Dashboard panels — the query, and what it's teaching

### Row: RED metrics — backend
Rate, Errors, Duration — the three questions to ask of any request-driven
service.

**Request rate by route**
```promql
sum(rate(http_requests_total[5m])) by (route)
```
A counter only ever goes up, so `rate()` turns it into a per-second value —
that's what actually plots as a useful line. `sum(...) by (route)` collapses
everything else down to one line per endpoint.

**5xx error rate by route**
```promql
sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (route)
```
Same query, just filtered to 5xx first.

**p50 / p95 / p99 request duration by method & route — three panels**
```promql
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, method, route))
```
(same shape for 0.50 and 0.95). `http_request_duration_seconds` is a
histogram — several counters, one per bucket boundary — and
`histogram_quantile()` estimates one percentile per call, hence three
panels, not one. `le` must always stay in the `by (...)` list — drop it and
the query silently returns nothing.

**Why three separate panels**: p50 is the typical request, p99 is the tail.
Once each line is also split by method+route, cramming all three quantiles
onto one panel gets unreadable fast.

### Row: Business metrics
Four counters the backend increments explicitly (`src/metrics.js`):
`users_registered_total`, `user_logins_total`, `expenses_created_total`,
`expense_amount_rupees_total`.
```promql
sum(users_registered_total)                               # total, no rate() — a Stat panel wants the raw number
sum(rate(expenses_created_total[5m])) by (category_name)   # speed, by category
```

### Row: MySQL (mysqld_exporter)
`mysqld_exporter` logs into MySQL and re-exposes its internal status
variables as Prometheus metrics — Prometheus can't read MySQL directly, an
exporter is the translation layer.
```promql
mysql_up                                                                             # gauge
mysql_global_status_threads_connected / mysql_global_variables_max_connections * 100  # gauge ÷ gauge
rate(mysql_global_status_slow_queries[5m])                                           # counter → rate
```

### Row: nginx (nginx-prometheus-exporter)
Same exporter pattern, reading nginx's `stub_status`.
```promql
rate(nginx_http_requests_total[5m])
nginx_connections_active
```

### Row: Host & container — the USE method
Utilization, Saturation, Errors — the mental model for a *resource*
(RED is for request-driven *services*).
```promql
100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])))    # host CPU
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 # host memory
sum(rate(container_cpu_usage_seconds_total{image=~".*v1-metrics-(backend|frontend|mysql).*"}[5m])) by (image)  # per-container CPU
```
The container query filters on `image`, not `name` — this host's cAdvisor
reads through containerd, which doesn't expose Docker's `--name` label (see
the `cadvisor` service comments in `docker-compose.yml`). If this folder is
ever renamed off `v1-metrics`, that regex needs to change with it.

### Row: Scrape target health
```promql
up
```
`1` if Prometheus reached the target's last scrape, `0` if not. Worth
demonstrating live: kill MySQL
(`docker compose exec mysql mysqladmin shutdown -u root -prootpass`) without
touching the backend — `up{job="expense-backend"}` stays `1` the whole time
even though every request is now failing. "Up" only means reachable, not
healthy. Restart with `docker compose up -d mysql`.

### Row: Alerting — Inactive / Pending / Firing
```promql
ALERTS
```
Any rule that stops being false shows up as a series on `ALERTS{alertname,
alertstate}` — no separate alerting database. `alertstate` is `pending`
until the condition has held for the rule's `for:` duration, then `firing`.
No series at all while inactive — an empty table is healthy, not broken.

`for:` exists so one noisy sample doesn't fire an alert instantly — the
condition has to stay true across multiple scrapes first.

Prometheus only evaluates rules; it never sends anything itself.
`alerting.alertmanagers` in `prometheus.yml` points it at Alertmanager,
which does the actual routing (`alertmanager/alertmanager.yml`).

---

## SLO burn-rate alerting — a second, deeper layer on the same data

Everything above is a threshold. This is the other kind: alerting on *rate
of error-budget consumption*, on the `Expense Tracker — v1 SLO & Burn Rate`
dashboard. Every SLI here comes from the backend's own metrics only.

### SLA vs. SLO
- **Availability** — SLA 99% (external) vs. SLO 99.5% (internal, what this
  stack alerts on) — stricter on purpose, so breaching the SLO is an early
  warning before the SLA itself is at risk.
- **Latency** — a single target, 200ms p99, no separate SLA.
- **Window: 30 days** — the standard rolling accounting period.

### SLI and error budget
```promql
sli:availability:ratio_rate1h    # fraction of requests, last 1h, that were NOT a 5xx
sli:availability:ratio_rate30d   # same, over 30 days
sli:latency:p99_seconds_rate1h   # estimated p99 duration, last 1h
sli:latency:p99_seconds_rate30d  # same, over 30 days
```
Error budget = `1 - SLO`. At 99.5% availability, the budget is the other
0.5%. That turns "is 99.3% bad?" into arithmetic: you've already spent 0.7
of a 0.5-point budget — over budget, not just under 100%.

### Burn-rate alert
```promql
(1 - sli:availability:ratio_rate1h) / 0.005 > 14
and
(1 - sli:availability:ratio_rate30d) / 0.005 > 14
```
Dividing the current error rate by the budget gives "how many multiples of
the sustainable rate we're failing at right now." A burn rate of 14 would
exhaust the whole 30-day budget in about 51 hours — worth paging over
(`AvailabilitySLOBurnRate`).

> Pairing a 1h window with the *full* 30d window (instead of a shorter
> medium window, the textbook pattern) only stays reactive because this
> Prometheus has no data volume — its history resets on every
> `--force-recreate`. The longer a container has been running uninterrupted,
> the less reactive the 30d side gets. Full reasoning in
> `slo-burn-rate-alerts.yml`'s own comments.

### Latency alert — a threshold, not burn-rate math
```promql
sli:latency:p99_seconds_rate1h > 0.2
and
sli:latency:p99_seconds_rate30d > 0.2
```
Availability is naturally a ratio, so it converts cleanly into a budget. A
p99 duration doesn't — there's no bucket boundary at exactly 200ms to turn
into a "% of requests under the line," so `LatencyP99High` is a direct
threshold instead.

This SLI is one number across every route combined, so it can't say *which*
endpoint is slow. The alert's own `description` runs a one-off `topk(1,...)`
query at notification time and names the worst route directly in the
Slack/email text (e.g. `Slowest endpoint right now: POST /auth/signup (p99
497ms)`) — reliably `/auth/signup` or `/auth/signin`, since bcrypt
(`BCRYPT_ROUNDS = 12`) makes those the slowest routes by a wide margin.

### Demoing it without burning a month's budget
```bash
./scripts/fault-load.sh   # 1-2 minutes alongside healthy-load.sh is enough to trip AvailabilitySLOBurnRate
docker compose up -d --force-recreate prometheus   # reset the 30-day baseline before repeating the demo
```
Recreating Prometheus wipes its TSDB (no named volume backs it) — it
doesn't touch the app itself. Skip that reset if you want to show what a
dented error budget looks like afterward instead.

### Two alert tables
The metrics dashboard's table queries bare `ALERTS` (everything). The SLO
dashboard's table queries `ALERTS{slo=~".+"}` (only burn-rate alerts) —
deliberate scoping, not an accidental filter.

### SLO dashboard panels
Every 1h/30d pair gets two separate panels rather than sharing one axis — a
1h and 30d ratio can sit fractions of a percent apart, and squeezed onto one
auto-scaled axis that looks like a dramatic jump that isn't really there.

- **Error budget remaining** — `(sli:availability:ratio_rate30d - 0.995) /
  (1 - 0.995) * 100`. 100% = untouched, 0% = exactly on target, negative =
  over budget.
- **Burn rate vs. threshold** (1h, 30d) — each window's burn rate, with a
  reference line at 14.
- **Availability SLI** (1h, 30d) — the raw ratio.
- **Latency SLI** (1h, 30d) — estimated p99, with a reference line at 0.2s.

### What's still missing
No automated response to a firing SLO alert — that's separate work.
