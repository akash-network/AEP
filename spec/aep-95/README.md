---
aep: 95
title: "GPU Interconnect (Multi-Node RDMA)"
description: "One SDL attribute for RDMA-connected multi-node GPU workloads plus how providers advertise and schedule InfiniBand and RoCE fabrics and wire pods onto them"
author: Zeke Ezagui (@zblocker64)
status: Draft
type: Standard
category: Core
created: 2026-09-04
discussions-to: https://github.com/orgs/akash-network/discussions
roadmap: major
requires: 24
---

## Summary

This AEP defines **GPU interconnect** on Akash: how a tenant asks for GPU nodes that are joined by a high-speed RDMA fabric (InfiniBand or RoCE), how a provider advertises that it has one, and what the provider does at bid time and deploy time to deliver it.

A tenant opts in with a single nested attribute on a GPU resource, `gpu.attributes.interconnect`, and steers the deployment to capable providers with the placement attribute `capabilities/gpu-interconnect`. The provider detects its fabric from the hardware, spreads the peers of an interconnect group across distinct nodes, allocates one RDMA device handle per GPU, injects the NCCL environment, and — on RoCE fabrics — attaches the rail networks the pods need to reach the fabric. The SDL is identical for InfiniBand and RoCE; the tenant never configures fabric, GID indices, or networks.

The design is implemented and validated: InfiniBand at HDR line rate (197 Gb/s) through a mainnet lease, and RoCE at 400G line rate (391 Gb/s per rail, 2.09+ Tb/s across eight rails) through a marketplace lease on an 8× H200 rail-optimized cluster.

## Motivation

[AEP-24](../aep-24) brought GPUs to the marketplace as a per-node resource. That is sufficient for inference and single-node training, but the workloads that now dominate GPU demand — training and fine-tuning models that do not fit on one node — need **several nodes acting as one job**, exchanging gradients over a fabric that is orders of magnitude faster than the pod network. In practice that means RDMA over InfiniBand or RoCE, driven by NCCL.

Before this proposal Akash had no way to express that need, and no way for a provider to satisfy it even if it wanted to:

1. **No tenant vocabulary.** An SDL could ask for 8 GPUs in each of two services, but could not say "these two services must be on different nodes *and* be able to reach each other over RDMA." Two services could land on the same node, or on nodes with no fabric between them.
2. **No provider advertisement.** A provider with an InfiniBand or RoCE fabric had no attribute to advertise it and no way to restrict bidding to orders it could actually serve.
3. **No device plumbing.** RDMA requires the pod to hold an RDMA device handle from the node's HCA, the right NCCL environment, and — on RoCE — network interfaces on the fabric's rail subnets. None of this is expressible from a tenant's SDL, and none of it should be: it is provider-side hardware detail.
4. **Fabric heterogeneity.** InfiniBand and RoCE need different plumbing (RoCE is IP-addressed, InfiniBand is not) but are interchangeable from NCCL's point of view. Tenants should not have to know which one a provider runs.

This AEP addresses all four.

## Design Principles

- **One attribute, fabric-agnostic.** A tenant writes `interconnect: []` and gets RDMA. Which fabric, which device plugin, which GID index — all provider concerns. A tenant *may* pin a fabric via a placement attribute, but never has to.
- **Groups, not just a flag.** Interconnect is a relationship between peers. The SDL expresses *which* services and replicas belong together, and the provider guarantees they land on distinct nodes. Several independent groups can coexist in one deployment.
- **Detect, don't configure.** The provider discovers fabric type, HCA families, and the RDMA resource name from the node. Operators install the standard NVIDIA operators and (for RoCE) create rail networks; they do not describe their hardware to the provider.
- **Opt-in and isolated.** Only resources that declare `interconnect` are affected. Every other GPU or non-GPU workload is untouched, on interconnect-capable providers and elsewhere.
- **Refuse rather than deliver broken.** A provider will not bid on an interconnect order it cannot serve, and will fail a RoCE deploy rather than produce pods that hold an RDMA device but cannot reach the fabric.

## Specification

### 1. SDL Changes

A tenant opts a GPU resource into interconnect with the `interconnect` key under `gpu.attributes`. It has exactly two forms.

**Form A — implicit group.** Every service and replica in the same placement that writes the empty sequence joins one shared group, named `auto` by the parser:

```yaml
profiles:
  compute:
    worker:
      resources:
        cpu: { units: 32 }
        memory: { size: 512Gi }
        storage: { size: 1Ti }
        gpu:
          units: 8
          attributes:
            vendor:
              nvidia:
                - model: h100
            interconnect: []
```

**Form B — explicit named groups.** For several independent peer groups in one deployment:

```yaml
        gpu:
          units: 8
          attributes:
            vendor:
              nvidia:
                - model: h100
            interconnect:
              group: trainer
```

The group name must be a valid Kubernetes label value (`[A-Za-z0-9]([-A-Za-z0-9_.]*[A-Za-z0-9])?`, at most 63 characters); the provider uses it verbatim as a pod label.

#### Placement requirement

Any deployment that uses `interconnect` must steer itself to capable providers:

```yaml
deployment:
  worker:
    dc:
      profile: worker
      count: 2
placement:
  dc:
    attributes:
      capabilities/gpu-interconnect: "true"
      # optional fabric pin — one of:
      # capabilities/gpu-interconnect/fabric/infiniband: "true"
      # capabilities/gpu-interconnect/fabric/roce: "true"
```

Without a fabric pin, any provider advertising `capabilities/gpu-interconnect` may bid and the provider's own fabric is used.

#### Validation rules

The SDL parser rejects:

1. Any `interconnect` opt-in whose deployment placement does **not** require `capabilities/gpu-interconnect: "true"`. This guarantees an interconnect workload can only reach an interconnect-capable provider.
2. The reserved group name `auto` written explicitly under `interconnect: { group: ... }`.
3. Mixing implicit (`[]`) and explicit (`{ group: ... }`) opt-ins within one placement. Every service in a placement shares one group-label vocabulary, so the anti-affinity rule is unambiguous.
4. `interconnect` on a resource with `gpu.units: 0`.
5. Any other shape: a bare scalar (including `true`/`false`), a non-empty sequence, or a mapping without `group`.

#### On-chain projection

The parser emits the opt-in as a single GPU attribute on the resource, `interconnect/group = <name>` (`auto` for Form A). Its *presence* is the opt-in signal; its *value* is the peer-group label. The placement attributes are ordinary deployment-group requirements. No new chain types, messages, or parameters are introduced; the feature rides entirely on the existing attribute vocabulary of [AEP-24](../aep-24).

#### Replicas

Each replica of a `count: N` service is a peer in its group and is placed on its own node. A service with `count: 2` and `interconnect: []` yields two pods on two nodes; two such services yield four pods on four nodes.

### 2. Provider Attributes

A provider with an RDMA fabric advertises, in `provider.yaml`:

| Attribute | Value | When |
|---|---|---|
| `capabilities/gpu-interconnect` | `"true"` | Always, if the provider offers multi-node RDMA |
| `capabilities/gpu-interconnect/fabric/infiniband` | `"true"` | If nodes use InfiniBand |
| `capabilities/gpu-interconnect/fabric/roce` | `"true"` | If nodes use RoCE |

A provider advertises every fabric it actually has. Bid matching on these keys is exact string equality, the same as every other placement attribute.

### 3. Provider Discovery

The provider's inventory operator discovers interconnect capability per node, with no operator-supplied hardware description:

| Capability | Source |
|---|---|
| **Fabric** (`infiniband` or `roce`) | `/sys/class/infiniband/<first device>/ports/1/link_layer` — `InfiniBand` → `infiniband`, `Ethernet` → `roce` |
| **HCA families** (e.g. `["mlx5"]`, or `["mlx5","bnxt_re"]` on a mixed-vendor host) | device names under `/sys/class/infiniband`, reduced to their driver-family prefix |
| **RDMA resource name** (e.g. `rdma/rdma_shared_device_ib`) | the node's allocatable extended resources, matched against a configurable glob list (`interconnect.resource_patterns`); the default `rdma/rdma_shared_device_*` matches the NVIDIA `k8s-rdma-shared-device-plugin` |

A node with a fabric and an RDMA resource is interconnect-capable. A node missing either is not, and is never selected for interconnect workloads. Discovery re-runs when the node or the operator configuration changes.

### 4. Bid Matching and Scheduling

When an order arrives, the provider's inventory `Adjust` step applies, per interconnect-enabled resource:

1. **Capability gate.** The candidate node must be interconnect-capable (Section 3). If the order pins a fabric, the node's fabric must equal it. Otherwise the node is skipped.
2. **Group separation.** Within one bid, two peers carrying the same `interconnect/group` value are never placed on the same node. This applies across services and across replicas of one service. If the cluster cannot host every peer on a distinct node, the provider does not bid.
3. **Device allocation.** One RDMA device handle per GPU unit is reserved from the node's RDMA resource pool (the shared device plugin publishes a pool — 63 handles per node by default — so a `gpu.units: 8` service consumes 8 handles, not 8 physical cards).
4. **Label validity.** A group name that is not a valid Kubernetes label value causes the provider to refuse the bid outright, rather than win a lease whose pods can never be admitted.

The pinned outcome is recorded in the per-service scheduler parameters of the provider's off-chain manifest: `enabled`, `units`, `resource_name`, `fabric`, and the HCA family list. Everything downstream reads from that record, so bid-time and deploy-time decisions cannot diverge.

### 5. Workload Construction

For a service whose scheduler parameters carry an interconnect pin, the workload builder produces:

**Resources.** The RDMA extended resource is requested with `requests == limits == <gpu units>` (the kubelet rejects mismatched device-plugin resources).

**Placement.** The pod is labelled `akash.network/interconnect-group=<group>` and carries a *required* pod anti-affinity on `kubernetes.io/hostname` against that label — the deploy-time enforcement of the bid-time guarantee in Section 4.

**NCCL environment.** The builder injects:

| Variable | Value |
|---|---|
| `NCCL_IB_DISABLE` | `0` |
| `NCCL_IB_HCA` | the node's HCA families, comma-joined (e.g. `mlx5`) |

An SDL that sets either variable in `service.env` wins; the provider never overwrites a tenant value. No `NCCL_IB_GID_INDEX` is injected on either fabric — see Rationale.

**RoCE rail attachment.** RoCEv2 is IP-addressed: establishing a queue pair resolves the *remote rail IP* through the pod's own network namespace. A pod with only the cluster CNI interface therefore cannot complete RDMA connection setup even with the RDMA device allocated — the fabric is unreachable from inside the pod. InfiniBand is LID-addressed and has no such requirement.

When the pinned fabric is `roce`, the provider attaches every multus `NetworkAttachmentDefinition` found in a dedicated rails namespace (default `akash-rails`, configurable via `--interconnect-roce-networks-namespace`) to the pod through the `k8s.v1.cni.cncf.io/networks` annotation. The list is resolved at deploy time, so an operator adding a rail needs neither a provider restart nor a configuration change. The pod comes up with one secondary interface per rail, each with an IP on that rail's subnet and a pod-owned RoCEv2 GID on the corresponding HCA.

If the fabric is `roce` and the namespace yields no attachments, **the deploy fails** with an explicit error rather than producing a lease whose RDMA can never work. Setting the namespace to the empty string is the explicit operator opt-out.

Nothing in this section runs for InfiniBand deployments beyond the resource, label, affinity, and NCCL environment.

### 6. Cluster Prerequisites

The provider software assumes, and the provider documentation prescribes, the standard NVIDIA operator stack:

- **GPU Operator** with `driver.rdma.enabled: true` (loads `nvidia-peermem` for GPUDirect RDMA).
- **Network Operator** with a `NicClusterPolicy` deploying MOFED/DOCA, the RDMA shared device plugin (`rdma_shared_device_ib`, `rdmaHcaMax: 63`), multus, and nv-ipam.
- **RoCE only:** each rail NIC addressed on its own subnet (one `/24` per rail is the convention), and one `MacvlanNetwork` + nv-ipam `IPPool` per rail in the `akash-rails` namespace. Pod IP blocks are carved per node by nv-ipam and must not overlap host rail addresses.

No Akash-specific components are installed on the cluster beyond the provider itself.

## Rationale

### Why a nested GPU attribute instead of `params`?

Interconnect is a property *of the GPU resource* — it says "these GPUs participate in a fabric" — and the group name must be visible to the bid engine, which reads on-chain resources. GPU attributes are already on-chain and already the vocabulary [AEP-24](../aep-24) established for GPU properties (`vendor`, `model`, `ram`, `interface`), so `interconnect` sits naturally beside them and needs no protobuf or chain changes. Service `params` are off-chain manifest data and invisible at bid time.

### Why `[]` for the implicit form?

The common case — "everything in this placement is one job" — should be as short as possible while remaining unambiguous. An empty sequence is a distinct YAML shape from a mapping, so the two forms cannot be confused, and it cannot collide with a future scalar or flag meaning. `true`/`false` scalars were deliberately rejected: a boolean invites `interconnect: false`, which has no meaningful semantics for a resource that is otherwise unaffected.

### Why is `auto` reserved?

If a tenant could write `group: auto` explicitly, a placement could mix `[]` resources and `{group: auto}` resources that *look* different but are the same group, or the reverse intent could be defeated by a name collision. Reserving the name keeps the "implicit means one shared group" reading exact.

### Why fabric-agnostic by default?

NCCL uses the InfiniBand verbs API for both fabrics; a training job behaves identically on InfiniBand and RoCEv2 at the same link rate. Forcing tenants to choose would fragment the marketplace (an InfiniBand-pinned order cannot use an otherwise-perfect RoCE provider) for no benefit to the workload. The pin exists for the tenant who genuinely needs one fabric — for example, a benchmark — not as the default path.

### Why detect the fabric instead of configuring it?

The link layer is a fact about the port that the kernel already exposes. Asking operators to restate it in `provider.yaml` would create a second source of truth that can disagree with the first. The same reasoning applies to HCA families and the RDMA resource name.

### Why attach rail networks from a namespace rather than a list?

An earlier design passed the NAD list as a provider flag. Every RoCE provider following the runbook would write the same value modulo rail count, the list would duplicate information the cluster already holds, and a typo would fail silently. Scanning a dedicated namespace makes the cluster the single source of truth, lets the default (`akash-rails`) cover every runbook-following provider with zero configuration, and lets rails be added live.

### Why no `NCCL_IB_GID_INDEX`?

An earlier implementation injected `NCCL_IB_GID_INDEX=3` on RoCE, the conventional index of the RoCEv2 GID on a host. With rail networks attached, the GID that NCCL must use belongs to the pod's *own* rail interface and is registered at a pod-specific index (observed as 5 on the validation cluster), while the host's entries at 0–3 are not visible from the pod's network namespace. Any fixed value is therefore wrong. Modern NCCL selects the RoCEv2 GID from the pod's table correctly on its own, so the correct behavior is to inject nothing. A tenant who needs a specific index can still set it in `service.env`.

### Why fail a RoCE deploy with no rails instead of proceeding?

A RoCE pod without rail interfaces holds an RDMA device that cannot reach any peer; NCCL hangs and the tenant pays for a dead fabric with no way to see why. Failing the deploy converts a silent tenant-facing failure into a loud operator-facing one, and enforces the provider documentation's rule that `fabric/roce` must not be advertised without rails.

## Backward Compatibility

- SDLs without `gpu.attributes.interconnect` are unchanged. The attribute is optional and its absence has no effect.
- The new attribute keys (`interconnect/group`, `capabilities/gpu-interconnect`, `capabilities/gpu-interconnect/fabric/*`) are ordinary attributes; no chain upgrade, parameter change, or protobuf change is required.
- Providers without RDMA hardware never advertise `capabilities/gpu-interconnect` and therefore never receive interconnect orders (SDL validation rule 1).
- InfiniBand providers are unaffected by the RoCE rail-attachment path; it is gated on the discovered fabric being `roce`.
- The provider's `Manifest` CRD schema gains an optional `interconnect` block under `scheduler_params.resources`. Existing manifests without it are valid.

## Security Considerations

- **No new privileges for tenant pods.** Interconnect pods run with the same unprivileged security context as any other Akash workload. RDMA device access comes from the device plugin's extended resource, not from `privileged` or added capabilities.
- **Shared HCA.** The RDMA shared device plugin exposes one physical HCA to many pods. Tenants on the same node share the device's queue-pair and memory-registration capacity; this is the standard Kubernetes RDMA model and is no different from sharing a GPU's PCIe link, but providers should be aware that a misbehaving tenant can degrade fabric performance for co-located tenants on the same node.
- **Rail network exposure (RoCE).** A RoCE interconnect pod holds an interface on each rail subnet and can reach every other pod and host address on those rails at L2. Rail subnets must be dedicated to the RDMA fabric and carry nothing else; they are not tenant-isolated networks. Providers should treat the rail namespace as security-sensitive: any NAD placed in `akash-rails` is attached to tenant pods.
- **Memory locking.** RDMA memory registration requires locked memory. On the validated clusters the container runtime's default memlock limit was unlimited; providers whose runtime imposes a limit may need to raise it for interconnect workloads. This is a cluster-configuration concern, not a provider-code one.
- **Group names are tenant-controlled label values.** The provider validates them against Kubernetes label-value rules before bidding, so a hostile name cannot produce an unadmittable pod or inject into a selector.

## Reference Implementation

- SDL grammar, validation, and on-chain projection: `akash-network/chain-sdk` (released in `v0.4.1`).
- Provider discovery, bid matching, and workload construction: `akash-network/provider` [#400](https://github.com/akash-network/provider/pull/400) (released in `v0.16.0`).
- RoCE rail-network attachment and GID-index removal: `akash-network/provider` [#424](https://github.com/akash-network/provider/pull/424).
- Provider and deployer documentation: `akash-network/website` [#1268](https://github.com/akash-network/website/pull/1268) and the RoCE rail-networks follow-up.

Implementation details — CRD fields, flags, cluster manifests, and validation data — are in [IMPLEMENTATION.md](./IMPLEMENTATION.md).

## Future Work

- **Rail auto-discovery by label** rather than by namespace, if operators want rails to live alongside other NADs.
- **HCA family scoping.** `NCCL_IB_HCA` currently names every discovered family (e.g. `mlx5`), which on rail-optimized nodes also matches storage/management HCAs of the same family. Inside the pod those have no usable GIDs and NCCL skips them, but restricting the list to fabric HCAs would be cleaner.
- **Topology-aware scheduling.** Placing peers on nodes that share a leaf switch, or on the same rail group, is not yet expressed. The group model is the natural place to attach such hints.
- **Non-NVIDIA device plugins.** The resource-pattern mechanism already admits Broadcom and Intel RDMA plugins; end-to-end validation on such hardware is outstanding.
