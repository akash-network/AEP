# AEP-90 Implementation Guide

Companion to [AEP-90](./README.md). This document covers the change surface across repositories,
validation procedure, and rollout guidance. Normative SDL and matching behaviour is specified in the
AEP itself.

## 1. Change Surface

The change spans two repositories. The SDL schema must land first, since the provider consumes the
types it defines.

| Repository  | Change                                                        | Status                              |
| ----------- | ------------------------------------------------------------- | ----------------------------------- |
| `chain-sdk` | `arch` attribute on the CPU resource block; inventory types    | Merged (`go/v0.4.4`, `go/sdl/v0.4.4`) |
| `provider`  | Inventory detection, bid-engine filtering, startup ordering    | Feature branch, not yet upstream    |

### 1.1 chain-sdk

- **SDL schema**: `arch` added to the CPU resource block, optional, validated against the permitted
  value set at parse time so that an invalid value fails before it reaches the marketplace.
- **Inventory types**: node architecture carried through the inventory structures the provider
  populates and the bid engine reads.

### 1.2 provider

- **Inventory operator**: detects the CPU architecture of each node and includes it in reported
  hardware inventory. Detection is per-node, not per-cluster.
- **Bid engine**: architecture participates in resource matching alongside the existing CPU, memory,
  storage, and GPU comparisons. A node whose architecture does not match the request is excluded from
  candidacy for that resource.
- **Startup ordering**: see below.

## 2. The Startup Race

A node's reported architecture can be empty during the window between the node becoming visible to
the provider and inventory detection populating its hardware details.

If empty architecture is treated as "matches anything," the consequences are worse than a transient
error:

- Mismatched bids are made during the window, producing leases that cannot run the tenant's workload.
- Nothing is logged as a failure, because from the matcher's perspective the node satisfied the
  request.
- The symptom appears at the tenant's container start, one system away from the cause.

Two properties are therefore required of any implementation:

1. **Empty architecture matches nothing.** The comparison must fail closed rather than treating an
   absent value as a wildcard.
2. **Architecture is populated before matching sees the node.** Ordering must guarantee that
   inventory detection has completed for a node before that node is offered as a bid candidate.

The first alone is sufficient for correctness — mismatched bids cannot occur — but produces spurious
non-bids during startup. The second removes that window. Implement both.

## 3. Validation Procedure

Validation requires a provider running on genuinely arm64 hardware. An amd64 provider cannot exercise
the interesting cases, because every request it receives is one it can satisfy.

### 3.1 Negative cases first

Run these before the positive case. They confirm the default is enforced rather than merely present,
and they are the cases that would silently pass if filtering were not actually wired in.

1. **SDL with no `arch`, against an arm64 provider** — expect no bid. Confirms the omitted-field
   default resolves to `amd64` and that the resulting request is enforced.
2. **SDL with explicit `arch: amd64`, against an arm64 provider** — expect no bid. Confirms explicit
   requests filter identically to defaulted ones.

A bid appearing in either case means filtering is not effective, regardless of what the positive case
shows.

### 3.2 Positive case

3. **SDL with `arch: arm64`, against an arm64 provider** — expect a bid, a lease, and a deployed
   workload that starts and is reachable over its exposed endpoint.

Reachability matters here, not merely lease creation. A lease that is created but whose container
never starts is the exact failure this AEP prevents, so the test must confirm the workload actually
runs.

### 3.3 Node reporting

4. Query the provider's reported inventory and confirm each node's architecture matches the host's
   actual architecture.
5. Restart the provider and confirm architecture is populated before bids are evaluated — no window
   in which the node is bid-eligible with an empty architecture.

## 4. Rollout Guidance

### 4.1 Ordering

The chain-sdk tags must be consumed by the provider build before provider-side filtering functions.
Providers adopt the capability by upgrading; there is no chain-side gate and no governance action.

### 4.2 Mixed-version fleet behaviour

| Provider build | Hardware | Behaviour                                                                   |
| -------------- | -------- | ---------------------------------------------------------------------------- |
| Updated        | amd64    | Bids on `amd64` and defaulted requests; declines `arm64`                     |
| Updated        | arm64    | Bids on `arm64` only                                                          |
| Not updated    | amd64    | Bids as before; correct, because defaulted requests are `amd64`              |
| Not updated    | arm64    | **Bids on everything** — including requests it cannot satisfy                 |

The last row is the reason arm64 capacity should not be brought onto the network on a provider build
without filtering. The protection for tenants comes from the provider side; an un-updated arm64
provider is the unprotected case.

### 4.3 Tenant guidance

- Existing SDLs need no change. They continue to request `amd64` implicitly.
- A tenant whose image is arm64-only must set `arch: arm64` explicitly. Without it their deployment
  will not match arm64 providers.
- A tenant with a multi-arch image is not automatically eligible for both. The SDL requests one
  architecture; there is no "either" value in this proposal. Deploying to arm64 capacity requires an
  explicit `arch: arm64`.

## 5. Diagnosing a Missing Bid

Architecture filtering produces silent non-bids by design, which is correct behaviour but
indistinguishable at first glance from any other unsatisfiable requirement. When a provider that
should bid does not:

1. Confirm the SDL's effective architecture — remember that an omitted `arch` is a request for
   `amd64`, not a wildcard.
2. Query the provider's reported inventory and confirm the node's architecture is populated and
   correct. An empty value matches nothing and is the most likely cause of a provider declining
   requests it should satisfy.
3. Confirm the provider build includes the inventory and bid-engine changes; an older build reports no
   architecture at all.