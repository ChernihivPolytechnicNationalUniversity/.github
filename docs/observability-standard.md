# DU-RFC-001: Observability Standard

| Field | Value |
|---|---|
| **Status** | Draft |
| **Version** | 0.2.0 |
| **Applies to** | Every DigitalUni service that runs in the shared Kubernetes cluster. |
| **Audience** | Service owners, application developers, SRE. |
| **Owner** | Platform team. |
| **License** | Internal. |

## 0. Conformance language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY** and **OPTIONAL** in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174). They appear in **bold uppercase** when used normatively.

A service is **conformant** with this standard when every applicable **MUST** is satisfied. A single violation of any **MUST** makes the service non-conformant; CI **MUST** reject such changes.

### 0.1 Scope (in)

- Helm charts produced by DU service teams (typically `*/.infra/helm/`).
- Application source code that emits metrics, logs, traces.
- Alerts, dashboards, and recording rules authored for a DU service.

### 0.2 Scope (out)

- Vendor- and operator-managed components shipped with their own observability conventions (cert-manager, CloudNativePG, Longhorn, KubeVirt, Redpanda, etc.). They are scraped as-is.
- General-purpose Kubernetes hygiene (image signing, security context, network policies). Other standards cover those.
- Synthetic monitoring, RUM, profiling. Future addenda.

### 0.3 Waivers

Deviation from any **MUST** **REQUIRES** an explicit waiver in the pull request that introduces it. A waiver consists of:

1. A top-level comment in the PR body starting with `OBSERVABILITY-WAIVER:` followed by a one-sentence justification.
2. Reference to the section number being waived (e.g. `OBSERVABILITY-WAIVER: §5.1 — third-party SDK ships fixed metric name we cannot rename until v3.0`).
3. Approval from a member of the Platform team.

The CI gate **MUST** still fail the build; the waiver is recorded for audit, not as an override path.

### 0.4 Versioning

This document follows [Semantic Versioning 2.0.0](https://semver.org). Backwards-incompatible changes increment the MAJOR version. Conformance always targets a specific MAJOR version, which a service declares in its chart (`digitaluni.io/observability-standard: "0"`).

---

## 1. Identification

### 1.1 Pod template labels (MUST)

Every workload kind (`Deployment`, `StatefulSet`, `DaemonSet`, `Job`, `CronJob`) **MUST** set the following labels on `spec.template.metadata.labels`:

| Label | Regex | Source | Example |
|---|---|---|---|
| `app.kubernetes.io/name` | `^[a-z][a-z0-9-]{0,40}$` | Chart name | `du-gateway` |
| `app.kubernetes.io/instance` | `^[a-z][a-z0-9-]{0,40}$` | Helm release name | `du-gateway-dev` |
| `app.kubernetes.io/version` | semver `^[0-9]+\.[0-9]+\.[0-9]+(-[A-Za-z0-9.-]+)?$` or `^sha256:[a-f0-9]{64}$` | Image version | `1.4.2` |
| `app.kubernetes.io/component` | one of: `api`, `worker`, `scheduler`, `migration`, `exporter`, `sidecar`, `cli` | Closed enum | `api` |

The same set **MUST** appear on the parent workload's `metadata.labels`.

**Forbidden:**
- `app: <name>` (legacy non-namespaced label).
- Free-form `tier`, `env`, `role` labels in the `app.kubernetes.io/...` group beyond the four above.
- Multiple workloads with the same `(app.kubernetes.io/name, app.kubernetes.io/instance)` pair.

**Rationale:** these labels are the only selectors the platform uses to find a service's pods, build dashboards, and route alerts. Deviation makes the service invisible.

### 1.2 Workload `metadata.name` (MUST)

`metadata.name` of the workload **MUST** equal `app.kubernetes.io/instance`. Helpers that generate `<release>-<chart>`-style names are **forbidden** for primary workloads.

```yaml
# OK
kind: Deployment
metadata:
  name: du-gateway-dev
  labels:
    app.kubernetes.io/name: du-gateway
    app.kubernetes.io/instance: du-gateway-dev

# NOT OK — name does not match instance
kind: Deployment
metadata:
  name: du-gateway-dev-du-gateway
  labels:
    app.kubernetes.io/name: du-gateway
    app.kubernetes.io/instance: du-gateway-dev
```

### 1.3 Namespace labels (set by Platform)

The Platform team sets the following on each namespace. Services **MUST NOT** override them from their charts.

| Label | Values | Purpose |
|---|---|---|
| `digitaluni.io/environment` | `prod \| dev \| staging \| poc` | Propagated as `environment` series label |

**The application MUST NOT emit `environment` from its own code, env vars, or chart values.** It would conflict with the platform-injected value.

### 1.4 Service capabilities (MUST)

Every service **MUST** declare its observability-relevant capabilities via the pod-template label `digitaluni.io/capabilities`. The value is a comma-separated list, sorted alphabetically, from the closed enum below.

```yaml
metadata:
  labels:
    digitaluni.io/capabilities: "db-client,http-server,messaging-publisher"
```

| Capability | Meaning |
|---|---|
| `http-server` | Service handles inbound HTTP requests. |
| `http-client` | Service makes outbound HTTP/gRPC calls. |
| `db-client` | Service queries one or more databases. |
| `cache-client` | Service reads/writes a cache. |
| `messaging-publisher` | Service publishes messages to a queue/topic. |
| `messaging-consumer` | Service consumes messages from a queue/topic. |
| `job-runner` | Service runs scheduled or background jobs inside its own process (independent of Kubernetes `CronJob`). |

The list:
- **MUST** contain at least one capability. A service with none has no observable surface and is not a service.
- **MUST NOT** contain duplicates.
- **MUST NOT** contain capabilities the service does not actually have. CI verifies that the service emits the canonical metrics required for each declared capability.

A capability **drives**:
- Which canonical metric sets from §5 the service is required to emit.
- Which mandatory alerts from §8.8 apply.
- Which Golden Signals panels populate Row 2 of the dashboard (§9.2.2).

---

## 2. Endpoints

### 2.1 Metrics port (MUST)

The container that exposes Prometheus-format metrics **MUST** declare a dedicated container port:

```yaml
ports:
  - name: metrics
    containerPort: 8081
    protocol: TCP
```

| Field | Required value |
|---|---|
| `name` | exactly `metrics` |
| `containerPort` | `8081` (single canonical port across all services) |
| `protocol` | `TCP` |

**Forbidden:**
- Reusing the application's HTTP traffic port for `/metrics`.
- Port names `http-metrics`, `prom`, `prometheus`, `telemetry`, or numeric-only ports.
- A `Service` whose `port.name` is anything other than `metrics` for the metrics endpoint.

### 2.2 Metrics HTTP path (MUST)

The metrics endpoint **MUST** be served at HTTP path `/metrics`. The endpoint **MUST**:

- Respond `200 OK` with `Content-Type: text/plain; version=0.0.4` (Prometheus text format) or `application/openmetrics-text` (OpenMetrics).
- Return within **5 seconds**. Slower endpoints lose scrapes.
- **NOT** require any form of authentication. Network-level isolation is the Platform's responsibility.
- **NOT** mutate state on `GET`.

**Forbidden paths:**
- `/actuator/prometheus`, `/internal/metrics`, `/admin/metrics`, `/_metrics`, `/-/metrics`, `/prometheus`, `/stats`, `/v1/metrics`, or any other framework default.
- Any path that requires query parameters.

If the chosen library or framework publishes metrics at a non-conformant path, the service **MUST** reconfigure or proxy it to `/metrics`.

### 2.3 Health endpoints (MUST)

Workloads **MUST** expose three health endpoints, served on the application's main HTTP port (not the `metrics` port):

| Path | Probe | Semantics |
|---|---|---|
| `/health/live` | `livenessProbe` | Process is alive (returns 200 if the event loop is responsive) |
| `/health/ready` | `readinessProbe` | Service is ready to serve traffic (dependencies reachable, warm-up complete) |
| `/health/startup` | `startupProbe` | Initial bootstrap finished (allows long warm-up without killing the pod) |

Health endpoints **MUST NOT** be the same as `/metrics`. They **MUST NOT** be authenticated.

### 2.4 Prohibited endpoints

The following endpoints **MUST NOT** be exposed in any environment, including dev:

- Endpoints that dump environment variables, configuration, secrets, or bean / DI definitions.
- Endpoints that allow live profiling (CPU, heap, allocation) without authentication.
- Endpoints that produce heap dumps or thread dumps without authentication.

If the framework exposes such endpoints by default, the service **MUST** disable them or restrict them to localhost.

---

## 3. Custom metric naming

### 3.1 Format (MUST)

Custom metrics emitted by service code **MUST** match the regex:

```
^<service_prefix>_[a-z][a-z0-9]*(_[a-z0-9]+)+_<unit_suffix>$
```

Where:
- `<service_prefix>` = `app.kubernetes.io/name` with dashes replaced by underscores. For `du-gateway`: `du_gateway_`.
- `<unit_suffix>` is one of the suffixes from §3.2.
- Total length **MUST NOT** exceed **80 characters**.

### 3.2 Unit suffixes (MUST)

The last segment of every metric name **MUST** be one of:

| Suffix | Type | Use |
|---|---|---|
| `_total` | Counter | Monotonic counter. Always combined with `rate()` in PromQL. |
| `_seconds` | Gauge / Histogram | Durations. **Always seconds.** Never milliseconds, microseconds, or nanoseconds in the metric name. |
| `_bytes` | Gauge / Histogram | Sizes. |
| `_ratio` | Gauge | Real-valued ratio in `[0..1]`. |
| `_info` | Gauge with value `1` | Static metadata. |
| `_count` | Gauge | Current size of a collection (active sessions, queue depth). |
| `_celsius` | Gauge | Temperature, when applicable. |

For histograms the client library emits `_bucket`, `_sum`, `_count` automatically; the base name **MUST** end in `_seconds` or `_bytes`.

### 3.3 HELP text (MUST)

Every metric **MUST** have a `HELP` string. The string:

- **MUST** be a single-line sentence.
- **MUST** be ≤ 200 characters.
- **MUST** describe **what** is measured and the **unit** if not obvious from the name.
- **MUST NOT** contain timestamps, emoji, or references to specific commit hashes / ticket IDs.

```
# OK
# HELP du_gateway_session_active_count Number of active sessions on this pod.
# TYPE du_gateway_session_active_count gauge

# NOT OK — no help text
# TYPE du_gateway_session_active_count gauge
```

### 3.4 Forbidden patterns (MUST NOT)

The following are **explicitly forbidden**:

| Pattern | Example | Replacement |
|---|---|---|
| `camelCase` | `requestCount` | `request_count` |
| `kebab-case` | `request-count` | `request_count` |
| `SCREAMING_SNAKE` | `REQUEST_COUNT` | `request_count` |
| Mixed unit + counter | `_seconds_total` | `_seconds` (the histogram counter is the `_count` series) |
| Non-second time units | `_ms`, `_milliseconds`, `_microseconds`, `_nanoseconds` | `_seconds` |
| Metric without service prefix | `request_total` | `du_gateway_request_total` |
| Metric without unit suffix | `du_gateway_request` | `du_gateway_request_total` |
| Plurals in unit suffix | `_seconds_per_request` | use rate in queries |
| Service prefix that differs from `app.kubernetes.io/name` | `gateway_*` for `du-gateway` chart | `du_gateway_*` |

### 3.5 Reserved prefixes

The following name prefixes are reserved by upstream conventions and **MUST NOT** be used for custom metrics. Use the canonical sets in §5 instead.

- `http_handler_*`, `http_client_*` (canonical HTTP, §5.1, §5.2).
- `db_client_*` (canonical DB, §5.3).
- `cache_*` (canonical cache, §5.4).
- `messaging_*` (canonical messaging, §5.5).
- `job_*` (canonical jobs, §5.6).
- `kube_*`, `node_*`, `container_*`, `process_*`, `go_*`, `python_*`, `nodejs_*`, `jvm_*`, `dotnet_*` (framework / runtime metrics).
- `up`, `scrape_*` (Prometheus internals).

### 3.6 Mandatory build info metric (MUST)

Every service **MUST** emit exactly one info metric:

```
<service_prefix>build_info  (gauge, always value 1)
```

| Label | Values |
|---|---|
| `version` | semver (`1.4.2`) or sha256 digest (`sha256:abc...`). **MUST** match the deployed image tag. |
| `commit` | Git short SHA (`^[a-f0-9]{7,40}$`) |
| `build_time` | RFC 3339 / ISO 8601 UTC timestamp of build (`2026-05-17T10:00:00Z`) |

Example series:
```
du_gateway_build_info{version="1.4.2",commit="a1b2c3d",build_time="2026-05-17T10:00:00Z"} 1
```

**Required:**
- Exactly one time series per running pod. Each new build produces a new label combination; old series naturally age out.
- Labels populate from build/CI environment (typically env vars injected at image build). They are **fixed for the lifetime of a pod** — the application **MUST NOT** mutate them after startup.

**Forbidden:**
- Multiple series per pod (e.g., one per subsystem).
- Free-text fields (`description`, `notes`).
- Labels with high cardinality (`build_id`, `pipeline_run_id`).
- Omitting any of the three required labels.

**Rationale:** dashboards and alerts need to answer "what's running right now" without consulting an external system. `build_info{version=...} == 1` joined to any operational metric immediately attributes behavior to a specific build.

---

## 4. Metric labels

### 4.1 Cardinality rule (MUST)

Every label on a custom metric **MUST** have a closed and predictable set of values. If a value originates from user input or external traffic, the label is forbidden.

**Cardinality budget per metric:** total unique time series count = product of unique label values **MUST NOT** exceed **1 000**. CI **MUST** emit a warning at 800 (80% of budget) and **MUST** reject the change at 1 000.

### 4.2 Allowed label patterns

| Label | Allowed values | Cardinality |
|---|---|---|
| `result` | `success`, `fail`, `dropped`, `timeout` | ≤ 4 |
| `status_code` | numeric HTTP code as string | ≤ 20 |
| `method` | UPPERCASE: `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS` | ≤ 7 |
| `route` | normalized path template (`/users/:id`, never `/users/123`) | ≤ 100 per service |
| `cache_name`, `formula`, `queue`, `topic` | closed set from service config | ≤ 20 |
| `kind`, `type` | closed enum | ≤ 10 |

### 4.3 Forbidden labels (MUST NOT)

The following labels are **never** acceptable on any metric:

| Label | Why |
|---|---|
| `user_id`, `account_id`, `email`, `username`, `session_id`, `device_id` | Unbounded user space. |
| `request_id`, `trace_id`, `span_id`, `correlation_id` | Per-request — belongs in traces, not metrics. |
| `ip`, `client_ip`, `external_host`, `peer_address` | Open set. |
| Raw `url`, `path` containing IDs (`/users/123/orders/456`) | Use normalized templates. |
| `error_message`, `exception_message`, `stack`, `cause` | Free text. Use `error_type` from a closed enum. |
| `query`, `sql`, `payload`, `body`, `headers` | Free text. |
| `timestamp`, `time`, `date` | Time is the metric's own dimension. |
| `version` (service version) | Already in `app.kubernetes.io/version` and auto-injected. |

### 4.4 Auto-injected labels (MUST NOT be set by app)

The Platform injects these labels on every series via vmagent relabel. The application **MUST NOT** emit them under any name:

- `namespace`
- `pod`
- `container`
- `node`
- `environment`
- `service`
- `cluster`

If the application attempts to set them, the platform-injected value wins, and the application's value is silently dropped.

### 4.5 Label name format (MUST)

Label names **MUST** match `^[a-z][a-z0-9_]{0,40}$`.

**Forbidden:** camelCase, dashes, dots, length > 41.

### 4.6 Label value length (MUST)

A label value **MUST NOT** exceed **128 characters**. Values that may exceed (URLs, descriptions) **MUST** be truncated or moved out of labels.

---

## 5. Canonical metric sets

For cross-service comparability, the following metric names are reserved. Every service that has the corresponding capability **MUST** emit metrics under exactly these names with exactly these labels and buckets.

### 5.0 Capability → metric set mapping (MUST)

| Capability (§1.4) | Mandatory canonical set |
|---|---|
| `http-server` | §5.1 `http_handler_duration_seconds` |
| `http-client` | §5.2 `http_client_duration_seconds` |
| `db-client` | §5.3 `db_client_duration_seconds` |
| `cache-client` | §5.4 `cache_operation_duration_seconds` |
| `messaging-publisher` | §5.5 `messaging_operation_duration_seconds` (operations `publish`, `ack`) |
| `messaging-consumer` | §5.5 `messaging_operation_duration_seconds` (operations `consume`, `ack`, `nack`) **AND** §5.5.1 `messaging_consumer_lag` |
| `job-runner` | §5.6 `job_duration_seconds` |

A service that declares capability `X` but does not emit the corresponding canonical metrics is non-conformant. A service that emits canonical metrics for a capability it has not declared is non-conformant.

### 5.1 HTTP server — `http_handler_duration_seconds`

**Type:** histogram.

**Required labels:**

| Label | Values |
|---|---|
| `method` | UPPERCASE HTTP method (`GET`, `POST`, ...) |
| `status_code` | numeric HTTP status as string (`"200"`, `"500"`). Class form (`"5xx"`) is **forbidden**. |
| `route` | normalized template (`/users/:id`); empty string `""` for un-routed (404, 0-byte 405). |

**Required buckets** (seconds):
```
0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10
```

**Forbidden:**
- Emitting `http_server_requests_seconds_*`, `http_request_duration_seconds_*`, or any other framework-default HTTP-server metric name. It **MUST** be renamed to `http_handler_duration_seconds`.
- Emitting a separate counter such as `http_requests_total`. The histogram's `_count` series is the counter.
- Labels `outcome`, `exception`, `uri`, `endpoint`, `error`, `instance`, `job`.
- Recording paths with embedded IDs (`/users/123` instead of `/users/:id`).

### 5.2 HTTP client — `http_client_duration_seconds`

**Type:** histogram.

**Required labels:**

| Label | Values |
|---|---|
| `method` | UPPERCASE |
| `status_code` | numeric status as string; `"0"` if no response received (network error before HTTP status). |
| `target` | **Logical** name from a closed enum in the service's config (`ms-graph`, `audit-service`, `s3`). Hostnames, IPs, and URLs are **forbidden**. |
| `outcome` | `success \| error \| timeout` |

**Required buckets:** same as §5.1.

**Forbidden:** `host`, `peer`, `url`, `full_url` labels.

### 5.3 Database client — `db_client_duration_seconds`

**Type:** histogram.

**Required labels:**

| Label | Values |
|---|---|
| `db_system` | `postgresql \| clickhouse \| redis \| mongodb` |
| `db_name` | database name |
| `operation` | `select \| insert \| update \| delete \| tx_commit \| tx_rollback \| ping` |
| `outcome` | `success \| error \| timeout` |

**Required buckets:** `0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5`.

**Forbidden:** SQL text, `query`, `statement`, `table` (free-text — high cardinality), full DSN values.

### 5.4 Cache — `cache_operation_duration_seconds`

**Type:** histogram.

**Required labels:**

| Label | Values |
|---|---|
| `cache_system` | `redis \| memcached \| caffeine \| in_process` |
| `cache_name` | logical cache name (`permissions`, `sessions`) |
| `operation` | `get \| set \| delete \| evict` |
| `result` | for `operation=get`: `hit \| miss`; for others: `success \| error` |

**Required buckets:** `0.0005, 0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5`.

### 5.5 Messaging — `messaging_operation_duration_seconds`

**Type:** histogram.

**Required labels:**

| Label | Values |
|---|---|
| `messaging_system` | `kafka \| redpanda \| rabbitmq \| sqs \| nats` |
| `destination` | topic / queue name |
| `operation` | `publish \| consume \| ack \| nack` |
| `outcome` | `success \| error \| timeout` |

**Required buckets:** `0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5`.

### 5.5.1 Messaging consumer lag — `messaging_consumer_lag`

Services with capability `messaging-consumer` **MUST** additionally emit:

**Metric:** `messaging_consumer_lag` (gauge).

**Required labels:**

| Label | Values |
|---|---|
| `messaging_system` | as in §5.5 |
| `destination` | topic / queue name |
| `consumer_group` | logical consumer group identifier |
| `partition` | partition / shard number as string (for systems that have partitions); empty `""` for unpartitioned queues |

**Value:** number of unprocessed messages currently behind the consumer (the natural lag metric of the underlying system). Units: **messages** (count). The metric name does not need a unit suffix because `_lag` is itself a recognized unit in the messaging context, but the value is dimensionless integer count.

**Update cadence:** sampled at least every 30 seconds.

**Forbidden:**
- Reporting lag in time units (seconds behind). Use messages.
- Per-message labels (`message_id`, `offset`).
- Emitting lag only when non-zero (the absence of a series is ambiguous; emit `0` explicitly).

**Rationale:** consumer lag is the most important health signal for a consumer service — it captures "are we keeping up?" in one number. Throughput and latency alone do not.

### 5.6 Scheduled/background jobs — `job_duration_seconds`

For periodic or cron-style jobs run inside the service (not k8s `CronJob` — for that, kube-state-metrics covers it).

**Type:** histogram.

**Required labels:**

| Label | Values |
|---|---|
| `job_name` | closed-enum name from service config |
| `outcome` | `success \| error \| skipped` |

**Required buckets:** `1, 5, 10, 30, 60, 300, 600, 1800, 3600` (jobs range from seconds to hours).

### 5.7 Bucket selection (Rationale)

Buckets are chosen for two competing goals:

1. **Resolution** at typical-case latencies (so p50, p95, p99 are meaningful, not interpolated across a 10× gap).
2. **Storage cost** (each bucket = a separate series; histogram with 11 buckets × 7 method × 20 status × 100 route = 154 000 series).

The canonical buckets in this section are non-negotiable. Adding a bucket affects the entire fleet's storage and PromQL `histogram_quantile` math. Bucket changes go through a separate RFC.

---

## 6. Tracing

This section defines the contract for distributed tracing. Tracing infrastructure (Tempo/Jaeger collector) **MAY NOT** be deployed at the time of writing; the conventions below apply when it is.

### 6.1 Trace context propagation (MUST)

Services **MUST** propagate the [W3C Trace Context](https://www.w3.org/TR/trace-context/) `traceparent` header on every outbound HTTP/gRPC call.

```
traceparent: 00-<trace-id 32 hex>-<span-id 16 hex>-<flags 2 hex>
tracestate: <vendor-specific, opaque>
```

Services **MUST** parse `traceparent` on inbound and, if no header is present, generate a fresh trace ID. They **MUST NOT** invent custom propagation headers (`X-Trace-ID`, `X-Request-ID` for tracing purposes).

### 6.2 Instrumentation coverage (SHOULD)

Services **SHOULD** emit a span for at least:

- Each inbound request handler.
- Each outbound HTTP / gRPC / database / cache / messaging call.
- Each significant unit of work within a request (≥ 50ms expected duration).

### 6.3 Span naming (MUST)

When tracing is enabled, span names **MUST** follow the pattern:

| Span kind | Name format | Example |
|---|---|---|
| HTTP server | `<METHOD> <route>` | `GET /users/:id` |
| HTTP client | `<METHOD> <target>` | `POST ms-graph` |
| DB | `<operation> <db_name>.<table-or-collection?>` | `select npp.users` |
| Cache | `<operation> <cache_name>` | `get sessions` |
| Messaging | `<destination> <operation>` | `npp-events publish` |

### 6.4 Sampling (SHOULD)

Services **SHOULD** sample traces at a head-rate **MUST NOT exceed 100%** in production. Sampling decisions **MUST** be carried in `traceparent` flags so child spans agree.

### 6.5 Log correlation (MUST when tracing is active)

When a span is active, the service **MUST** include `trace_id` and `span_id` fields in JSON logs emitted within the span.

---

## 7. Logs

### 7.1 Format (MUST)

Logs **MUST** be JSON, one object per line, written to **stdout**. Logs to files, syslog, or remote sinks directly from the application are **forbidden** — the platform collects from stdout.

A log line **MUST NOT** exceed **64 KiB**. Larger payloads **MUST** be truncated by the application; the truncation indicator field `truncated: true` **MUST** be set.

### 7.2 Required fields (MUST)

| Field | Type | Format |
|---|---|---|
| `ts` | string | RFC 3339 / ISO 8601, UTC, nanosecond precision. Trailing `Z`. |
| `level` | string | Lowercase enum: `debug \| info \| warn \| error`. |
| `msg` | string | Plain text. **MUST NOT** contain JSON, control characters except `\n`, ANSI escape codes. |

Alternative names (`time`, `@timestamp`, `timestamp`, `tstamp`, `WARN`, `WARNING`, `ERR`, `FATAL`, `TRACE`) are **forbidden**. Use exactly these three keys.

### 7.3 Reserved optional fields (SHOULD when applicable)

| Field | Type | When |
|---|---|---|
| `service` | string (kebab-case) | The service identifies itself. Otherwise platform injects from pod labels. |
| `error` | object `{type, message, stack}` | When `level=error`. `stack` is `\n`-joined string, total ≤ 16 KiB. |
| `trace_id` | string (W3C 32-hex) | When a span is active. |
| `span_id` | string (W3C 16-hex) | When a span is active. |
| `correlation_id` | string | When propagated from an inbound request. |

### 7.4 Domain fields (MUST be snake_case)

Any other field a service emits **MUST**:
- Be at the root of the JSON object (no nesting beyond `error`).
- Be `snake_case` matching `^[a-z][a-z0-9_]{0,40}$`.
- Have a stable type across log lines (a field that is sometimes string and sometimes number is forbidden).

### 7.5 Forbidden patterns (MUST NOT)

| Pattern | Replacement |
|---|---|
| camelCase fields (`correlationId`) | `correlation_id` |
| PascalCase (`RequestPath`) | `request_path` |
| Dot-naming (`req.method`, `record.error_severity`) | flat: `request_method`, `error_severity` |
| Plain-text log lines (anything that is not valid JSON) | JSON |
| Multi-line stack traces as separate lines | embed in `error.stack` |
| JSON-stringified blob in `msg` | emit as a sibling field |
| Including newlines in field values | replace with `\n` or escape |
| Logging the same event multiple times at multiple levels | log once at the highest applicable level |

### 7.6 Secrets and PII (MUST NOT)

Logs **MUST NOT** include, under any field name or in `msg`:

- Passwords, API keys, OAuth tokens, JWT tokens (full or partial), session cookies, signing secrets, TLS private keys.
- Full credit card numbers, CVV, full IBAN, full national ID numbers.
- Email addresses (use a redacted form `usr_<hash>@<domain>` if identification is needed).
- Phone numbers, full names, physical addresses.
- Authorization headers, `Cookie` headers verbatim.
- Request or response bodies that may contain any of the above.

The list is non-exhaustive. The rule of thumb: **if it identifies a real person or unlocks a real account, it does not belong in logs**.

### 7.7 Sampling (MUST NOT for error level)

Logs at `level=error` **MUST NOT** be sampled or dropped by the application. They are the only signal an operator gets for failures.

Logs at `level=info` **MAY** be sampled if a single event class generates more than **100 events/second per pod** sustained. Sampling **MUST** be deterministic (e.g., every Nth event) and **MUST** be documented in code comments.

### 7.8 Log volume budget (SHOULD)

A service **SHOULD NOT** sustain more than **10 MiB/s of log output per pod**. Above this, the platform collector starts dropping. If a service legitimately needs more, it **MUST** notify the Platform team.

---

## 8. Alerts (VMRule)

Alerts about a service **MAY** be authored by either the service team or the Platform team. Regardless of authorship, they **MUST** conform to this section.

### 8.1 Severity (MUST)

| Severity | Definition | Reaction |
|---|---|---|
| `critical` | Service is degraded or down, users affected, SLO at risk. | On-call paged, any hour. |
| `warning` | Deviation from healthy state, users not yet affected. | Investigated next business day. |
| `info` | Contextual event (deploy, rebalance, manual operation). | Not paged. Logged to a low-priority channel. |

Other values (`p0`, `p1`, `p2`, `p3`, `high`, `low`, `urgent`, `fatal`, `disaster`, `error`, `notice`, `emergency`) are **forbidden**.

### 8.2 Required labels (MUST)

Every alert rule's `labels` block **MUST** include exactly these:

| Label | Value | Source |
|---|---|---|
| `severity` | from §8.1 enum | author |
| `service` | `app.kubernetes.io/name` of the monitored service | author |

The platform-side relabel injects `environment` and `namespace`, which the AM uses for routing. The author **MUST NOT** hardcode them in alert labels.

### 8.3 Required annotations (MUST)

| Annotation | Limit | Content |
|---|---|---|
| `summary` | ≤ 100 chars | One sentence: what happened. Goes into alert title. |
| `description` | ≤ 500 chars | What is wrong, with values: `{{ $value }}`, `{{ $labels.* }}`. |

`runbook_url` **SHOULD** be present for `severity=critical`. It will be **REQUIRED** in a future version of this standard.

### 8.4 Alert name (MUST)

Regex: `^[A-Z][a-zA-Z0-9]{0,49}$`. Length ≤ 50 characters.

**Format:** `<ServicePascalCase><DescriptionPascalCase>`.

`<ServicePascalCase>` is derived from `app.kubernetes.io/name`:
- `du-gateway` → `DuGateway`.
- `audit-service` → `AuditService`.
- `npp-api` → `NppApi`.

Abbreviations are written as regular words (`Npp`, not `NPP`).

```
# OK
NppApiHighErrorRate
DuGatewayP95LatencyHigh
AuditServiceEventsDropped

# NOT OK
high_error_rate                   # snake_case is for metrics, not alerts
HighErrorRate                     # missing service prefix
NPPAPIHighErrorRate               # uppercase abbreviation
NppApi_HighErrorRate              # mixed style
NppApi High Error Rate            # spaces
```

### 8.5 Expression (MUST)

Every `expr` **MUST**:

1. Include a `namespace=` or `namespace=~` selector.
2. Reference at least one metric whose name conforms to §3 or §5.
3. Not use `absent()` without an accompanying alert that fires when the metric is present (otherwise a metric rename silently breaks the alert).

```promql
# OK
up{namespace="npp", app_kubernetes_io_name="npp-api"} == 0

# NOT OK — no namespace selector
up{app_kubernetes_io_name="npp-api"} == 0
```

### 8.6 Pending duration (MUST)

Every alert **MUST** declare `for:`. Minimum values:

| Severity / kind | Minimum `for:` |
|---|---|
| `critical` availability (down / not ready) | `2m` |
| `critical` rate / latency / saturation | `5m` |
| `warning` | `10m` |
| `info` | `5m` |

Exception: `for: 0s` is permitted only for conditions that **cannot** be a spurious spike, e.g. `kube_deployment_status_replicas_available == 0`. Approval from Platform is required.

### 8.7 Group name (MUST)

Format: `<service>.<category>`. `<service>` is `app.kubernetes.io/name` (kebab-case). `<category>` is one of:

| Category | Contents |
|---|---|
| `availability` | Service alive, pods running, ready replicas |
| `golden-signals` | RED: rate, errors, duration |
| `saturation` | CPU, memory, GC, pools |
| `dependencies` | DB, cache, queue, external APIs |
| `business` | Domain KPIs, not infrastructure |

Free-form categories (`misc`, `other`, `tmp`, `experimental`) are **forbidden**.

### 8.8 Mandatory alerts (MUST, by capability)

Every conformant service **MUST** ship the alerts in the **Universal** block plus every block that matches a declared capability (§1.4). Services **SHOULD** add more.

`<Svc>` denotes the service's PascalCase name (e.g. `NppApi`, `DuGateway`).

#### Universal (every service, regardless of capability)

| Alert | Severity | Category |
|---|---|---|
| `<Svc>Down` | critical | availability |
| `<Svc>PodCrashLoop` | critical | availability |
| `<Svc>HighMemoryUsage` (working set > 90% of limit) | warning | saturation |

#### Capability `http-server`

| Alert | Severity | Category |
|---|---|---|
| `<Svc>HighErrorRate` (5xx ratio > 5% for 5m) | critical | golden-signals |
| `<Svc>HighLatency` (p95 over service-specific threshold for 10m) | critical | golden-signals |

#### Capability `http-client`

| Alert | Severity | Category |
|---|---|---|
| `<Svc>OutboundErrors` (`http_client_duration_seconds_count{outcome!="success"}` rate > threshold) | warning | dependencies |

#### Capability `db-client`

| Alert | Severity | Category |
|---|---|---|
| `<Svc>DbHighLatency` (p95 of `db_client_duration_seconds` over threshold) | warning | dependencies |
| `<Svc>DbErrors` (`db_client_duration_seconds_count{outcome="error"}` rate > 0 for 5m) | critical | dependencies |

#### Capability `cache-client`

| Alert | Severity | Category |
|---|---|---|
| `<Svc>CacheLowHitRatio` (hit ratio < 90% for 30m, only meaningful if traffic > threshold) | warning | dependencies |
| `<Svc>CacheErrors` (`cache_operation_duration_seconds_count{result="error"}` rate > 0 for 5m) | warning | dependencies |

#### Capability `messaging-publisher`

| Alert | Severity | Category |
|---|---|---|
| `<Svc>PublishErrors` (`messaging_operation_duration_seconds_count{operation="publish",outcome!="success"}` rate > threshold) | critical | dependencies |

#### Capability `messaging-consumer`

| Alert | Severity | Category |
|---|---|---|
| `<Svc>HighConsumerLag` (`messaging_consumer_lag` over service-specific threshold for 10m) | critical | golden-signals |
| `<Svc>ConsumerErrors` (`messaging_operation_duration_seconds_count{operation="consume",outcome!="success"}` rate > threshold) | critical | golden-signals |
| `<Svc>ConsumerStalled` (rate of `messaging_operation_duration_seconds_count{operation="consume"}` = 0 while `messaging_consumer_lag` > 0 for 5m) | critical | availability |

#### Capability `job-runner`

| Alert | Severity | Category |
|---|---|---|
| `<Svc>JobFailures` (`job_duration_seconds_count{outcome="error"}` rate > 0) | warning | golden-signals |
| `<Svc>JobOverdue` (no `job_duration_seconds_count` increase within expected interval for a given `job_name`) | warning | golden-signals |

A service that lacks any applicable mandatory alert **MUST** record a waiver (§0.3).

### 8.9 Recording rules (MUST follow Prom convention)

Recording rules **MUST** follow the [Prometheus recording rule naming convention](https://prometheus.io/docs/practices/rules/#naming-and-aggregation):

```
<level>:<metric>:<aggregation>
```

| Field | Example |
|---|---|
| `<level>` | aggregation level: `pod`, `namespace`, `service`, `cluster` |
| `<metric>` | original metric name (without `_bucket`/`_sum`/`_count`) |
| `<aggregation>` | the operation: `rate5m`, `sum_rate5m`, `histogram_quantile_95` |

```
# OK
namespace:http_handler_duration_seconds:histogram_quantile95_5m
service:http_handler_duration_seconds:sum_rate5m

# NOT OK
my_recording_rule
http_p95
```

---

## 9. Dashboards

Each service **MUST** have **exactly one** Grafana dashboard. The Platform team may produce it on the service team's behalf; the convention applies regardless of authorship.

### 9.1 Layout (MUST)

The dashboard **MUST** be divided into five rows, in this fixed order:

1. **Availability** (expanded by default)
2. **Golden Signals** (expanded by default)
3. **Saturation** (collapsed)
4. **Dependencies** (collapsed)
5. **Business** (collapsed)

If a row has no applicable panels (no dependencies, no business metrics), the row **MUST** still exist with a single text panel reading `No external dependencies` / `No business metrics`. Absence is intentional, not forgotten.

### 9.2 Minimum required panels per row (MUST)

#### 9.2.1 Availability

- `Replicas Ready` (stat) — `count(up{namespace="$ns", app_kubernetes_io_name="$svc"}==1) / count(up{namespace="$ns", app_kubernetes_io_name="$svc"})`. Unit %, thresholds green ≥ 1.0.
- `Pod Restarts (1h)` (stat) — `sum(increase(kube_pod_container_status_restarts_total{namespace="$ns", pod=~"$svc-.*"}[1h]))`. Thresholds green = 0, yellow ≥ 1, red ≥ 3.
- `Pod up` (timeseries) — `up{namespace="$ns", app_kubernetes_io_name="$svc"}` by pod.

#### 9.2.2 Golden Signals — content depends on declared capability (§1.4)

A service with **multiple capabilities** shows the panels for each, stacked top-to-bottom. The four stats at the top of the row (throughput / error rate / p95 / p99) come from the service's **primary** capability — the one users interact with most directly. The mapping:

- `http-server` → primary if declared.
- `messaging-consumer` → primary if `http-server` is not declared.
- `job-runner` → primary if neither of the above is declared.

##### If capability `http-server`

- `Throughput` (stat, RPM) — `sum(rate(http_handler_duration_seconds_count{namespace="$ns"}[5m])) * 60`.
- `Error rate` (stat, %) — `sum(rate(http_handler_duration_seconds_count{namespace="$ns", status_code=~"5.."}[5m])) / sum(rate(http_handler_duration_seconds_count{namespace="$ns"}[5m]))`. Thresholds: green < 1%, yellow ≥ 1%, red ≥ 5%.
- `Latency p95` (stat, seconds).
- `Latency p99` (stat, seconds).
- `Throughput by status` (timeseries, stacked, RPM).
- `Latency p50 / p95 / p99` (timeseries, seconds).

##### If capability `messaging-consumer`

- `Consume rate` (stat, msg/min) — `sum(rate(messaging_operation_duration_seconds_count{namespace="$ns",operation="consume"}[5m])) * 60`.
- `Consumer lag` (stat, messages) — `sum(messaging_consumer_lag{namespace="$ns"})`. Thresholds service-specific.
- `Consume error rate` (stat, %) — `sum(rate(messaging_operation_duration_seconds_count{namespace="$ns",operation="consume",outcome!="success"}[5m])) / sum(rate(messaging_operation_duration_seconds_count{namespace="$ns",operation="consume"}[5m]))`.
- `Consume latency p95` (stat, seconds).
- `Consume rate over time by destination` (timeseries, msg/min).
- `Consumer lag over time by destination/partition` (timeseries, messages).

##### If capability `messaging-publisher`

- `Publish rate` (stat, msg/min) — `sum(rate(messaging_operation_duration_seconds_count{namespace="$ns",operation="publish"}[5m])) * 60`.
- `Publish error rate` (stat, %).
- `Publish latency p95` (stat, seconds).
- `Publish rate over time by destination` (timeseries, msg/min).

##### If capability `job-runner`

- `Jobs run per hour` (stat) — `sum(rate(job_duration_seconds_count{namespace="$ns"}[1h])) * 3600`.
- `Job failure ratio` (stat, %) — `sum(rate(job_duration_seconds_count{namespace="$ns",outcome="error"}[1h])) / sum(rate(job_duration_seconds_count{namespace="$ns"}[1h]))`.
- `Job duration p95 by job_name` (timeseries, seconds).
- `Job outcomes by job_name` (timeseries, stacked).

#### 9.2.3 Saturation

- `CPU usage` (stat, % of limit).
- `Memory usage` (stat, % of limit).
- `CPU by pod` (timeseries, cores; request and limit drawn as horizontal markers).
- `Memory by pod` (timeseries, bytes; request and limit drawn as horizontal markers).
- Application-specific: pool, JVM heap, GC, event loop lag (timeseries).

#### 9.2.4 Dependencies

Per external dependency (DB, cache, queue, external API):

- `<dep> request rate by target` (timeseries, RPM).
- `<dep> error rate by target` (timeseries, %).
- `<dep> latency p95 by target` (timeseries, seconds).

#### 9.2.5 Business

Service-specific KPI panels using `<service>_*` custom metrics from §3. No HTTP / DB / JVM metrics in this row.

### 9.3 Display units (MUST)

| Quantity | Unit |
|---|---|
| HTTP throughput | req/min (RPM); PromQL multiplies `rate(...) * 60` at the panel level |
| Durations | seconds (Grafana auto-converts to ms/µs) |
| Ratios | percent |
| Sizes | bytes |

Time-based queries **MUST NOT** use raw milliseconds even when the underlying metric is histogram-derived.

### 9.4 Template variables (MUST)

The dashboard **MUST** include the following variables, in this order:

| Name | Type | Source | Visibility |
|---|---|---|---|
| `datasource` | `datasource` (prometheus) | — | visible |
| `environment` | `query`: `label_values(up, environment)` | — | visible, default `prod` |
| `ns` | `constant` | namespace where the service runs | **hidden** |
| `svc` | `constant` | `app.kubernetes.io/name` of the service | **hidden** |
| `pod` | `query`: `label_values(up{namespace="$ns", app_kubernetes_io_name="$svc", environment="$environment"}, pod)` | — | visible, multi, includeAll |

Every panel's PromQL **MUST** reference `namespace="$ns"`.

### 9.5 UID (MUST)

The dashboard `uid` **MUST** equal `<service>` (kebab-case `app.kubernetes.io/name`). The UID is permanent — it does not change when the service is renamed. A renamed service records its old UID in dashboard annotation `digitaluni.io/renamed-from: <old-name>`.

### 9.6 Forbidden patterns (MUST NOT)

- More than one dashboard per service (split files).
- Hardcoded namespace or service name in any PromQL — use `$ns`, `$svc`.
- Tables that show long lists of pods, requests, traces; link out to logs / traces instead.
- Free-text panels with operator notes, runbook content, or "TODO" markers.
- Time range default longer than 6 hours (heavy queries on dashboard open).
- Panels with no `description` (Grafana panel-level documentation).

---

## 10. Service Level Objectives (RECOMMENDED)

This section is non-normative in version 0; planned to become normative in version 1.

Every service **SHOULD** declare at least one SLO:

```yaml
# slo.yaml (committed to service repo)
service: du-gateway
slos:
  - name: availability
    target: 0.999
    indicator: |
      sum(rate(http_handler_duration_seconds_count{status_code!~"5.."}[28d]))
      / sum(rate(http_handler_duration_seconds_count[28d]))
  - name: latency_p99_under_1s
    target: 0.99
    indicator: |
      sum(rate(http_handler_duration_seconds_bucket{le="1"}[28d]))
      / sum(rate(http_handler_duration_seconds_count[28d]))
```

Format and tooling for automated SLO burn-rate alerts will be specified in a follow-up RFC.

---

## 11. Glossary

| Term | Meaning |
|---|---|
| **Service** | A deployed application owned by one team, identified by `app.kubernetes.io/name`. |
| **Project** | Repository or repository group containing a service's source code and Helm chart. |
| **Platform team** | The team that operates the cluster, the observability stack, this standard. |
| **Canonical metric** | A reserved metric name (§5) every service implementing the capability **MUST** emit under exactly this name. |
| **Conformant** | All applicable **MUST**s of this standard are satisfied. |
| **Waiver** | An explicit, recorded deviation from a **MUST**, approved by the Platform team. |
| **PII** | Personally Identifiable Information — anything that identifies a real person, partially or fully. |

---

## 12. Change log

| Version | Date | Changes |
|---|---|---|
| 0.1.0 | 2026-05-17 | Initial draft. |
| 0.2.0 | 2026-05-17 | Added §1.4 service capabilities, §3.6 build_info metric, §5.0 capability→metric mapping, §5.5.1 consumer lag, §8.8 mandatory alerts per capability, §9.2.2 golden signals per capability. |

---

This document is **language- and framework-agnostic**. The standard describes what metrics, logs, and traces look like on the wire — names, labels, buckets, JSON field shapes — not how they are produced. Every conformant service emits the same byte-level output regardless of whether it is written in Java, Node, Go, Python, Rust, or any other stack.

If a chosen library publishes telemetry under non-conformant names, defaults, or formats, the service **MUST** reconfigure, wrap, or adapt the library's output so that the wire-level result conforms. The cost of this adaptation is a property of the library choice and is borne by the service team.
