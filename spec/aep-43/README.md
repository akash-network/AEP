---
aep: 43
title: "Workload Utilization Metrics"
description: "Per-lease workload utilization metrics surfaced by every provider and aggregated by clients for a unified multi-provider view across the Akash Network."
author: Maxime Beauchamp (@baktun14) Anil Murty (@anilmurty) Artur Troian (@troian)
status: Draft
type: Standard
category: Interface
created: 2024-12-01
updated: 2026-05-11
estimated-completion: 2026-07-15
discussions-to: https://github.com/orgs/akash-network/discussions
roadmap: minor
---

## Abstract

This AEP proposes a standard for collecting, exposing, and consuming **per-lease workload utilization metrics** on the Akash Network. It defines (a) the set of metrics every provider must surface for each lease it hosts, (b) the provider-side architecture for collection and storage, (c) a versioned Provider API for tenants and clients (e.g. Akash Console) to query these metrics, (d) the authentication and authorization model that ties metric access to lease ownership, and (e) the aggregation pattern clients use to present a unified view across the multiple providers a tenant may have leases with.

The goal is parity with — and where reasonable, an improvement over — the per-service metrics experience offered by centralized cloud platforms (Railway, Render, Fly.io, Heroku), while preserving Akash's decentralized, multi-tenant, multi-provider topology.

## Motivation

Tenants on Akash today have **no first-class way to observe how much of their allocated capacity their workloads are actually consuming**. The on-chain marketplace records the resources requested in the SDL — CPU units, RAM, GPU units and VRAM, storage — but the network exposes nothing about realized utilization back to the tenant.

This creates four concrete user-experience problems:

1. **No cost optimization signal.** A tenant who has provisioned an 80GB H100 lease has no way to discover that their workload only ever touches 12GB of VRAM. They cannot right-size confidently and tend to over-provision "to be safe," paying for capacity they do not use.
2. **No peak-load awareness.** Tenants cannot detect when their workload is running hot against the limits of their SDL and is therefore at risk of throttling, OOM-kills, or degraded performance, until users report symptoms.
3. **No diagnostic baseline.** When something does go wrong, tenants have neither a utilization timeline nor a deployment-event overlay to investigate against. Today the only signal is `lease-events` plus container logs.
4. **No comparison signal across providers.** Tenants who run the same workload across multiple providers cannot tell whether one provider's host is consistently performing worse or differently than another's.

Centralized platforms have solved this with a per-service Metrics tab that shows CPU, memory, disk, and network over time (e.g. [Railway's Metrics page](https://docs.railway.com/observability/metrics)). Akash needs an equivalent — but because no single party operates the entire fleet, the design must be **federated**: each provider produces metrics for the leases *it hosts*, and clients are responsible for aggregating across the providers a tenant is using.

This proposal is also a prerequisite for several downstream features that have been discussed across other AEPs and community channels:

- **Workload-driven autoscaling** of replicas inside a lease (a future AEP will depend on this metric surface as its trigger source).
- **Provider quality scoring and SLAs** — tenants and on-chain scoring systems both need a verifiable signal of how a provider's hosts performed under load.
- **Provider Console capacity dashboards** — AEP-32 lists "observability tooling set up and reporting (metrics & logs)" as a planned addition, and reuses the same collection layer.

## Goals

This AEP MUST deliver:

1. A defined, minimum set of utilization metrics every conformant provider exposes for every lease it hosts, covering CPU, memory, storage, network, and (where applicable) GPU.
2. A versioned, authenticated Provider REST/gRPC endpoint for querying time-series metric data scoped to a tenant's own leases.
3. A reference provider-side implementation in `akash-network/provider` that runs as part of the standard provider install with no manual operator configuration required.
4. A retention guarantee of at least **30 days** of data at a defined minimum resolution.
5. JWT-based authorization ([AEP-64](../aep-64)) so tenants do not need certificates to query metrics, and so granular per-deployment / per-service access tokens can be issued.

## Non-Goals

This AEP does **not** address:

1. **Application-level metrics** (request latency, error rates, business KPIs). These remain the tenant's responsibility; the standard answer is to ship them to a third-party observability stack via OpenTelemetry. A separate AEP on Workload Log Forwarding overlaps here and is complementary.
2. **Log aggregation.** Log access already exists through the `LeaseLogs` provider endpoint. This AEP intentionally limits scope to numeric time-series.
3. **Distributed tracing.**
4. **Alerting.** A first-class alerts/notifications service is tracked under separate Console-side proposals. This AEP exposes the metric surface those alerts can be built on.
5. **On-chain settlement based on utilization.** The marketplace continues to settle on *allocated* resources per the SDL. Reconciling settlement with realized usage is a tokenomics question outside this AEP's scope.

## Terminology

- **Lease** — an on-chain agreement between a tenant and a provider, identified by `(owner, dseq, gseq, oseq, provider)`.
- **Service** — a named container group within a lease's SDL.
- **Replica** — an individual running instance (Kubernetes Pod) of a service. A service with `count: N` runs N replicas.
- **TSDB** — time-series database; the provider-side store for scraped metric samples.
- **Exporter** — a process that exposes Prometheus-format metrics over HTTP.
- **Scrape** — a periodic pull of metrics from an exporter into the TSDB.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Provider Kubernetes Cluster                      │
│                                                                          │
│  ┌──────────────┐   ┌──────────────────┐    ┌────────────────────────┐  │
│  │  cAdvisor    │   │ kube-state-      │    │ dcgm-exporter          │  │
│  │  (in kubelet)│   │ metrics          │    │ (GPU nodes only)       │  │
│  └──────┬───────┘   └────────┬─────────┘    └────────────┬───────────┘  │
│         │                    │                            │              │
│         │  scrape (Prom fmt) │                            │              │
│         ▼                    ▼                            ▼              │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │              metrics-operator (NEW)                                │ │
│  │   - manages exporters lifecycle                                    │ │
│  │   - manages TSDB lifecycle                                         │ │
│  │   - applies Akash labels on scrape (owner,dseq,gseq,oseq,service)  │ │
│  └─────────────────────────┬──────────────────────────────────────────┘ │
│                            │                                             │
│                            ▼                                             │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │   TSDB (VictoriaMetrics single-node, or Prometheus)                │ │
│  │   - 30 day retention                                               │ │
│  │   - per-tenant cardinality limits                                  │ │
│  │   - lease-scoped query enforcement at proxy layer                  │ │
│  └─────────────────────────┬──────────────────────────────────────────┘ │
│                            │                                             │
│                            ▼                                             │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │   provider gateway (akash-network/provider/gateway)                │ │
│  │   GET /lease/{dseq}/{gseq}/{oseq}/metrics                          │ │
│  │   - validates JWT (AEP-64)                                         │ │
│  │   - enforces ownership: JWT.iss == lease.owner                     │ │
│  │   - rewrites tenant query into a label-filtered TSDB query         │ │
│  └─────────────────────────┬──────────────────────────────────────────┘ │
└────────────────────────────┼─────────────────────────────────────────────┘
                             │ HTTPS
                             ▼
                 ┌───────────────────────┐
                 │  Akash Console        │
                 │  (or any client)      │
                 │  - fans out across    │
                 │    all providers      │
                 │    for a deployment   │
                 │  - merges series      │
                 │  - draws charts       │
                 └───────────────────────┘
```

Five components, three of which are off-the-shelf:

1. **Metric sources** (off-the-shelf): kubelet/cAdvisor, kube-state-metrics, dcgm-exporter, node_exporter, CSI driver metrics.
2. **metrics-operator** (new): Kubernetes operator inside the provider's cluster that manages the exporter and TSDB lifecycle and ensures Akash-specific scrape relabeling is in place.
3. **TSDB** (off-the-shelf): VictoriaMetrics single-node is the reference choice; Prometheus is an allowed alternative.
4. **Provider gateway extension** (new): a new REST handler in `akash-network/provider/gateway` plus a new gRPC service in `akash-network/akash-api` that proxies tenant queries to the TSDB with mandatory label enforcement.
5. **Client aggregation** (new, in Akash Console initially): logic to identify all providers hosting a given deployment and merge the per-provider metric responses.

## Metrics Catalog

The following metrics MUST be exposed by every conformant provider for every active lease it hosts. Metric names are normalized at the Provider API layer; the underlying exporter source is implementation-defined and shown for reference.

### CPU

| Metric name | Type | Unit | Description | Reference source |
| --- | --- | --- | --- | --- |
| `akash_lease_cpu_usage_cores` | gauge | cores | Instantaneous CPU usage in fractional cores | `container_cpu_usage_seconds_total` (rate) |
| `akash_lease_cpu_throttled_seconds` | counter | seconds | Cumulative CPU throttling time | `container_cpu_cfs_throttled_seconds_total` |
| `akash_lease_cpu_limit_cores` | gauge | cores | CPU limit from SDL | `kube_pod_container_resource_limits` |

### Memory

| Metric name | Type | Unit | Description | Reference source |
| --- | --- | --- | --- | --- |
| `akash_lease_memory_working_set_bytes` | gauge | bytes | Working set memory (excludes inactive page cache); the value to alert on | `container_memory_working_set_bytes` |
| `akash_lease_memory_rss_bytes` | gauge | bytes | Resident set size | `container_memory_rss` |
| `akash_lease_memory_limit_bytes` | gauge | bytes | Memory limit from SDL | `kube_pod_container_resource_limits` |
| `akash_lease_memory_oom_kills_total` | counter | count | Cumulative OOM-kill events | `container_oom_events_total` |

### Storage

| Metric name | Type | Unit | Description | Reference source |
| --- | --- | --- | --- | --- |
| `akash_lease_filesystem_usage_bytes` | gauge | bytes | Used filesystem bytes per volume | `kubelet_volume_stats_used_bytes` |
| `akash_lease_filesystem_capacity_bytes` | gauge | bytes | Provisioned capacity per volume | `kubelet_volume_stats_capacity_bytes` |
| `akash_lease_filesystem_inodes_used` | gauge | count | Used inodes per volume | `kubelet_volume_stats_inodes_used` |

### Network

| Metric name | Type | Unit | Description | Reference source |
| --- | --- | --- | --- | --- |
| `akash_lease_network_rx_bytes_total` | counter | bytes | Cumulative bytes received | `container_network_receive_bytes_total` |
| `akash_lease_network_tx_bytes_total` | counter | bytes | Cumulative bytes transmitted | `container_network_transmit_bytes_total` |
| `akash_lease_network_rx_errors_total` | counter | count | Cumulative receive errors | `container_network_receive_errors_total` |
| `akash_lease_network_tx_errors_total` | counter | count | Cumulative transmit errors | `container_network_transmit_errors_total` |

Egress and ingress MUST be exposed separately and SHOULD distinguish public vs. private (in-cluster) traffic where the provider can do so cheaply.

### GPU (NVIDIA, required where GPUs are present)

| Metric name | Type | Unit | Description | Reference source |
| --- | --- | --- | --- | --- |
| `akash_lease_gpu_utilization_percent` | gauge | % (0–100) | SM utilization | `DCGM_FI_DEV_GPU_UTIL` |
| `akash_lease_gpu_memory_used_bytes` | gauge | bytes | VRAM in use | `DCGM_FI_DEV_FB_USED` |
| `akash_lease_gpu_memory_total_bytes` | gauge | bytes | VRAM installed | `DCGM_FI_DEV_FB_FREE + DCGM_FI_DEV_FB_USED` |
| `akash_lease_gpu_memory_copy_utilization_percent` | gauge | % | Memory-controller utilization | `DCGM_FI_DEV_MEM_COPY_UTIL` |
| `akash_lease_gpu_power_watts` | gauge | watts | Instantaneous board power | `DCGM_FI_DEV_POWER_USAGE` |
| `akash_lease_gpu_temperature_celsius` | gauge | °C | GPU die temperature | `DCGM_FI_DEV_GPU_TEMP` |
| `akash_lease_gpu_sm_clock_mhz` | gauge | MHz | SM clock frequency | `DCGM_FI_DEV_SM_CLOCK` |

AMD and Intel GPUs are out of scope for v1 of this AEP but the metric naming scheme is intentionally vendor-neutral so that `rocm-smi-exporter` or similar can be wired in later without API breakage.

### Labels

Every series MUST carry the following labels:

- `owner` — Akash address of the lease owner (`akash1...`)
- `dseq` — deployment sequence
- `gseq` — group sequence
- `oseq` — order sequence
- `provider` — provider Akash address
- `service` — service name from the SDL
- `replica` — replica index (`0`-based, stable within a service's lifetime)

GPU series additionally carry:

- `gpu_index` — local GPU index on the host (`0`, `1`, …)
- `gpu_model` — e.g. `H100`, `A100-80GB`, `RTX-4090`
- `gpu_uuid` — DCGM UUID

These labels are the mandatory contract for any tenant or aggregator (Console) consuming the API; the provider MAY add internal labels (node name, namespace, etc.) but MUST NOT expose those to the tenant.

## API Specification

### Endpoint

```
GET /lease/{dseq}/{gseq}/{oseq}/metrics
```

All metric queries are scoped to a single lease. The path mirrors the existing `LeaseStatus`, `LeaseLogs`, `LeaseEvents`, and `LeaseShell` endpoints in the provider gateway.

### Authentication

JWT per [AEP-64](../aep-64). The token's `iss` claim MUST match the lease owner on chain, and the token's `leases` claim MUST grant either:

- `access: full` with the `metrics` scope, or
- `access: granular` with a permission entry naming this provider and (at minimum) this `dseq`, scoped to `metrics`.

This AEP adds a new value `metrics` to the JWT `scope` enum defined in AEP-64. The full updated enum is:

```
"send-manifest", "get-manifest", "logs", "shell", "events",
"status", "restart", "hostname-migrate", "ip-migrate", "metrics"
```

mTLS clients (pre-AEP-64) MUST also be supported for backward compatibility, with the existing cert-based ownership check.

### Query parameters

| Parameter | Required | Type | Description |
| --- | --- | --- | --- |
| `metric` | yes | string (comma-separated) | One or more metric names from the catalog above, OR the special value `all`. |
| `start` | yes | RFC3339 timestamp or Unix seconds | Start of the query range. |
| `end` | yes | RFC3339 timestamp or Unix seconds | End of the query range. |
| `step` | no | duration (e.g. `30s`, `5m`) | Sample spacing. Defaults per resolution table below. |
| `service` | no | string (comma-separated) | Restrict to specific services within the lease. |
| `replica` | no | string (comma-separated) | Restrict to specific replicas. |
| `aggregation` | no | enum: `sum`, `avg`, `max`, `replica` | How to combine replicas for multi-replica services. Default: `replica` (returns one series per replica). `sum` returns a single combined series. |
| `format` | no | enum: `json`, `prometheus` | Response format. Default `json`. `prometheus` returns standard Prometheus query_range JSON for tools that already speak it. |

### Response (`format=json`)

```json
{
  "lease": {
    "owner": "akash1...",
    "dseq": 1234567,
    "gseq": 1,
    "oseq": 1,
    "provider": "akash1..."
  },
  "range": {
    "start": "2026-05-10T00:00:00Z",
    "end":   "2026-05-11T00:00:00Z",
    "step":  "1m"
  },
  "series": [
    {
      "metric": "akash_lease_cpu_usage_cores",
      "labels": {
        "service": "web",
        "replica": "0"
      },
      "samples": [
        { "t": "2026-05-10T00:00:00Z", "v": 0.42 },
        { "t": "2026-05-10T00:01:00Z", "v": 0.45 }
      ]
    }
  ],
  "warnings": []
}
```

The `warnings` array surfaces non-fatal conditions (e.g. partial data because retention was hit, downsampling applied, cardinality limit hit).

### Resolution table

| Range size | Default `step` | Minimum `step` |
| --- | --- | --- |
| ≤ 1 hour | `15s` | `15s` |
| ≤ 24 hours | `1m` | `30s` |
| ≤ 7 days | `5m` | `1m` |
| ≤ 30 days | `15m` | `5m` |

Providers MAY offer finer resolution; they MUST NOT offer coarser data than the table allows.

### Errors

Standard provider gateway error envelope. Notable error codes:

- `401 unauthorized` — JWT signature invalid or expired.
- `403 forbidden` — JWT valid but does not grant `metrics` scope for this lease.
- `404 not_found` — lease not hosted on this provider.
- `416 range_not_satisfiable` — requested range exceeds retention.
- `429 too_many_requests` — per-tenant query rate or cardinality limit exceeded.
- `503 service_unavailable` — metrics subsystem degraded or disabled on this provider.

### Discoverability

A new field is added to the existing `ProviderStatus` response:

```json
"features": {
  "metrics": {
    "enabled": true,
    "retention_days": 30,
    "min_step": "15s",
    "gpu_metrics": true
  }
}
```

This lets Console gracefully hide the Metrics tab when a particular provider has disabled the feature, rather than blindly issuing requests that 503.

## Provider-Side Implementation

### Changes to `akash-network/provider`

1. **New package: `operator/metrics`.** Follows the pattern of `operator/inventory` and `operator/hostname`. Responsibilities:
   - Deploy the chosen TSDB (default: VictoriaMetrics single-node) as a `StatefulSet` in the provider's control namespace.
   - Deploy `kube-state-metrics`, `node_exporter`, and `dcgm-exporter` (where GPU nodes exist) as DaemonSets.
   - Maintain a scrape config that relabels every series with the seven mandatory Akash labels by joining on the Akash `manifests.akash.network` CRD that already owns each tenant namespace.
   - Reconcile retention, cardinality limits, and per-tenant rate limits from provider config.
2. **New package: `cluster/kube/metrics_client.go`.** Wraps the TSDB query API behind a Go interface (`MetricsClient`) so the gateway is decoupled from whether the TSDB is VictoriaMetrics or Prometheus. The interface MUST enforce label-filter injection — callers pass `(owner, dseq, gseq, oseq)` separately from the metric expression, and the client is responsible for splicing those into a safe label selector. There is no path that lets a tenant-supplied raw PromQL string reach the TSDB.
3. **New REST handlers in `gateway/rest`.** Add `GET /lease/{dseq}/{gseq}/{oseq}/metrics` and the corresponding routing. Reuse the existing JWT validator middleware introduced by AEP-64. Add a feature flag (`metrics_enabled`) that defaults to `true` once the migration window closes.
4. **gRPC service in `akash-network/akash-api`.** Add `LeaseMetricsService` with `QueryRange` and `QueryInstant` RPCs, mirroring the REST surface.
5. **Provider config additions** (`provider-services` config file):
   ```yaml
   metrics:
     enabled: true
     tsdb:
       kind: victoriametrics   # or prometheus
       retention: 720h         # 30 days
       storage_size: 100Gi     # per-provider sizing
     limits:
       per_tenant_qps: 5
       per_tenant_series: 10000
       max_concurrent_queries: 20
     gpu:
       enabled: auto           # auto | true | false
       dcgm_exporter_version: "..."
   ```
6. **Helm chart updates** in `helm-charts` to package the metrics stack as a chart dependency that ships with the provider install, with opinionated defaults.

### Why VictoriaMetrics as the reference TSDB

- Single binary, lower memory footprint than Prometheus at equivalent retention.
- PromQL-compatible, so any future change of backend does not break the API contract.
- Built-in per-tenant rate limiting and series limiting that we need anyway.
- Permissive license (Apache 2.0).

Prometheus is an acceptable substitute for providers that already operate one. Providers MUST NOT ship a TSDB that breaks the PromQL surface used by the gateway proxy.

### Provider sizing guidance

The TSDB sizing is a function of:

```
storage_bytes ≈ retention_seconds × scrape_targets × series_per_target × bytes_per_sample / step_seconds
```

For a provider with 200 active leases, a 15s scrape interval, ~25 series per replica, and 30-day retention, the reference single-node VictoriaMetrics deployment lands in the **30–80 GiB** range for the metrics PVC. Helm defaults SHOULD provision 100 GiB. RAM overhead is dominated by active series count; budget ~2 GiB resident for the reference workload.

These numbers are the provider's cost of joining the metrics ecosystem and SHOULD be factored into provider pricing.

## Client-Side Implementation (Akash Console)

A new Metrics tab on the deployment view in [Akash Console](https://console.akash.network/) is the reference consumer of this API. The implementation:

1. Reads the lease list for the deployment from chain.
2. For each unique provider in the lease list, calls `GET /lease/{dseq}/{gseq}/{oseq}/metrics` on that provider's gateway.
3. Merges the series client-side. Replica-level series stay per-replica; lease-level series are summed.
4. Overlays vertical markers at deployment-update timestamps (sourced from chain events for the deployment) so users can correlate utilization changes with manifest changes — matching Railway's commit-marker UX.
5. Offers Sum vs. Replica toggles for services with `count > 1`.

For tenants using the Provider API directly (no Console), a CLI command is added:

```
provider-services lease-metrics \
    --dseq 1234567 --gseq 1 --oseq 1 \
    --provider akash1... \
    --metric cpu,mem,gpu \
    --since 24h \
    --step 1m \
    --output json
```

## Security Considerations

1. **Tenant isolation.** The single most important property: a tenant must not be able to read another tenant's metrics. Enforcement is at three layers:
   - JWT validation rejects mismatched `iss`/`owner`.
   - The `MetricsClient` interface refuses to issue any query that does not include a complete `(owner, dseq, gseq, oseq)` label filter.
   - The TSDB MAY additionally be configured with VictoriaMetrics' tenant-aware mode (`vmauth`) as defense-in-depth.

2. **Query-cost DoS.** A tenant could craft expensive queries (long range × short step × `all` metrics × `aggregation=replica` on a high-replica service). Mitigation:
   - Per-tenant QPS and concurrent-query caps (config above).
   - Server-enforced maximum series-returned-per-query (default 5000).
   - Server-enforced maximum (range / step) ratio.
   - Optional response streaming for legitimate large queries.

3. **Side-channel leakage.** Aggregated provider-level metrics MUST NOT be exposed via this API. A tenant should not be able to infer a competing tenant's footprint by querying provider-wide series. Only label-filtered, lease-scoped data is returned.

4. **JWT replay.** Already mitigated by AEP-64's short token lifetime (≤15 min) and `jti` nonce.

5. **Metric authenticity.** Providers self-report their tenants' utilization. There is no on-chain attestation in v1. This is acceptable because (a) the metrics are not used for settlement, (b) a misbehaving provider is detectable through cross-correlation if a tenant runs the same workload on multiple providers, and (c) future AEPs on confidential computing and TEE attestation could extend this with signed metric attestations. Listed as a future consideration.

6. **GPU metric privacy.** `gpu_model` is exposed to the tenant, which is informational and acceptable. Hostnames, kernel versions, and other provider-internal identifiers MUST NOT be exposed.

## Operational Considerations

### Cardinality

The label set above produces, per replica:

```
series_per_replica ≈ |metrics_catalog| × |gpu_index per replica (usually 0 or 1)|
                  ≈ 15–25 series for CPU/RAM/disk/net workloads
                  ≈ 25–35 series for GPU workloads
```

With sensible per-tenant limits (10k active series) tenants can run hundreds of replicas without hitting caps. Provider-global cardinality is the more important concern; the reference Helm chart sets a global cap that providers can lift.

### Failure modes

- **TSDB pod restart.** Causes a gap in data, surfaced in the response as a `warnings` entry. The provider gateway MUST return 503 only when the TSDB is unreachable, not when individual series are missing.
- **dcgm-exporter crash.** GPU metrics gap; CPU/memory unaffected.
- **High scrape lag.** Reflected by drift between `t` values and "wall clock". Providers SHOULD expose internal scrape-lag metrics on their own operational dashboard but MUST NOT expose them via the tenant API.

### Resource overhead

The reference stack (single-node VictoriaMetrics + exporters) is sized to fit comfortably on a single 4-vCPU / 8 GiB node. This is intentionally modest so small providers can participate without a dedicated metrics node pool.

## SDL Compatibility

This AEP does not require an SDL change. Tenants do not opt-in to having metrics collected — collection happens unconditionally for any lease the provider hosts, because there is no way to know in advance whether a tenant will want metrics, and collection is cheap.

A future SDL extension MAY add:

```yaml
profiles:
  compute:
    web:
      metrics:
        retention: 90d        # request higher retention if provider supports it
        scrape_interval: 5s   # request finer resolution
```

…but this is explicitly deferred. v1 ships with provider-side defaults only.

## Backward Compatibility

This AEP is purely additive. It does not change any existing on-chain message, SDL field, or Provider API endpoint. Providers running pre-metrics versions remain valid network participants; they simply advertise `features.metrics.enabled = false` in their status response and Console hides the tab for those providers.

The new JWT scope `metrics` is additive in AEP-64's enum; old tokens that lack it cannot read metrics but continue to function for every other endpoint they were issued for.

## Migration Guide

### For providers

1. Upgrade `provider-services` to the version that includes this AEP's changes.
2. Pull the updated Helm chart; the `metrics-operator`, exporters, and TSDB will be installed into the provider's control namespace on the next reconcile.
3. Size the metrics PVC (default 100 GiB) and verify GPU nodes have `dcgm-exporter` running.
4. Confirm the `/status` endpoint now advertises `features.metrics.enabled = true`.
5. Optionally tune per-tenant limits and retention in `provider-services` config.

### For tenants

1. Update client (Console or CLI) to a version that knows the `/metrics` endpoint exists.
2. Issue a JWT (per AEP-64) with the `metrics` scope.
3. Query.

No SDL changes, no redeploy needed.

## Future Considerations

1. **Attested metrics.** Pair this with [AEP-65](../aep-65) (Confidential Computing) or a future TEE-attestation AEP to produce signed metric summaries that can serve as inputs to on-chain provider scoring.
2. **Long-term retention via downsampling.** Providers MAY offer 1y+ retention at hourly resolution by adding a Thanos- or VictoriaMetrics-cluster-style downsampler.
3. **Multi-tenant query API for chain-level analytics.** A network-wide aggregator (operated by a neutral party) could anonymize and aggregate provider metrics to publish utilization heatmaps — useful for the network statistics already published on stats.akash.network. This requires explicit provider opt-in and tenant anonymization.
4. **OTLP/remote-write export.** Allow tenants to specify a remote-write URL in their SDL so their provider's metrics pipeline mirrors data to their own Grafana Cloud / Datadog / VictoriaMetrics Cloud account. Treated as a future AEP because it interacts with billing and egress.
5. **Workload-driven autoscaling.** Once stable, this metric surface becomes the input to a future AEP on horizontal/vertical autoscaling at the lease level.
6. **AMD and Intel GPU support.** When AMD/Intel GPU exporters reach the maturity of `dcgm-exporter`, extend the GPU section of the metrics catalog.

## Open Questions

1. **Should `format=prometheus` accept arbitrary PromQL,** restricted to a label-filtered subset? It would let advanced tenants use Grafana directly against a provider, but it widens the attack surface. The current proposal says no for v1; revisit after the basic API is in production.
2. **How is the metrics PVC accounted for in provider economics?** It is a fixed cost per provider, not a per-lease cost. Should it be implicit (provider absorbs it), opt-in (provider charges a small surcharge per lease), or community-subsidized (similar to the rationale behind the provider-incentive pool)? Recommend leaving to provider operators in v1.
3. **Should retention beyond 30 days be standardized,** or left as a provider-by-provider differentiator? The proposal makes 30 days a *floor*, not a ceiling. Providers MAY advertise longer retention via the `features.metrics.retention_days` field.
4. **Egress/ingress separation granularity.** Splitting public vs. private network bytes is technically possible with eBPF (Cilium/Hubble) but adds operator complexity. v1 says SHOULD-where-cheap; revisit if there is tenant demand.

## Reference Implementation Sketch

A reference proof-of-concept is intended to be built in two stages:

- **Stage 1 (provider-side, behind a feature flag).** `operator/metrics` package, `MetricsClient` interface, REST handler, gRPC service, Helm chart. Disabled by default; opt-in by provider operators for testing.
- **Stage 2 (client-side + flag flipped on).** Console Metrics tab, CLI command, default flag flip to enabled after a 4-week soak with opted-in providers.

Tracking issues will be filed against:

- `akash-network/provider` — provider daemon changes
- `akash-network/akash-api` — gRPC and JWT scope additions
- `akash-network/console` — Console UI
- `akash-network/helm-charts` — provider install chart updates
- `akash-network/AEP` — this document

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
