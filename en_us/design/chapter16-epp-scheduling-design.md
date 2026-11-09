# Chapter 16 EPP Scheduling Design

## Chapter Goals

Through this chapter, the reader will understand:

- Why EPP intelligent scheduling is needed: the limitations of BFE's local WRR (Weighted Round Robin) in LLM inference scenarios;
- The technical origins of EPP: the CNCF llm-d project, and the relationship between ai-gateway-epp and the upstream EPP;
- Where EPP (Endpoint Picker) sits in the overall architecture of Rainway AI Gateway, and how it divides responsibilities with traditional load balancing;
- How the Control Plane (AI Gateway API) manages the EPP instance pool, compiles scheduling configuration, assigns instance groups, and distributes configuration in both directions;
- How the Data Plane (BFE) connects to EPP via the gRPC ext_proc protocol, and the fallback semantics upon failure;
- Configuration consumption, the scheduling plugin chain, and affinity mechanisms inside the ai-gateway-epp component;
- The end-to-end flow of the EPP scheduling path and its boundary semantics.

---

## Background: Limitations of WRR and the Positioning of EPP

A Cluster in Rainway AI Gateway uses BFE's built-in WRR (Weighted Round Robin) for backend selection by default. WRR is simple and deterministic, but it knows nothing about backend state, and has three obvious limitations in LLM (Large Language Model) inference scenarios:

1. **No load awareness**: WRR distributes requests purely by weight in rotation and cannot perceive the real-time pressure of inference backends. The queue length and KV cache utilization of GPU inference services fluctuate sharply with requests; round robin keeps sending requests to already saturated backends, causing queuing delay to spike.
2. **No prefix cache affinity**: in LLM inference, requests sharing the same prompt prefix can directly reuse the cached KV cache if they land on the same backend, significantly reducing time to first token. WRR's round robin naturally scatters requests with the same prefix, wasting this cache benefit.
3. **No session affinity**: multi-turn conversations (multiple requests of the same session) gain even more from reusing KV cache on a backend. WRR cannot converge requests of the same session onto the same backend.

The component that takes on intelligent scheduling is EPP (Endpoint Picker), ai-gateway-epp. It runs in a "multi-Cluster, single process" form: before forwarding a request, BFE hands it to EPP via the gRPC ext_proc (External Processing) protocol; EPP selects the optimal backend based on signals such as backend utilization, queue length, prefix cache, and session binding; BFE then forwards according to the decision. For the basic principles of routing and scheduling, see [Chapter 4 Routing and Scheduling Principles of the AI Gateway](../principle/chapter04-routing-and-scheduling.md).

### Technical Origins of EPP: The CNCF llm-d Project

The concept and scheduling engine of EPP come from [llm-d](https://llm-d.ai/): a Kubernetes-native distributed LLM inference framework, jointly donated to the Cloud Native Computing Foundation (CNCF) by Google Cloud, Red Hat, IBM Research, CoreWeave, NVIDIA, and others in March 2026, and now a CNCF Sandbox project. llm-d builds on vLLM and the Kubernetes Gateway API Inference Extension (GAIE), in which the Endpoint Picker (EPP) is the reference scheduling implementation of that inference extension: before forwarding a request, the gateway calls back to EPP via the Envoy ext_proc protocol, and EPP completes backend selection with a Filter → Score → Pick plugin pipeline. EPP's scheduling model is designed specifically for inference workloads — it uses KV cache utilization and request queue length as decision signals, and ships inference-specific scheduling optimizations such as prefix cache awareness and session affinity.

The upstream EPP targets Kubernetes environments: the list of inference backend instances comes from the InferencePool CRD (Custom Resource Definition), and scheduling policies and instance roles are determined by control components on the Kubernetes side.

### The Relationship Between ai-gateway-epp and the Upstream EPP

ai-gateway-epp reuses the scheduling engine and plugin system of llm-d (the llm-d-router engine package), recombined into a multi-Cluster scheduling process adapted to the Rainway AI Gateway Control Plane. The two share the ext_proc protocol and the Filter → Score → Pick plugin pipeline, and the inference scheduling semantics of KV cache / prefix cache / session affinity are identical; the scheduling configuration consumed by ai-gateway-epp (the compiled artifact EndpointPickerConfig) is exactly the upstream EPP configuration format, so llm-d's scheduling capabilities and plugin ecosystem are inherited smoothly.

The differences between the two concentrate on orchestration environment and lifecycle management:

| Dimension | Upstream EPP (llm-d) | ai-gateway-epp |
|-----------|----------------------|----------------|
| Runtime environment | Kubernetes: backend instances come from the InferencePool CRD, scheduling policies come from the inference extension configuration | No Kubernetes dependency: backend instances come from the cluster_table export of the AI Gateway API InnerAPI, scheduling configuration and assignments come from the `epp_data/config` interface (epp_config compiled artifact + assignment full view) — there is no CRD reconciler; for local debugging/testing, `-local-config-dir` can replace the InnerAPI configuration source with local JSON files so the process runs standalone without the Control Plane |
| Process model | One EPP process serves one inference pool (InferencePool) | One process serves multiple Clusters: each assigned Cluster maps to one Cell (resident in the data plane: datastore + metrics collection; the engine switches atomically on configuration version), ext_proc requests are routed to the corresponding Cell by the inference-pool metadata injected by BFE |
| Configuration activation | CRD updates take effect gradually through Kubernetes reconcile | `epp_config` changes are compiled into a new engine and switched atomically; the old engine drains (flow-control queue eviction + in-flight request wait cap); a compile failure of a single Cluster does not affect other Clusters |
| Assignment granularity | Determined by components on the Kubernetes side | Determined by the assignment full view: assignments are per instance group (cluster→group), and each instance self-matches by its own group identity in the view to determine which Clusters' scheduling rights its group holds |

EPP scheduling is a first-class, Cluster-level configuration: when a Cluster's `balance_mode` field is `EPP`, backend selection of that Cluster is taken over by EPP; when it is `WRR` (the default), BFE's local weighted round robin handles it. The two modes can be switched during the Cluster lifecycle.

---

## Overall Architecture

EPP intelligent scheduling involves four components: the Control Plane AI Gateway API, the Data Plane BFE, the scheduler ai-gateway-epp, and the configuration agent Conf Agent. The overall architecture is as follows:

```mermaid
flowchart LR
    C[Client] -->|HTTP| B[BFE Data Plane]
    B -->|BalanceMode=EPP<br/>gRPC ext_proc| E0[ai-gateway-epp<br/>instance group g1]
    B -->|EPP failure fallback<br/>local WRR| R[Inference backend instances]
    E0 -->|Scheduling decision + metrics scrape| R

    subgraph Control Plane
        A[AI Gateway API]
    end
    E0 -->|InnerAPI polling<br/>epp_data / cluster_table| A
    A -->|server_data_conf with EPPAddr| CA[Conf Agent]
    CA -->|Hot reload| B
    A -->|cluster_table| CT[(cluster_table export)]
    E0 -.->|cluster-table-discovery| CT
```

Responsibilities of each component:

- **AI Gateway API**: maintains the EPP instance pool (`/epp-pool`), each Cluster's `balance_mode` and `epp_config` / `epp_runtime`, and the Cluster→instance-group assignments (`epp_assignments`); distributes the compiled scheduling configuration and the assignment view to EPP via InnerAPI, and distributes `GslbBasic` with `BalanceMode=EPP` and `EPPAddr` (a single-element list: the address of the assigned instance group's instance) to BFE via server_data_conf.
- **BFE**: requests hitting an EPP-mode Cluster are sent via gRPC ext_proc to the currently active EPP address (the sole instance of the instance group); if the EPP call fails or there is no candidate, it silently falls back to local WRR without interrupting business.
- **ai-gateway-epp (EPP component)**: runs with the instance group (exactly 1 instance per group) as the deployment unit; polls InnerAPI for the scheduling configuration and the assignment view, and decides which Clusters' scheduling rights this instance's group holds according to the assignment (cluster→group); discovers inference backends via the `cluster-table-discovery` plugin from the cluster_table export, and periodically scrapes backend `/metrics` metrics to drive scheduling. For debugging/testing, `-local-config-dir` can replace the InnerAPI configuration source with local JSON files so the process runs standalone without the Control Plane.
- **Conf Agent**: delivers server_data_conf (including `GslbBasic`) to BFE and triggers hot reload; the mechanism is consistent with [Chapter 14 Config Export and Version Control Design](./chapter14-config-export-and-version-control.md).

---

## Control Plane Design

The Control Plane has three core problems: how the EPP instance pool is managed, how each Cluster's scheduling configuration is expressed and distributed, and how Clusters are assigned to EPP instance groups. This section explains them in turn.

### balance_mode and epp_config: The Cluster-Level Scheduling Entry

The Cluster resource carries two EPP-related fields:

| Field | Type | Semantics |
|-------|------|-----------|
| `balance_mode` | string | Cluster balancing mode: `WRR` (default, BFE local weighted round robin) or `EPP` (the EPP scheduler takes over backend selection). This field is the sole source of truth for EPP mode |
| `epp_config` | object | EPP scheduling configuration in a simplified user-facing form (see below). Required and effective when `balance_mode=EPP`; optional when `balance_mode=WRR` — if provided it is retained and format-validated, but not compiled and not exported |
| `epp_runtime` | object | EPP runtime parameters (health check and call timeouts), see below. Layered separately from `epp_config` (scheduling configuration): the former is compiled into BFE `GslbBasic`, the latter into EPP's `EndpointPickerConfig`. Optional, nullable |

`epp_config` does not expose the plugin declaration details of the llm-d `EndpointPickerConfig` directly; instead it expresses intent as "load profile + a few first-class tuning parameters". The parameters fall into four groups — load profile, admission watermarks, bad-backend eviction, and capacity queuing — and affinity strength separately controls the weights of the affinity scorers:

| Field | Default | Description |
|-------|---------|-------------|
| `load_profile` | `balanced` | Load profile: `queue-first` (queue first, queue scorer weight highest) / `balanced` / `kv-first` (KV cache first, kv scorer weight highest) |
| `affinity` | `medium` | Affinity strength: `off` / `low` / `medium` / `high`, mapped to affinity scorer weights 0 / 0.3 / 0.6 / 1.0; when `off`, prefix-cache-scorer and session-affinity-scorer are not injected (even if the corresponding feature switches are on) |
| `prefix_cache_affinity` | `true` | Prefix cache affinity switch (soft affinity) |
| `session_affinity_enabled` | `false` | Session affinity switch |
| `session_affinity_header` | - | Request header carrying the session id; required when `enabled=true`, the two appear in pairs |
| `kv_cache_utilization_max` | `0.9` | Admission watermark, endpoint filter threshold `(0,1]`: endpoints whose KV cache utilization exceeds this value are filtered out |
| `waiting_queue_max` | `0` (disabled) | Admission watermark, waiting queue length threshold `≥0`: effective when greater than 0, utilization-filter appends a waiting-queue filter condition |
| `running_requests_max` | `0` (disabled) | Admission watermark, running requests threshold `≥0`: effective when greater than 0, utilization-filter appends a running-requests filter condition |
| `fallback_on_empty` | `false` | Admission watermark supplement: whether to fall back and admit when all endpoints are filtered; passed through to utilization-filter's fallbackOnEmpty |
| `metrics_staleness_threshold_ms` | `200` | Bad-backend eviction threshold (milliseconds, `>0`): a backend whose metric scraping fails consecutively and whose data is stale beyond this value is evicted as a bad backend (about 4 consecutive metric scrape failures) |
| `flow_control` | - | Capacity queuing parameters: `max_requests` (unlimited by default, `-1` for explicitly unlimited), `queue_ttl` (seconds), `no_endpoint_queue_ttl` (seconds, follows `queue_ttl` by default), `enable_eviction` (only `false` accepted); the compiled artifact always carries an explicit priority 0 band, so the global `max_requests` is not silently truncated by the hidden band defaults (5000 concurrency / 1GB memory) |

`epp_config` is parsed with strict field parsing: a request body carrying undefined fields is rejected with 422 directly.

Value rules of the two fields:

- `balance_mode=EPP`: `epp_config` is required and must pass field validation (a create request without it returns 422, eliminating the dangling state of "an EPP Cluster without scheduling configuration").
- `WRR → EPP` switch: a valid `epp_config` must be provided at the same time.
- `EPP → WRR` switch: only `balance_mode` needs to be modified; `epp_config` and assignment records are retained (dormant: not compiled, not exported), and the original configuration and assignments continue to take effect when switching back to `EPP`.
- A non-empty `epp_config` must pass field validation regardless of `balance_mode` — dormant retained configurations remain valid as well.

In storage, the `clusters` table contains a `balance_mode` column (default `'WRR'`) and an `epp_config` column (JSON, nullable). The system preserves the raw JSON written by the user; fields not explicitly carried are not persisted; GET reads back exactly what was written; defaults only materialize at export compile time.

### epp_runtime: Cluster-Level EPP Runtime Parameters

`epp_runtime` expresses the runtime parameters of BFE's EPP call path, layered separately from `epp_config` (scheduling configuration), which targets the EPP scheduling engine. The parameters fall into two groups — health check and call timeouts — exported as `EPPCheck` and `EPPTimeout` of `GslbBasic` respectively:

| Field | Default | Description |
|-------|---------|-------------|
| `check.disabled` | `false` | Whether to disable the background health check of EPP addresses |
| `check.check_interval` | `"2s"` | Health check interval |
| `check.fail_threshold` | `3` | Consecutive failure threshold of the active address before it is considered unavailable |
| `check.cooldown` | `"45s"` | Cooldown period, preventing state flapping |
| `check.success_threshold` | `2` | Consecutive successes required to re-enable an address after the cooldown |
| `timeout.connect` | `"500ms"` | EPP gRPC connection timeout |
| `timeout.call` | `"3s"` | EPP call timeout |

Value rules:

- Strict field parsing: undefined fields such as `tls` / `breaker` are rejected with 422 directly. TLS (EPPTLS) and circuit breaking (EPPBreaker) are not open in this release — export fixes `EPPTLS{Plaintext:true}` and `EPPBreaker{Disabled:true}`.
- Duration fields must be positive; sub-fields not explicitly carried are filled with defaults and persisted at write time, and GET reads back what was persisted.
- Only when `BalanceMode=EPP` and the assignment is valid are they compiled into `GslbBasic.EPPCheck` / `EPPTimeout` and exported; dormant or unassigned Clusters are not exported.
- In storage, the `clusters` table contains an `epp_runtime` column (JSON text, nullable), preserving the raw JSON written by the user.

### /epp-pool: The EPP Instance Pool

The EPP instance pool is a singleton resource: `GET` for detail + `PATCH` for full replacement. The pool consists of instance groups (groups) and instance lists:

```json
{
    "name": "EPP.pool",
    "groups": [
        {
            "name": "g1",
            "instances": [
                { "id": "epp-a", "host": "10.0.0.1", "port": 9002 }
            ]
        }
    ]
}
```

Design points:

- **Instance id and group identity convention**: EPP is deployed as a StatefulSet, so the instance id is the Pod hostname; when registering in `/epp-pool`, the instance id is the Pod name (non-K8s deployments register the deployment facts). The EPP process itself self-matches by its **group identity** (`-group-name` / `AI_GATEWAY_EPP_GROUP_NAME`, taken from the group name registered at deployment time) in the assignment view, instead of matching roles by instance id.
- **Static configuration mode (no registration/heartbeat)**: the instance list is a deployment fact, maintained by the deployment pipeline calling `PATCH` after instance changes (scaling, machine replacement); it is naturally idempotent. The system provides no registration/heartbeat interfaces, and the AI Gateway API performs no liveness marking — instance liveness is driven by BFE-side `EPPAddr` connection hysteresis.
- **Group size**: exactly 1 instance per group — the `instances` field keeps its array form (reserved for future multi-instance scaling), but an empty group or a group with ≥2 instances is rejected with 422 (`requires exactly 1 instance`).
- **Validation**: group names are non-empty and unique within the pool; instance ids are globally unique within the pool; `(host, port)` combinations are globally unique within the pool; host is a Hostname or IP (IPv6 literals without brackets).
- Instance pool changes do not directly bump the `ConfigTopicEppData` topic (instance additions/removals do not change the cluster→group mapping).

Instances are expanded and stored in the `epp_instances` table (id, host, port, group_name); when generating `EPPAddr`, they are joined as `host:port` via `net.JoinHostPort` (IPv6 gets brackets automatically).

### /epp-assignments: Assignment Model and Assigner

The association between Clusters and EPP instance groups is stored in the `epp_assignments` table:

```
cluster (unique key), group_name
```

Assignments are **per instance group**: each record stores only the cluster→group mapping, and the instances of a group are derived from `/epp-pool` by group name. The management plane `GET /epp-assignments` returns the full view: per-cluster elements keep only `{cluster, group}`, together with aggregated `unassigned_clusters` (EPP-mode Clusters without a valid assignment, including dangling assignments whose group has been removed from the pool) and `idle_groups` (groups bearing no assignment).

Assignments are **generated automatically by the assigner**; the manual override `PUT /epp-assignments/{cluster}` serves only as an operations intervention entry, and its request body carries only `{"group_name"}`. Automatic assignment uses a greedy + deterministic tie-break algorithm:

```
Input: instance pool (group → instance list), existing assignments (epp_assignments), target cluster
1. Candidate group filtering: the group exists in the pool and is non-empty
2. Group selection: group load = number of clusters the group currently bears
        → pick the group with the smallest load; ties break by smallest group name (inter-group balance)
3. Write epp_assignments (upsert on the cluster unique key)
```

The algorithm is fully deterministic with no randomness; results are replayable and unit-testable. Assignments are per-Cluster, with no one-to-one group↔Cluster constraint: one instance group may bear multiple Clusters, and one Cluster belongs to exactly one group.

Triggers and repair paths of automatic assignment:

| Trigger | Behavior |
|---------|----------|
| A cluster is created in EPP mode, or a `WRR → EPP` change | No assignment record → automatic assignment |
| After a `/epp-pool` PATCH leaves an assignment dangling (the instance group has been removed from the pool) | Reassign across groups (rerun the greedy algorithm over the whole pool); no assignable candidate group in the pool → clear the assignment and enter the unassigned state |
| Periodic reconciliation reconciler (30 seconds, configurable) | Scans all EPP Clusters; automatically fills any without a valid assignment (idempotent, zero writes in most rounds) |
| `PUT /epp-assignments/{cluster}` | Manual override of `{"group_name"}`, triggers a server_data_conf version bump |

### epp_data Unified Distribution: Compiled Configuration + Assignment Full View

EPP instances pull all Control Plane configuration through a single InnerAPI endpoint:

```http
GET /inner-api/v1/configs/epp_data/config?version=<last version number>
Authorization: Token <token>
```

The response `Config` contains two sections: `epp_config` is `map[cluster name]EndpointPickerConfig` (deterministically compiled from the simplified `epp_config` at export time), and `assignment` is `map[cluster name]{group}` (the full cluster→group mapping). Core semantics:

- **Version increment**: goes through the existing `ExportConfig` framework (topic `ConfigTopicEppData`, MD5 signature comparison); returns `Data: null` when nothing changed.
- **All EPP instances receive exactly the same assignment content** (same version snapshot); each instance matches per Cluster by its own group identity: if the entry's group equals the instance's own group → bear the scheduling right of that Cluster; otherwise skip that Cluster. The server therefore needs no instance identity validation, and there are no edge-case errors of per-instance views. The group identity is declared by an EPP-side parameter (the InnerAPI contract writes it as `-group-name` / environment variable `AI_GATEWAY_EPP_GROUP_NAME`; some design documents also write it as epp-group / AI_GATEWAY_EPP_GROUP, and the ai-gateway-epp implementation is authoritative), which must match a group name registered in the instance pool. This parameter goes together with the group-form assignment view, is incompatible with the instance-level primary/standby matching of the older form, and requires ai-gateway-epp, the Control Plane, the schema upgrade and the front end to be released in the same version.
- **Single-endpoint merged distribution**: Cluster creation, mode changes, and assignment changes often change both sections at once; a single topic naturally guarantees both sections share the same version as an atomic snapshot, eliminating the role/configuration mismatch window caused by cross-topic version skew; the data volume is tiny, so full redistribution costs are negligible.
- **Unassigned abnormal state**: a Cluster present in `epp_config` but absent from `assignment` is unassigned; EPP raises a local alert and does not build a scheduling engine for that Cluster.

The compilation rules from the simplified configuration to `EndpointPickerConfig` are fixed in the AI Gateway API code template and covered by unit tests:

| Simplified field | Compiled artifact |
|------------------|-------------------|
| Fixed part | Injects the `cluster-table-discovery` endpoint discovery plugin (clusterName taken from this Cluster), two scorers `kv-cache-utilization-scorer` + `queue-scorer`, the `max-score-picker` picker, and `openai-parser` |
| `load_profile` → scorer weights (kv, queue) | `queue-first` → (0.2, 1.0); `balanced` → (0.6, 0.6); `kv-first` → (1.0, 0.2) |
| `affinity` → affinity scorer weights | `off` → not injected; `low` → 0.3; `medium` → 0.6; `high` → 1.0 (prefix-cache-scorer and session-affinity-scorer share the same weight) |
| utilization-filter conditions | Always contains `kv-cache-utilization` (threshold from `kv_cache_utilization_max`); `waiting-queue` / `running-requests` are appended only when the corresponding threshold is greater than 0; `fallback_on_empty` is passed through as fallbackOnEmpty |
| Bad-backend eviction | `saturation-detector` (utilization-detector type) is always injected: `kvCacheUtilThreshold` shares the same source as the filter's KV threshold, `queueDepthThreshold` takes `waiting_queue_max` (5 when not enabled), `metricsStalenessThreshold` takes `metrics_staleness_threshold_ms`, `stalenessPolicy=ignore`; this plugin does not enter the schedulingProfiles scoring chain and is referenced jointly by utilization-filter and flow-control backpressure |
| `prefix_cache_affinity=true` | A `prefix-cache-scorer` is appended to the scorer chain (weight taken from the `affinity` mapping) |
| `session_affinity_enabled=true` | A `session-affinity-scorer` is appended to the scorer chain (strategy=session_id, session id taken from `session_affinity_header`, weight taken from the `affinity` mapping) |
| `flow_control` | The `flowControl` section is always emitted (including the `saturationDetector.pluginRef` reference and the priority 0 band, seconds converted to Go duration); `flowControl` is appended to `featureGates` only when `flow_control` is configured; when `max_requests` is unset or `-1`, the global `maxRequests` field is not generated, and `enable_eviction` is not emitted |

A key note on flow-control compilation: all EPP requests run in the priority 0 band, so the compiled artifact always carries an explicit band 0. llm-d falls back to hidden defaults for unconfigured bands (5000 concurrency / 1GB memory), while the global limit and band limits are enforced independently (hasCapacity checks both), so omitting the band would silently truncate a global `max_requests` above 5000. When `max_requests` is set explicitly, band 0 mirrors it; when it is unset or `-1`, the global `maxRequests` field is not generated, but band 0 is still emitted explicitly (10000 concurrency, 5Gi memory), keeping the capacity deterministic and auditable.

The weights of affinity scorers are uniformly mapped from the `affinity` strength and are orthogonal to `load_profile`: `load_profile` only tunes the weights of the two base scorers (kv, queue); when `affinity=off`, neither affinity scorer is injected even if `prefix_cache_affinity` / `session_affinity_enabled` are on.

### cluster_table Export: The Source for EPP to Discover Inference Backends

An EPP-mode Cluster's instance pool is marked with type `Role=EPP`, but the pool instance list still syncs the Provider's `instance_pool` as usual (the `ProviderInstancePoolSyncer` syncs it to all Clusters referencing that Provider when the Provider instance pool changes, including EPP pools). These instances serve two consumers:

1. **cluster_table export** (`/configs/gslb_data/cluster_table`): EPP's `cluster-table-discovery` plugin discovers inference backends through it; an instance `Weight=0` means the instance is drained.
2. **BFE fallback WRR**: the backend list used by BFE's local WRR fallback when EPP is entirely down or has no candidates.

Under `BalanceMode=EPP`, BFE performs no local WRR and the instance list does not participate in normal balancing; it serves only the two purposes above.

### server_data_conf Export: EPPAddr and Degradation Semantics

An EPP-mode Cluster exports `GslbBasic` in server_data_conf:

```json
"GslbBasic": {
    "BalanceMode": "EPP",
    "EPPAddr": ["10.0.0.1:9002"]
}
```

- `EPPAddr` is a **single-element list**: the address of the assigned instance group's instance (exactly 1 instance per group). Addresses are joined from the host/port of `epp_instances` via `net.JoinHostPort`; the `[]string` type is retained for future multi-address extension.
- **Degraded export when there is no valid assignment**: the Cluster's `BalanceMode` is set to `WRR` and no `EPPAddr` is generated; meanwhile AI Gateway API outputs an error-level log (including the Cluster name and reason). This is explicit degradation rather than a silent error: degrading a single Cluster does not block the whole server_data_conf distribution; during degradation BFE schedules the Provider instances in that Cluster's pool with local WRR (requests remain servable, only losing EPP intelligent scheduling); after the assignment recovers, the next export automatically returns to `EPP`.

### Design Trade-off: Why Not Expose EndpointPickerConfig Directly

`EndpointPickerConfig` is an internal implementation abstraction of llm-d, containing plugin instances, the pluginRef reference graph, DAG layer ordering, and Quantity formats. Exposing it directly to users costs: users need to understand the whole plugin system to assemble a correct configuration, while AI Gateway API can only perform JSON Schema structural validation — reference errors surface only at EPP compile time.

With the simplified "load profile + tuning parameters" form: the user's expression cost drops from "assembling a plugin chain" to "pick a load profile, set an affinity strength, and fill in a few numbers"; the compiled artifact is deterministically generated from a code template, with reference integrity and DAG acyclicity guaranteed by construction, so the configuration is valid out of the box; the EPP-side consumption format stays unchanged. There is no advanced pass-through mode; if a need for custom plugin chains emerges in the future, it can be opened up then.

---

## Data Plane Design: BFE's ext_proc Integration

### The EPP Fields of GslbBasic

Each Cluster's `GslbBasic` section in BFE's `cluster_conf.data` carries EPP-related fields (all optional, with defaults):

| Field | Description |
|-------|-------------|
| `BalanceMode` | `"EPP"` means this Cluster goes through ext_proc scheduling; defaults to WRR, in which case EPP fields are ignored |
| `EPPAddr` | Instance group address list, always a single element (the `host:port` of the group's sole instance); non-empty when `BalanceMode=EPP` (load validation: elements must be `host:port`, deduplicated within the list) |
| `EPPCheck` | Health check and hysteresis parameters: `Disabled` (default false), `CheckInterval` (default 2s), `FailThreshold` (default 3), `Cooldown` (default 45s), `SuccessThreshold` (default 2) |
| `EPPTimeout` | Call timeouts: `Connect` (default 500ms), `Call` (default 3s) |
| `EPPTLS` | Transport security, three mutually exclusive choices (fail-fast mutual-exclusion validation at load time): `Plaintext` (plaintext dial, the default) / `Insecure` (TLS with certificate verification skipped, test environments only) / `CAFile` (TLS with verification of the EPP server certificate, recommended for production). When unconfigured, it is auto-filled with `{Plaintext: true}` (plaintext dial) plus a warning log; explicit `{"Insecure":true}` or `{"CAFile":...}` still uses TLS |
| `EPPBreaker` | Circuit breaker parameters (sliding-window error rate), complementary to address-level failover |

### Request Processing Flow

Requests hitting an EPP-mode Cluster are processed as follows:

1. BFE constructs the first ext_proc message (RequestHeaders), injecting `"llm-d.ai": {"inference-pool": "<cluster name>"}` into the metadata; EPP's demux routes the stream to the scheduling Cell of the corresponding Cluster by pool name.
2. BFE sends the request via gRPC ext_proc to the currently active EPP address (`EPPAddr[0]`, the sole instance of the group), and waits for EPP's decision.
3. EPP returns the selected endpoint address in `dynamic_metadata` of the response, at `envoy.lb → x-gateway-destination-endpoint`.
4. BFE constructs a temporary backend in the local cluster_table according to the decided address and forwards the request; on the response path, it relays the streaming response back to EPP via ResponseHeaders and the body filter (for purposes such as token metering).

### Failover and Fallback Semantics

- **Hysteresis health check (EPPCheck)**: each EPP address has a background health check (gRPC health). If the active address fails consecutively `FailThreshold` times, it is considered unavailable; it is not re-enabled during the `Cooldown` period; after the cooldown, the address must pass the health check consecutively `SuccessThreshold` times before being re-enabled. Since `EPPAddr` is currently a single element, requests go straight to the local fallback while the address is unavailable.
- **Error-driven immediate retry**: for `Unavailable / cell is not serving` and `unknown inference pool` errors, BFE does not wait for the health check cycle; it retries the request directly on the next EPP address of the same Cluster (the list form of `EPPAddr` is reserved for this extension).
- **Silent fallback to local WRR**: when the EPP call fails (transport error, timeout, address unavailable) or EPP has no candidate (fail-closed), BFE falls back to the local `Balance()` using the Cluster RS (i.e. Provider instances) to continue serving; business is uninterrupted, only losing intelligent scheduling. The fallback is silent and normal traffic is undisturbed.
- **Failure recovery path**: while an instance is down, local WRR carries the traffic; recovery relies on the Control Plane reassigning the Cluster to a healthy instance group — the periodic reconciliation reconciler repairs automatically, or a manual `PUT /epp-assignments/{cluster}` override is used, and BFE switches to the new instance group address at its next server_data_conf pull.
- **Observability**: BFE's monitoring port provides `/monitor/epp_metrics` (Prometheus text format); the core metrics are `epp_calls_total{cluster,result}` (result ∈ ok/no_pool/unknown_pool/draining/transport) and `epp_fallback_local_total{cluster}`, plus failover/failback counters and the active address index.

---

## Design on the ai-gateway-epp Side

### Dual Poller and Cell Engines

The ai-gateway-epp process runs two configuration pollers, both based on the unified `Source[T]` incremental polling framework (version negotiation + failure backoff + fail-static retention of locally known configuration):

- **epp_data poller**: polls `/configs/epp_data/config` for the compiled scheduling configuration and the assignment full view. When configuration changes, it recompiles the scheduling engine per Cluster and swaps it atomically (the old engine drains and is destroyed, waiting 60 seconds by default); when the assignment changes, it drives the Cell lifecycle (Ensure/Promote/Demote/Drop) by diffing against local roles.
- **cluster_table discovery poller**: polls `/configs/gslb_data/cluster_table`, diffing the backend instance list into each Cluster's `cluster-table-discovery` plugin; an instance `Weight=0` is treated as drained (the instance is skipped, and instances already gone from the list are removed accordingly).

The assignment determines which Clusters this instance's group holds: each assigned Cluster gets one Cell that loads configuration, compiles the engine, and serves scheduling; Clusters the group does not bear get no Cell. The primary/standby role concept is removed (the EPP-side `cell.Role` goes away as well, released in the same version as the Control Plane) — with exactly 1 instance per group, an assignment change simply means Ensure/Drop of Cells.

### The Scheduling Plugin Chain

The compiled `EndpointPickerConfig` defines each Cluster's scheduling pipeline. Plugins injected according to the parameter combination include:

- **cluster-table-discovery**: endpoint discovery, whose data source is the cluster_table export;
- **utilization-filter**: filters out endpoints whose KV cache utilization exceeds `kv_cache_utilization_max` (fail-closed: no candidate when all endpoints are filtered); when `waiting_queue_max` / `running_requests_max` are enabled, the corresponding filter conditions are appended; when backend metrics are missing, the corresponding scores are neutralized (kv score treated as 1.0, the filter not applied) and scheduling degrades gracefully;
- **saturation-detector** (utilization-detector type): the always-injected saturation detection plugin; the KV cache threshold shares the same source as utilization-filter, and metric staleness determination takes the bad-backend eviction threshold; it does not enter the schedulingProfiles scoring chain and is referenced jointly by utilization-filter and flow-control backpressure;
- **kv-cache-utilization-scorer / queue-scorer**: base scoring items, with weights determined by `load_profile`;
- **prefix-cache-scorer**: prefix cache affinity (soft affinity). The index is a per-endpoint LRU in EPP's local memory (written autonomously by the `approx-prefix-cache` producer), requiring no external storage such as Redis;
- **session-affinity-filter / session-affinity-scorer**: session affinity. Binding state is likewise in EPP's local memory; the session id is parsed from the request header specified by `session_affinity_header`;
- **max-score-picker**: takes the endpoint with the highest total weighted score; ties yield deterministically in rotation.

The scheduling semantics are "weighted sum + highest score wins", and filters run before scoring. Therefore both prefix and session affinities are **best-effort convergence** rather than hard routing: when the matching backend is filtered by utilization-filter, when its match rate is low and another backend overtakes its total score, or when the first request has no affinity data, the request is scheduled to a non-matching backend. This is expected behavior — load/utilization factors can break affinity to avoid dragging a hot backend to death.

### Runtime Metrics Pipeline

The utilization and queue metrics consumed by the kv/queue scorers are obtained by EPP periodically scraping each inference backend's `/metrics` (Prometheus format) over HTTP (refreshed at `RefreshMetricsInterval`, default 50ms, cached in local memory); the metrics source/extractor is injected by ai-gateway-epp by default. EPP's external dependencies across the whole chain are only two kinds of stateless pulls: Control Plane configuration (epp_data / cluster_table) and Data Plane metrics (backend `/metrics`). The prefix affinity LRU, session bindings, flow-control queues, and in-flight counts are all in-process state; after failover/restart they converge from scratch on cold start.

---

## End-to-End Flow

The complete sequence of an EPP-mode Cluster from creation to request scheduling is as follows:

```mermaid
sequenceDiagram
    participant OP as Ops/Deployment
    participant API as AI Gateway API
    participant EPP as ai-gateway-epp
    participant CA as Conf Agent
    participant BFE as BFE
    participant RS as Inference backends

    OP->>API: PATCH /epp-pool (groups + instance list)
    OP->>API: POST /clusters (balance_mode=EPP + epp_config [+ epp_runtime])
    API->>API: Validate epp_config / epp_runtime, greedy assigner picks group
    API->>API: Write epp_assignments (cluster → group)
    CA->>API: GET server_data_conf (version increment)
    API-->>CA: GslbBasic.BalanceMode=EPP + EPPAddr=[instance address]
    CA->>BFE: Hot reload cluster_conf.data
    EPP->>API: GET /configs/epp_data/config (version increment)
    API-->>EPP: Compiled EndpointPickerConfig + assignment full view
    EPP->>EPP: Match assignment by group identity, build Cells, compile engines
    EPP->>API: GET /configs/gslb_data/cluster_table
    API-->>EPP: Backend instance list (with Weight)
    EPP->>RS: Periodically scrape /metrics
    Client->>BFE: Inference request
    BFE->>EPP: gRPC ext_proc (metadata: inference-pool=<cluster>)
    EPP-->>BFE: Selected endpoint address
    BFE->>RS: Forward request
    RS-->>BFE: Response
    BFE-->>Client: Response
```

---

## Boundary Semantics

| Scenario | Behavior |
|----------|----------|
| Both backends fully overloaded | utilization-filter fail-closed filters all endpoints → EPP has no candidate → BFE silently falls back to local WRR and continues serving (`epp_fallback_local_total` grows) |
| **EPP admission rejection** | When EPP explicitly rejects via an ext_proc `ImmediateResponse` (429/503), BFE maps it to 429 `EPP_OVERLOAD` / 503 `EPP_NO_CAPACITY` and passes through `Retry-After`; the request is **not retried, does not fall back to local WRR, and does not feed the circuit breaker** — overload scenarios partially reject with 429/503 instead of admitting everything and queueing (distinct from the fail-open fallback semantics of the rows above) |
| Instance of a group lost | `EPPAddr` carries only the sole instance address of the group; when it is lost, BFE degrades to local balancing; after the instance recovers, EPP calls resume automatically per the EPPCheck hysteresis rules |
| EPP instance failure | BFE finds the address unavailable via EPPCheck, and the failed EPP path is carried by local WRR (requests still return 200); recovery is either the instance healing itself, or the Control Plane reassigning the Cluster to a healthy instance group (reconciler automatic / `PUT` override), with BFE switching at its next server_data_conf pull |
| Dangling assignment (instance group removed from pool) | `/epp-pool` PATCH triggers automatic reassignment; when the pool has no assignable candidate group, the assignment is cleared, export degrades to WRR + error log, and the automatic repair path reassigns after capacity recovers |
| `EPP → WRR` and switching back | epp_config and assignment records remain dormant; switching back to EPP continues with the original configuration and assignments |
| `-group-name` inconsistent with /epp-pool | The EPP instance matches no assignment in any Cluster: no Cells, no service; startup logs and the `ai_epp_assignment_no_match` metric alert |

---

## Chapter Summary

- EPP intelligent scheduling solves the three limitations of WRR: no load awareness, no prefix cache affinity, and no session affinity; BFE delegates backend selection to ai-gateway-epp via gRPC ext_proc.
- A Cluster's `balance_mode` (WRR/EPP) is the sole source of truth for EPP mode; `epp_config` uses a simplified "load profile + tuning parameters" user-facing form, deterministically compiled into the llm-d `EndpointPickerConfig` at export time, valid out of the box; `epp_runtime` carries health-check and call-timeout parameters in a simplified form, layered separately from `epp_config`, exported only when the mode is EPP and the assignment is valid, with EPPTLS/EPPBreaker fixed and not user-configurable in this release.
- `/epp-pool` statically registers the instance pool in singleton + full-replacement mode (exactly 1 instance per group, the array form reserved for scaling); the assigner automatically generates Cluster→instance-group assignments with greedy + deterministic tie-break, storing only the cluster→group mapping, with automatic repair when dangling.
- epp_data distributes the compiled configuration and the assignment full view merged in a single endpoint (version increment); all EPP instances share the same snapshot and self-match by group identity locally.
- server_data_conf exports the single-element `EPPAddr` to BFE; when there is no valid assignment, a single Cluster degrades explicitly to WRR + error log without blocking the whole distribution.
- On the BFE side, EPPCheck hysteresis drives address health determination; when EPP fails or has no candidate, BFE silently falls back to local WRR without interrupting business; recovery from an instance failure relies on Control Plane reassignment, and BFE switches address as soon as it pulls the new server_data_conf.
- Release constraint: ai-gateway-epp, ai-gateway-api, the schema upgrade (`epp_assignments` has no `primary_instance_id` column; the upgrade script handles the existing column and carries a rollback `ADD COLUMN`), and the frontend must be released together in the same version and are incompatible with older versions (in a mixed deployment, EPP instances all raise explicit no-match alerts rather than failing silently).

For implementation details see [Chapter 36 EPP Component Implementation](../implementation/chapter36-epp-implementation.md); for configuration and operations see [Chapter 26 EPP Scheduling Configuration and Operations](../operation/chapter26-epp-scheduling-operation.md).

---

## References

- [llm-d project homepage](https://llm-d.ai/) and [CNCF llm-d project page](https://www.cncf.io/projects/llm-d/)
- [Kubernetes Gateway API Inference Extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension) (EPP protocol and InferencePool API)
- `ai-gateway-epp/README.md` (differences between ai-gateway-epp and the upstream EPP)
- `ai-gateway-api/design-docs/sys-design/details/EPP调度对接.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/epp-pool.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/epp-assignments.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/clusters.md`
- `ai-gateway-api/design-docs/api-define/InnerAPI接口定义/epp-data.md`
- `ai-gateway-api/design-docs/api-define/InnerAPI接口定义/server-data-conf.md`
- `bfe/docs/zh_cn/sys_design/epp_integration.md`
