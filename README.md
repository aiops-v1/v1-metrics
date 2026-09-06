# Metrics & Alerting (Prometheus + Alertmanager + Grafana)

Prometheus scraping the app + a few sidecar exporters, Alertmanager routing
firing alerts to email + Slack, and two Grafana dashboards built from that
data. No traces, no log shipping — just: where does a metric come from,
what shape is it, how do you turn it into a number worth looking at, and
what does it take to get paged about it. Metrics and alerting stay one
stage here rather than two: both are the same three tools (Prometheus,
Alertmanager, Grafana), just different questions asked of the same data.

## What's running

The app repos (`../expense-backend-v1`, `../expense-frontend-v1`,
`../expense-mysql-v1` — siblings of this folder) are unmodified; this
folder only adds the observability layer around them:

- **Prometheus** — scrapes the backend's own `/metrics`, `mysqld-exporter`
  (a sidecar process reading MySQL's internals), `nginx-exporter` (reading
  nginx's `stub_status`), nginx-module-vts (request-duration histograms,
  straight from nginx — see the SLO section's "Known gap" for why this one
  currently has no data), `node-exporter` (host CPU/mem/disk via `/proc`,
  `/sys`), and `cadvisor` (per-container CPU/mem)
- **Alertmanager** — receives firing alerts from Prometheus, routes them to
  email + Slack (`alertmanager/alertmanager.yml`; one-time credential setup
  in `alertmanager/secrets/README.md`)
- **Grafana** — two dashboards, provisioned automatically on startup:
  `Expense Tracker — v1 Metrics` (RED/business/USE panels + a live
  all-alerts table) and `Expense Tracker — v1 SLO & Burn Rate` (error
  budget, burn rate vs. thresholds, the SLIs underneath)

Two layers of alerting, both live here:
- **Plain thresholds** (`prometheus/alert-rules.yml`) — `up==0`, a flat 5%
  error-rate cutoff, p95 latency over 1s, host CPU over 80%, host memory
  usage over 70%. The simplest thing Prometheus alerting can express.
- **SLO burn-rate alerts** (`prometheus/slo-rules.yml` +
  `slo-burn-rate-alerts.yml`) — built on error budgets and multi-window
  burn-rate math instead of a fixed line. A bigger conceptual jump than a
  threshold, covered in its own section below rather than folded into the
  panel-by-panel walkthrough.

## Quick start

```bash
cd expense-app-stages/v1-metrics
docker compose up -d --build
```

- App: `http://<host>/` (port 80, via nginx)
- Prometheus: `http://<host>:9090`
- Alertmanager: `http://<host>:9093`
- Grafana: `http://<host>:3000` (`admin` / `admin`)

Alerting needs a one-time credential setup before it actually works — see
`alertmanager/secrets/README.md`. Until `smtp_password` and
`slack_webhook_url` exist there (and the placeholder addresses in
`alertmanager/alertmanager.yml` are filled in), the `alertmanager`
container will exit right after starting — it reads both files at startup
and has nothing sensible to fall back to, so this is a loud, contained
failure rather than a silently-broken notification path
(`docker compose logs alertmanager` will show exactly which file it
couldn't read). Prometheus itself is unaffected either way — it still
evaluates the rules and shows Inactive/Pending/Firing on its own `/alerts`
page and the dashboard's alert table regardless of whether Alertmanager is
up.

Generate some traffic first (sign up, log in, add a few expenses) — every
panel on the dashboard is empty/flat on a cold start, which is itself a
fine thing to point out ("no data yet" is a different, better state than
"broken"). `scripts/` has two ready-made load generators for this — see
[`scripts/README.md`](scripts/README.md):

```bash
./scripts/healthy-load.sh   # random signups + expenses, through nginx only
./scripts/fault-load.sh     # real 4xx/5xx traffic, no debug API needed
```

---

## Dashboard panels — the query, and what it's actually teaching

### Row: RED metrics — backend
The RED method (Rate, Errors, Duration) — the three questions to ask of any
request-driven service.

**Request rate by route**
```promql
sum(rate(http_requests_total[5m])) by (route)
```
`http_requests_total` is a **counter** — it only ever goes up (or resets to
0 if the backend restarts). A raw counter value is meaningless on its own
("14,203 requests" — since when?); `rate()` turns 5 minutes of accumulated
count into a per-second average, which is what actually plots as a useful
line. `sum(...) by (route)` then collapses the method/status_code labels
away, leaving one line per route. This is the query to build up one
function at a time live: raw counter (a flat-looking staircase) →
`rate()` (now a real per-second value) → `sum() by (route)` (now one line
per endpoint instead of one per method+route+status combination).

**5xx error rate by route**
```promql
sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (route)
```
Same counter, same `rate()`, just filtered first by a label regex
(`status_code=~"5.."` matches any 5xx). This is the same shape of query as
the one above — worth pointing out explicitly, since it reinforces that
PromQL's power is mostly "filter by labels, then apply the same handful of
functions," not a large vocabulary to memorize.

**p95 request duration by route**
```promql
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))
```
`http_request_duration_seconds` is a **histogram**, not a counter or gauge
— under the hood it's actually several counters, one per bucket boundary
(`le="0.1"`, `le="0.5"`, `le="1"`, ...), each counting "how many requests
took ≤ this long." `histogram_quantile()` is the one function that knows
how to turn a set of cumulative bucket counters into an estimated
percentile. The `by (le, route)` is easy to get backwards: `le` **must** be
kept (it's the input `histogram_quantile()` needs), `route` is kept because
that's what we want a separate line per. Drop `le` from the `by (...)` and
the query silently returns nothing useful, not an error — worth
demonstrating that failure mode once, live.

### Row: Business metrics
Four counters the backend increments explicitly at points that matter to
the business, not the framework (`src/metrics.js` in `expense-backend-v1`):
`users_registered_total`, `user_logins_total`, `expenses_created_total`,
`expense_amount_rupees_total`. The dashboard reads these two ways —
```promql
sum(users_registered_total)              # instant — "how many, right now, total"
sum(rate(expenses_created_total[5m])) by (category_name)   # rate — "how fast, and of what kind"
```
— worth contrasting directly: a plain `sum()` with no `rate()` is correct
here because a Stat panel showing "total users ever" wants the raw
cumulative number, not a per-second speed. Wrapping `rate()` around a
counter is only useful when the *speed* is the interesting number, not the
total.

### Row: MySQL (mysqld_exporter)
None of these metric names come from this app's own code — `mysqld_exporter`
is a separate process that logs into MySQL with a dedicated read-only user
and re-exposes MySQL's internal status variables as Prometheus metrics.
This is the pattern to generalize: Prometheus doesn't know how to read
MySQL, Redis, or anything else's internals itself — an exporter's whole job
is being that translation layer.

```promql
mysql_up                                                              # gauge: 1 or 0
mysql_global_status_threads_connected / mysql_global_variables_max_connections * 100   # gauge ÷ gauge
rate(mysql_global_status_slow_queries[5m])                            # counter → rate
```
`mysql_up` (from mysqld_exporter, distinct from Prometheus's own
`up{job="mysql"}`) is a **gauge** — read it directly, no `rate()`, because
it's not accumulating anything, it's just "is the connection to MySQL
working right now." The connection-pool-usage query is two gauges divided
against each other — a legitimate, common pattern, and worth noting it's
*not* the same shape as the counter-based queries above it.

### Row: nginx (nginx-prometheus-exporter)
Same exporter pattern as MySQL, different target: `nginx-exporter` polls
nginx's built-in `stub_status` page and re-exposes it in Prometheus format.
```promql
rate(nginx_http_requests_total[5m])   # counter → rate, same shape as backend's request rate
nginx_connections_active              # gauge, read directly
```

### Row: Host & container — the USE method
Utilization, Saturation, Errors — the mental model for any *resource*
(as opposed to RED, which is for request-driven *services*). Two different
vantage points on the same physical machine:

```promql
100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])))   # host CPU, from node-exporter
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100   # host memory, from node-exporter
sum(rate(container_cpu_usage_seconds_total{name=~"backend|frontend|expense-mysql"}[5m])) by (name)   # per-container CPU, from cadvisor
container_memory_usage_bytes{name=~"backend|frontend|expense-mysql"}   # per-container memory, from cadvisor
```
`node_cpu_seconds_total` is, like `http_requests_total`, a counter — CPU
time is measured in *seconds accumulated in each mode* (idle, user, system,
...), never as a direct percentage. The idiom is always the same: take the
idle mode's rate (fraction of each second spent idle), subtract from 1,
multiply by 100. node-exporter answers "how busy is the whole host";
cadvisor answers "how much of that is this one container" — same
underlying kernel accounting, two different aggregation levels.

### Row: Scrape target health
```promql
up
```
Every target Prometheus scrapes gets an automatic `up` series: `1` if the
last scrape succeeded, `0` if it didn't (connection refused, timeout,
wrong content-type). This is the single most important — and most
misleading if you stop here — metric in the whole stack: `up == 1` only
means *Prometheus could reach the port*. It says nothing about whether the
application behind that port is doing its job correctly. Kill MySQL
(`docker compose exec mysql mysqladmin shutdown -u root -prootpass`)
without touching the backend container at all: `up{job="expense-backend"}`
stays `1` the entire time while every request to the app starts failing —
that gap is worth making the class sit with, since "the dashboard says
everything's up" is exactly the false confidence real on-call engineers
have been burned by. Restart mysql afterwards
(`docker compose up -d mysql`) to recover.

### Row: Alerting — Inactive / Pending / Firing
```promql
ALERTS
```
The moment any rule in `prometheus/alert-rules.yml` stops being false,
Prometheus automatically exposes it as a series on a metric called
`ALERTS{alertname=..., alertstate=...}` — there's no separate alerting
database, this is just another metric you can query like any other.
`alertstate` is `"pending"` while the condition is true but hasn't been
true for the rule's `for:` duration yet, then flips to `"firing"` once it
has; there's no series at all while a rule is `"inactive"` (a healthy
system means an *empty* table here, not a table full of green rows — worth
confirming directly, since "no data" and "all clear" look identical and
that's correct, not a bug).

**Why `for:` exists at all**: without it, a single noisy sample (one slow
request, one momentary scrape hiccup) would fire an alert instantly. `for:
2m` means the condition has to stay true continuously across multiple
scrape intervals before anything fires — the deliberate trade being a
slower reaction in exchange for not paging anyone over a blip.

**Where a firing alert actually goes**: Prometheus itself never sends an
email or a Slack message — it only evaluates rules and exposes state. The
`alerting.alertmanagers` block in `prometheus.yml` points it at
Alertmanager (`alertmanager:9093`), which does the actual routing based on
`alertmanager/alertmanager.yml`'s `route`/`receivers`. That two-binary
split (Prometheus decides *what's* true, Alertmanager decides *who hears
about it and how*) is deliberate in the real project too, not a shortcut
for this lab — it's what lets the same alert definition route differently
without touching Prometheus at all (e.g. grouping, silencing, or changing
notification channels are all Alertmanager-side concerns).

---

## SLO burn-rate alerting — a second, deeper layer on the same data

Everything above treats "alerting" as a threshold. This section is the
other kind: alerting on *rate of error-budget consumption*, on the
`Expense Tracker — v1 SLO & Burn Rate` dashboard.

Every SLI below actually comes in two layers: `sli:availability:*` /
`sli:latency:*` (no second segment) is the **official** number, measured at
nginx via nginx-module-vts — the boundary closest to the real client.
`sli:backend:availability:*` / `sli:backend:latency:*` is the same question
asked one layer downstream, straight from the backend. Alerting fires on
the official one; the dashboard plots both, because *the gap between them*
is diagnostic on its own — if only the official line dips, the problem is
nginx (or the network between nginx and the backend); if both dip together,
it's downstream, in the backend or MySQL. See "Known gap" near the end of
this section, though, before assuming the official layer has real data —
it currently doesn't, in this stage, for an unrelated reason.

### SLA vs. SLO — why there are two numbers, not one
Applied to both signals this stage tracks, same pattern each time: the
internal target is deliberately *stricter* than the external commitment,
so breaching the internal number is the early warning that gives you room
to fix things before the external one — the one you'd owe a customer over
— is actually at risk.

- **Availability** — SLA **99%** (external/contractual) vs. SLO **99.5%**
  (internal, what this stack actually alerts on). Every availability alert
  and error-budget calculation below is against the SLO (0.995), never
  directly against the 99% SLA.
- **Latency** — client-facing commitment **500ms p99** vs. internal target
  **400ms p99** (session-only numbers, chosen for this teaching session
  specifically — not derived from any real measured baseline). Same shape
  as availability: the alert fires on 400ms, well before the 500ms
  commitment is actually broken.
- **Window: 30 days.** Both SLOs are measured over a rolling 30-day period
  — the industry-standard accounting window, and long enough that a single
  bad hour is a real, visible dip rather than the entire signal.

### What's an SLI and an error budget
- **SLI** (Service Level *Indicator*) — the actual measured number.
  Availability's is a ratio; latency's, this session, is a percentile
  value rather than a ratio (more on why below):
  ```promql
  sli:availability:ratio_rate1h            # official (nginx): fraction of requests, last 1h, NOT a 5xx
  sli:availability:ratio_rate30d           # official (nginx): same, over the full 30-day SLO window
  sli:backend:availability:ratio_rate1h    # diagnostic (backend): same question, one layer downstream
  sli:latency:p99_seconds_rate1h           # official (nginx): estimated p99 duration, last 1h, in seconds
  sli:backend:latency:p99_seconds_rate1h   # diagnostic (backend): same estimate, one layer downstream
  ```
- **Error budget** — the SLO restated as "how much failure is allowed":
  target 99.5% availability means the budget is the other 0.5%
  (`1 - 0.995 = 0.005`). This turns "is 99.3% availability bad?" from a
  vague question into an arithmetic one: at 99.3%, you've already spent
  0.7 of your 0.5-point budget — you're *over* budget, not just "a little
  under 100%." Latency's 400ms p99 target doesn't have an equally clean
  error-budget reading (see below) — it's a threshold, not a budget.

### The burn-rate alerts, with real numbers
```promql
(1 - sli:availability:ratio_rate1h) / 0.005 > 14
and
(1 - sli:availability:ratio_rate30d) / 0.005 > 14
```
`1 - sli:availability:ratio_rate1h` is the *current* error rate. Dividing
by the error budget (`0.005`) converts that into "how many multiples of
the sustainable rate are we currently failing at" — the burn rate. A burn
rate of exactly 1 means "failing at precisely the rate that would consume
the whole 30-day budget in exactly 30 days." A burn rate of 14 means the
whole budget would be gone in `30d / 14 ≈ 51 hours` if it kept up — worth
paging over (`AvailabilitySLOBurnRate`, `severity: page`).

**The real caveat on pairing 1h with the full 30d period** (stated in
`slo-burn-rate-alerts.yml`'s own comments, worth repeating here): the
textbook multi-window pattern pairs a short window with a *medium* long
window — still much shorter than the full SLO period (Google's own
examples use things like 5m+1h or 30m+6h) — specifically so the long
window stays sensitive to a real, recent incident. Pairing with the
literal full period, like here, means the 30d side is only as sensitive as
however much traffic has actually accumulated in it. On a Prometheus
instance with weeks of real history, a short, sharp fault would barely
move a genuine 30-day ratio, and this alert would be far less reactive
than the classic pattern. **It works for a live classroom demo
specifically because this stack's Prometheus has no named data volume**
(see `docker-compose.yml` — nothing under `volumes:` backs `/prometheus`),
so recreating the container wipes its TSDB. Early in a fresh session,
`[30d]` and `[1h]` see almost the same data, so the 30d side reacts nearly
as fast as the 1h side. The longer a single Prometheus container has been
running (more real history accumulated), the more diluted — and less
demo-friendly — the 30d side becomes.

### The latency alert is a threshold, not burn-rate math — and why
```promql
sli:latency:p99_seconds_rate1h > 0.4
and
sli:latency:p99_seconds_rate30d > 0.4
```
Availability has a natural error-budget reading because "% of requests
that were non-5xx" is already a ratio in [0, 1] — subtracting it from 1
directly gives "% that were bad," which is exactly what an error budget
measures. A p99 duration doesn't reduce the same way: `histogram_quantile()`
estimates *a duration value* (via linear interpolation between whichever
two bucket boundaries straddle the 99th percentile), not a fraction of
requests below a fixed line. Getting an exact "% of requests under 400ms"
would need an actual `le="0.4"` bucket boundary, which doesn't exist here
(the nearest real boundaries are `le="0.25"` and `le="0.5"`) — so
`LatencyP99High` is a direct threshold on the estimated value instead,
same style as `alert-rules.yml`'s plain thresholds, just still carrying
the `slo: latency` label so it shows up on this dashboard rather than the
metrics one. Worth knowing as a limitation, not a bug: because the
estimate is interpolated between the 0.25s and 0.5s buckets, the reported
"p99" here is an approximation of the true p99, more so than it would be
with a bucket boundary actually placed near 0.4s.

### Sizing fault injection so it moves the needle without exhausting a month's budget
The 0.5% error budget is small on purpose — it doesn't take much to
consume a lot of it, which is exactly why "don't overdo it" is a real
concern, not caution for its own sake:
- Every 5xx response consumes budget; the fraction consumed is roughly
  `(bad requests) / (total requests over the window)`. With
  `healthy-load.sh` running underneath, a **1–2 minute** fault burst
  (`fault-load.sh` with a modest `ITERATIONS`, or a short `inject`-style
  toggle) is plenty to push the 1h burn rate past 14x and watch
  `AvailabilitySLOBurnRate` transition Pending→Firing — there's no need
  for a long, sustained outage to see the mechanism work.
- **Before repeating the demo**, recreate Prometheus so the fault you just
  injected doesn't keep dragging the 30-day error-budget-remaining stat
  down for the rest of the session:
  ```bash
  docker compose up -d --force-recreate prometheus
  ```
  This wipes Prometheus's TSDB (no named volume backs it), giving every
  fresh demo run a clean 30-day/1h baseline instead of an already-dented
  one. Recreating Prometheus doesn't touch the app itself — nothing about
  the running backend/frontend/mysql containers changes.
- If you want the error budget to visibly *stay* dented for a while (to
  show what a real incident's aftermath looks like on the
  error-budget-remaining stat), that's the opposite move: inject longer,
  and deliberately don't recreate Prometheus before the next class.

### Two alert tables, on purpose
The metrics dashboard's alert table queries bare `ALERTS` (shows
everything: `InstanceDown`, `MySQLDown`, every threshold alert, and these
SLO alerts too). The SLO dashboard's table queries `ALERTS{slo=~".+"}` —
only alerts carrying an `slo` label, i.e. only the burn-rate alerts. That's
deliberate scoping for a dashboard specifically about SLOs, not an
accidental filter that'll silently miss alerts added later — if you add
more `slo`-labeled alerts, they show up here automatically; non-SLO alerts
show up on the *other* table, also automatically.

### SLO dashboard panels
- **Error budget remaining** — `(sli:availability:ratio_rate30d - 0.995) /
  (1 - 0.995) * 100`. Same "restate as a percentage of budget" idea as
  above, phrased as "how much is left" instead of "how fast is it going."
  100% = no budget spent yet; 0% = exactly on target; negative = already
  over budget.
- **Burn rate vs. threshold** — plots both
  `(1 - sli:availability:ratio_rate1h) / 0.005` and the 30d equivalent as
  two lines, with a reference line at 14 (`fieldConfig.thresholds` with
  `thresholdsStyle.mode: "line"` — a visual threshold line, not a second
  query). Watching both lines cross 14 at the same moment the alert table
  shows `AvailabilitySLOBurnRate` as `firing` is the single best "this is
  how it all connects" moment in this stage — and watching the 30d line
  react *more slowly* than the 1h line as the session goes on is the
  caveat above made visible.
- **Availability SLI panel** — the raw ratio the burn rate above is
  computed from, 1h vs. 30d side by side, so a real incident is visible as
  *both* lines dropping together (early in a session), or just the 1h line
  dropping alone (later in a session, once 30d has enough history to be
  less reactive).
- **Latency SLI panel** — estimated p99, 1h vs. 30d, with reference lines
  at 0.4s (internal target) and 0.5s (client-facing commitment) — a rising
  line crossing 0.4s while `LatencyP99High` shows `firing` in the alert
  table is the equivalent "how it connects" moment for latency.

### Known gap: the frontend currently won't start in this stage
`../expense-frontend-v1`'s `nginx.conf` references another service by
hostname that isn't part of this stage's `docker-compose.yml` — reserved
for a later stage this curriculum hasn't reached yet. nginx resolves a
static hostname reference like that once, at startup, not per-request; when
it can't resolve at all (no service by that name exists on this network,
not just temporarily unreachable), nginx typically refuses to start rather
than degrading gracefully — so the whole frontend container fails to come
up, not just the one feature that reference was for.

Practical effect right now: `mysql`, `backend`, and every exporter that
doesn't depend on the frontend still work fine, including the
`sli:backend:*` (diagnostic) SLIs and their panels. The `nginx-vts` scrape
job, the `sli:availability:*`/`sli:latency:*` (official) SLIs, and the two
SLO burn-rate alerts that read them will show "no data" and never fire
until the frontend can actually start. Deferred deliberately for now, not
an oversight — fixing it means either bringing that missing service into
this stage ahead of schedule, or editing this copy of the frontend's config
to drop the reference to it. Neither has been done yet.

### What else is still deliberately missing
No automated response to a firing SLO alert — that's a distinct piece of
work of its own.
