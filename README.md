# Metrics & Alerting (Prometheus + Alertmanager + Grafana)

Prometheus scraping the app + a few sidecar exporters, Alertmanager routing
firing alerts to email + Slack, and two Grafana dashboards built from that
data. No traces, no log shipping — just: where does a metric come from,
what shape is it, how do you turn it into a number worth looking at, and
what does it take to get paged about it. Metrics and alerting stay one
stage here rather than two: both are the same three tools (Prometheus,
Alertmanager, Grafana), just different questions asked of the same data.

## What's running

The app repos (`../expense-backend-v1.1`, `../expense-frontend-v1.1`,
`../expense-mysql-v1.1` — siblings of this folder) are otherwise
unmodified; this folder mostly adds the observability layer around them.
One exception: `../expense-backend-v1.1`'s `src/app.js` has its middleware
order fixed — `httpLogger`/`baseLogContext`/`httpMetricsMiddleware` now run
*before* `express.json()`, not after. Before that fix, a malformed request
body (`express.json()` throwing) skipped every middleware registered after
it, including the metrics middleware (so that 500 was invisible to
`http_requests_total`) and the logging middleware (so `req.log` was
undefined when the error handler tried to use it — a second crash on top
of the first). Confirmed live with `fault-load.sh`'s malformed-JSON case
before this fix existed.

- **Prometheus** — scrapes the backend's own `/metrics`, `mysqld-exporter`
  (a sidecar process reading MySQL's internals), `nginx-exporter` (reading
  nginx's `stub_status`), nginx-module-vts (request-duration histograms,
  straight from nginx), `node-exporter` (host CPU/mem/disk via `/proc`,
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

**p50 / p95 / p99 request duration by method & route — three separate panels**
```promql
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, method, route))
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, method, route))
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, method, route))
```
`http_request_duration_seconds` is a **histogram**, not a counter or gauge
— under the hood it's actually several counters, one per bucket boundary
(`le="0.1"`, `le="0.5"`, `le="1"`, ...), each counting "how many requests
took ≤ this long." `histogram_quantile()` is the one function that knows
how to turn a set of cumulative bucket counters into an estimated
percentile — and it only ever answers for *one* quantile per call, which
is why this is three queries (and three panels), not one query with three
numbers in it. The `by (le, method, route)` is easy to get backwards: `le`
**must** be kept (it's the input `histogram_quantile()` needs); `method`
and `route` are kept together because that's the actual dimension worth a
separate line per — `POST /expenses/` (a write, touches the DB) and
`GET /expenses/` (a read) can have very different latency profiles, and
grouping by `route` alone would blend them into one misleading line. Drop
`le` from the `by (...)` and the query silently returns nothing useful, not
an error — worth demonstrating that failure mode once, live.

**Why three panels instead of three lines on one**: p50, p95, and p99 tell
different stories — p50 is the typical request, p99 is the tail, the one
slow outlier in a hundred that a p50-only view hides completely — and once
each line is *also* split by method+route, three quantiles times several
routes on one panel gets unreadable fast. Separate panels per quantile
keep each one legible on its own.

### Row: Business metrics
Four counters the backend increments explicitly at points that matter to
the business, not the framework (`src/metrics.js` in `expense-backend-v1.1`):
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

Backend-only — every SLI below (`sli:availability:*`, `sli:latency:*`)
comes straight from the backend's own metrics, one source, no second
layer. nginx-module-vts metrics exist and work (see the metrics
dashboard's "nginx vs backend latency" panel), but an earlier version of
this section paired them with the backend's own numbers as an
"official vs diagnostic" comparison, and that added a second axis of
complexity — which layer is which, why two numbers for the same idea — that
wasn't worth it for what an SLO needs to teach here. One number, one
source, is enough. If a proxy-layer comparison becomes useful later, that's
a second, separate dashboard once there's a reason to build it, not a
second set of names bolted onto this one.

### SLA vs. SLO — availability only
- **Availability** — SLA **99%** (external/contractual) vs. SLO **99.5%**
  (internal, what this stack actually alerts on), deliberately *stricter*
  than the SLA — breaching the SLO is the early warning that gives you room
  to fix things before the SLA itself, the number you'd owe a customer
  over, is actually at risk. Every availability alert and error-budget
  calculation below is against the SLO (0.995), never directly against the
  99% SLA.
- **Latency** — a single target, **200ms p99**, no separate SLA number.
  Simpler on purpose: not every SLO needs the SLA/SLO pair to be useful,
  and one clear number beats two numbers whose relationship needs
  explaining.
- **Window: 30 days.** Both SLOs are measured over a rolling 30-day period
  — the industry-standard accounting window, and long enough that a single
  bad hour is a real, visible dip rather than the entire signal.

### What's an SLI and an error budget
- **SLI** (Service Level *Indicator*) — the actual measured number.
  Availability's is a ratio; latency's is a percentile value rather than a
  ratio (more on why below):
  ```promql
  sli:availability:ratio_rate1h   # fraction of requests, last 1h, that were NOT a 5xx
  sli:availability:ratio_rate30d  # same fraction, over the full 30-day SLO window
  sli:latency:p99_seconds_rate1h  # estimated p99 request duration, last 1h, in seconds
  sli:latency:p99_seconds_rate30d # same estimate, over the full 30-day window
  ```
- **Error budget** — the SLO restated as "how much failure is allowed":
  target 99.5% availability means the budget is the other 0.5%
  (`1 - 0.995 = 0.005`). This turns "is 99.3% availability bad?" from a
  vague question into an arithmetic one: at 99.3%, you've already spent
  0.7 of your 0.5-point budget — you're *over* budget, not just "a little
  under 100%." Latency's 200ms p99 target doesn't have an equally clean
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
sli:latency:p99_seconds_rate1h > 0.2
and
sli:latency:p99_seconds_rate30d > 0.2
```
Availability has a natural error-budget reading because "% of requests
that were non-5xx" is already a ratio in [0, 1] — subtracting it from 1
directly gives "% that were bad," which is exactly what an error budget
measures. A p99 duration doesn't reduce the same way: `histogram_quantile()`
estimates *a duration value* (via linear interpolation between whichever
two bucket boundaries straddle the 99th percentile), not a fraction of
requests below a fixed line. Getting an exact "% of requests under 200ms"
would need an actual `le="0.2"` bucket boundary, which doesn't exist here
(the nearest real boundaries are `le="0.1"` and `le="0.25"`) — so
`LatencyP99High` is a direct threshold on the estimated value instead,
same style as `alert-rules.yml`'s plain thresholds, just still carrying
the `slo: latency` label so it shows up on this dashboard rather than the
metrics one. Worth knowing as a limitation, not a bug: because the
estimate is interpolated between the 0.1s and 0.25s buckets, the reported
"p99" here is an approximation of the true p99, more so than it would be
with a bucket boundary actually placed near 0.2s.

**One more real limitation, worth naming rather than working around
silently**: `sli:latency:p99_seconds_rate1h` is a single aggregate number
across *every* route combined — it can tell you the backend's p99 is over
200ms, but not *which* endpoint is actually slow. Rather than break the
SLO itself apart per-route (turning one clear "is the backend fast enough"
number into several), `LatencyP99High`'s `description` runs a second,
one-off query at the moment the alert notification is actually rendered —
Prometheus's `query` template function, not a recording rule — that finds
the single worst route+method via `topk(1, ...)` and names it directly in
the Slack/email text: `Slowest endpoint right now: POST /auth/signup (p99
497ms)`. Confirmed live: bcrypt's cost factor (`BCRYPT_ROUNDS = 12` in
`expense-backend-v1.1/src/routes/auth.js`) makes `/auth/signup` and
`/auth/signin` reliably the slowest routes in this app, by a wide margin
over anything DB-bound — worth pointing out as the real, first thing this
alert is likely to ever name, not a hypothetical.

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
Every 1h/30d pair is two separate panels, not two lines sharing one panel
and one auto-scaled y-axis — a 1h ratio and a 30d ratio can sit only
fractions of a percent apart, and squeezed onto the same tight axis
(sometimes together with the 30d line only stepping once every 5 minutes,
per `slo-rules.yml`'s two rule-group intervals) the two lines can look like
they're doing something dramatic — a "jump" — that's really just each
window's own scale being different. Separate panels, each free to
auto-scale to its own data, read as flat, boring, correct — which is what
a healthy SLO should look like.

- **Error budget remaining** — `(sli:availability:ratio_rate30d - 0.995) /
  (1 - 0.995) * 100`. Same "restate as a percentage of budget" idea as
  above, phrased as "how much is left" instead of "how fast is it going."
  100% = no budget spent yet; 0% = exactly on target; negative = already
  over budget.
- **Burn rate vs. threshold** (1h panel, 30d panel) — each plots its own
  window's `(1 - ratio) / 0.005`, with a reference line at 14
  (`fieldConfig.thresholds` with `thresholdsStyle.mode: "line"` — a visual
  threshold line, not a second query). Watching the 1h panel's line cross
  14 at the same moment the alert table shows `AvailabilitySLOBurnRate` as
  `firing` is the best "this is how it all connects" moment in this stage
  — then checking the 30d panel next to it and seeing it react more slowly
  (or not at all, later in a long session) is the multi-window caveat made
  visible, without the two panels' different scales fighting each other
  for one shared axis.
- **Availability SLI** (1h panel, 30d panel) — the raw ratio the burn rate
  above is computed from.
- **Latency SLI** (1h panel, 30d panel) — estimated p99, with a reference
  line at 0.2s (the SLO target). A rising line crossing it while
  `LatencyP99High` shows `firing` in the alert table is the equivalent
  "how it connects" moment for latency — and the alert's own description
  (see above) is where you'd actually find out *which* route is behind it,
  since this panel's number is the whole-backend aggregate.

### What's still deliberately missing
No automated response to a firing SLO alert — that's a distinct piece of
work of its own.
