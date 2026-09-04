# AEP-95 Implementation Guide

> Companion to [AEP-95: GPU Interconnect (Multi-Node RDMA)](./README.md) (the authoritative specification).
>
> This document contains **implementation-specific details only**: repository identifiers, CRD fields, flags, the cluster manifests the provider expects, and the validation record. For the design, behavior, rules, and rationale, see [README.md](./README.md).

---

## 1. Identifiers

| Concept | Identifier | Where |
|---|---|---|
| SDL opt-in key | `gpu.attributes.interconnect` | chain-sdk `go/sdl` |
| Reserved implicit group | `auto` | chain-sdk `sdl.InterconnectGroupAuto` |
| On-chain GPU attribute | `interconnect/group=<name>` | chain-sdk `sdl.GPUAttributeInterconnectGroup` |
| Placement attribute | `capabilities/gpu-interconnect` | provider `inventory.AttributeInterconnectPlacement` |
| Fabric pins | `capabilities/gpu-interconnect/fabric/infiniband`, `.../fabric/roce` | provider `inventory.AttributeInterconnectFabric*` |
| Pod label | `akash.network/interconnect-group` | provider `builder.AkashInterconnectGroupLabelName` |
| Anti-affinity topology key | `kubernetes.io/hostname` | provider `builder.(*Workload).affinity` |
| NCCL env | `NCCL_IB_DISABLE`, `NCCL_IB_HCA` | provider `builder.addEnvVarsForDeployment` |
| Multus annotation | `k8s.v1.cni.cncf.io/networks` | provider `builder.(*Workload).podAnnotations` |
| Fabric detection | `/sys/class/infiniband/<dev>/ports/1/link_layer` | provider `operator/inventory/infiniband.go` |
| Default RDMA resource pattern | `rdma/rdma_shared_device_*` | provider `inventory.DefaultInterconnectResourcePattern` |

## 2. Node Capabilities (chain-sdk `inventory/v1`)

Discovery results are published per node in `NodeCapabilities`:

```protobuf
message NodeCapabilities {
  repeated string storage_classes            = 1;
  string          interconnect_resource_name = 2;  // e.g. rdma/rdma_shared_device_ib
  string          interconnect_fabric        = 3;  // "infiniband" | "roce"
  repeated string nccl_hca_prefixes          = 4;  // e.g. ["mlx5"]
}
```

A node is interconnect-capable when both `interconnect_resource_name` and `interconnect_fabric` are non-empty.

## 3. Manifest CRD (provider `akash.network/v2beta2`)

The bid-time pin is recorded per service under `scheduler_params.resources.interconnect`:

```yaml
interconnect:
  type: object
  nullable: true
  properties:
    enabled:           { type: boolean }
    units:             { type: integer, format: int64, minimum: 0 }
    resource_name:     { type: string }
    fabric:            { type: string }
    nccl_hca_prefixes: { type: array, items: { type: string } }
```

The off-chain service also carries `interconnect_group` (string) so the workload builder can label pods without re-walking attributes.

## 4. Provider Configuration

### Inventory operator (`provider.yaml` → inventory config)

```yaml
interconnect:
  # Glob patterns matched against kubelet extended-resource names to find
  # the RDMA pool. Empty list = the default "rdma/rdma_shared_device_*".
  # Bare strings (no glob metacharacters) match as a literal prefix.
  resource_patterns: []
```

Non-NVIDIA plugins: e.g. `["broadcom.com/rdma"]`, `["intel.com/iwarp_shared"]`.

### Provider service flag

| Flag | Default | Purpose |
|---|---|---|
| `--interconnect-roce-networks-namespace` | `akash-rails` | Namespace scanned for multus NADs to attach to RoCE interconnect pods; every NAD found is attached. Empty string disables attachment. Viper env: `AP_INTERCONNECT_ROCE_NETWORKS_NAMESPACE`. |

The provider service account needs `list` on `network-attachment-definitions.k8s.cni.cncf.io` in that namespace. The `akash-provider` Helm chart binds `cluster-admin`, which already covers it.

## 5. Bid-Time Algorithm (provider `cluster/kube/operators/clients/inventory`)

Within `Adjust`, for each service resource `i` and each replica:

```
requiresInterconnect[i], group[i] = ResourceInterconnectGroup(origResources[i])
if requiresInterconnect[i] and not IsValidLabelValue(group[i]):
    refuse bid (ErrInvalidInterconnectGroup)

for node in nodes:
    if requiresInterconnect[i]:
        if not interconnectCapsSuitable(node.Capabilities, requiredFabric): next node
        if group[i] != "" and groupClaims[group[i]][node]:            next node   # peer already here
    ... CPU / memory / GPU / storage gates ...
    if requiresInterconnect[i]:
        reserve gpu.units handles from node.Resources.GPUInterconnect
        sparams.Resources.Interconnect = {enabled, units, resource_name, fabric, nccl_hca_prefixes}
        groupClaims[group[i]][node] = true
```

`groupClaims` is scoped to one `Adjust` call (one bid), so unrelated orders cannot interfere. `requiredFabric` is read once from the deployment group's placement attributes.

## 6. Deploy-Time Rail Resolution (provider `cluster/kube/client.go`)

```
if any service's sparams has interconnect.enabled and fabric == "roce":
    if settings.InterconnectRoCENetworksNamespace == "":
        log opt-out; no annotation
    else:
        nads = list NetworkAttachmentDefinitions in namespace
        if API NotFound (multus CRD missing) or len(nads) == 0:
            fail deploy: ErrNoRoCERailNetworks
        settings.InterconnectRoCENetworks = sorted "<ns>/<name>" joined by ","
```

`podAnnotations()` applies the value only to services whose own pin is `fabric == "roce"`.

## 7. Cluster Manifests (RoCE rails)

One `IPPool` (nv-ipam) and one `MacvlanNetwork` (Network Operator) per rail. `perNodeBlockSize × nodes` must stay below the lowest host rail address.

```bash
kubectl create namespace akash-rails
RAILS=(enp26s0np0 enp60s0np0 enp77s0np0 enp94s0np0 enp156s0np0 enp188s0np0 enp204s0np0 enp220s0np0)
for k in "${!RAILS[@]}"; do
cat <<YAML | kubectl apply -f -
apiVersion: nv-ipam.nvidia.com/v1alpha1
kind: IPPool
metadata: { name: rail$k, namespace: nvidia-network-operator }
spec: { subnet: 10.100.$k.0/24, perNodeBlockSize: 16 }
---
apiVersion: mellanox.com/v1alpha1
kind: MacvlanNetwork
metadata: { name: rail$k }
spec:
  networkNamespace: akash-rails
  master: ${RAILS[$k]}
  mode: bridge
  mtu: 9000
  ipam: '{"type": "nv-ipam", "poolName": "rail$k"}'
YAML
done
```

Resulting pod (8-rail node): `net1`…`net8`, one per rail, each with a nv-ipam address and a RoCEv2 GID registered on the matching HCA.

## 8. Validation Record

| Environment | Test | Result |
|---|---|---|
| Mainnet provider, ConnectX-6 HDR InfiniBand, A100 | Marketplace lease, 2 services `interconnect: []`; pod-to-pod `ib_write_bw` | **197.39 Gb/s** (HDR line rate); distinct nodes; label, RDMA resource, NCCL env all present |
| 4-node Dell 8× H200 SXM, 8× ConnectX-7 400G rails (RoCEv2), k3s, Network Operator 26.1.1 | Raw host-to-host `ib_write_bw` | 391 Gb/s (line rate) — fabric baseline |
| same | Marketplace lease **before** rail attachment | RDMA device, env, placement correct; `ib_write_bw` fails at QP→RTR — pod has only the CNI interface |
| same | Akash-shape pod + multus rail attachment, single rail | **391.32 Gb/s** — fix confirmed |
| same | Akash-shape pods, all 8 rails, nv-ipam auto-IP, 8 concurrent streams | **2.09 Tb/s aggregate**, every rail live (benchmark CPU-bound; per-rail 190–392 Gb/s) |
| same, provider with #424 | Marketplace lease: `net1`–`net8` attached, annotation stamped, no `NCCL_IB_GID_INDEX`; pod-to-pod `ib_write_bw` | **391.37 Gb/s** — RoCE at parity with InfiniBand |

## 9. Diagnostic Notes

- The RDMA subsystem's netns mode (`rdma system show`) was `shared` on the validation cluster; the RoCE failure was **not** the device namespace but L3 resolution — the pod needs a netdev on the rail subnet.
- Host GID entries (indices 0–3 on `mlx5_0`) are not enumerable from inside the pod netns; the pod's own rail GIDs appear at 4–7 (v1/v2 × IPv4/link-local). `-x 5` is the IPv4 RoCEv2 GID in that layout.
- A macvlan interface moved into a running pod by hand does **not** register a usable RoCEv2 GID; attachment must happen at pod creation through the CNI.
- `kubectl get ippools` resolves to Calico's CRD; use `ippools.nv-ipam.nvidia.com`.
