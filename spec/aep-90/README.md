---
aep: 90
title: "CPU Architecture Awareness in Bid Matching"
author: Scott Carruthers (@chainzero)
status: Draft
type: Standard
created: 2026-09-03
category: Core
---

## Summary

Akash has no way for a provider to report the CPU architecture of its hardware and no way for a
tenant to request one. The ecosystem has been effectively amd64-only, so the omission has been
harmless. As arm64 hardware becomes viable for providers, it stops being harmless: an amd64-built
container image scheduled onto an arm64 node fails at runtime, after the tenant has already paid for
the lease.

This AEP adds CPU architecture as a first-class attribute of the compute resource. The SDL gains an
`arch` field on the CPU resource block, the provider inventory reports each node's detected
architecture, and the bid engine filters candidate nodes against the requested architecture. An SDL
that does not specify `arch` defaults to `amd64`, so existing deployments cannot be silently routed
onto incompatible hardware.

## Abstract

Architecture mismatch is a failure that surfaces late and costs the tenant. Nothing in the current
bid path carries architecture information, so a provider with arm64 capacity either cannot advertise
it or advertises capacity that will fail for most tenant images, and a tenant has no way to express a
requirement their workload genuinely has.

This proposal closes the gap in three places: the SDL schema, where a tenant declares the
architecture their image was built for; the provider inventory, where each node reports its detected
architecture; and the bid engine's resource matching, where the two are compared before a bid is
made. The design is deliberately fail-closed — an unspecified request defaults to `amd64`, and a node
that reports no architecture matches nothing — so that both older SDLs and older provider builds
behave safely rather than permissively.

## Motivation

Akash providers today operate homogeneous clusters, and the network has assumed a single
architecture throughout. Two consequences follow.

**Tenants cannot express a real requirement.** A container image is built for a specific
architecture. An `amd64` image will not execute on an `arm64` kernel, and the failure is not
graceful: the workload fails at container start with an architecture mismatch. Because nothing in the
bid path carries architecture, a tenant has no way to prevent this. The mismatch is discovered after
bid acceptance and lease creation — that is, after payment has begun — and the tenant bears the cost
of a lease that was never capable of running their workload.

**Providers cannot advertise heterogeneous capacity.** Arm64 server SKUs and Apple Silicon hardware
are increasingly viable as provider capacity. A provider bringing that hardware onto the network
today has no way to signal it. Their nodes bid on every matching order regardless of architecture, so
adding arm64 capacity actively degrades tenant outcomes rather than expanding network supply.

The gap is not merely missing metadata. Because the ecosystem is currently all-amd64, the absence of
an architecture signal is indistinguishable from an implicit amd64 assumption held by every existing
SDL. Introducing arm64 capacity without an explicit signal converts that implicit assumption into a
silent failure mode for deployments written before arm64 existed on the network.

## Specification

### SDL Schema

The CPU resource block gains an optional `arch` attribute:

```yaml
profiles:
  compute:
    web:
      resources:
        cpu:
          units: 1
          arch: arm64
        memory:
          size: 512Mi
        storage:
          size: 512Mi
```

- **Permitted values**: `amd64`, `arm64`. Any other value is rejected at SDL validation.
- **Optional**: the field may be omitted.
- **Default when omitted**: `amd64`.

The attribute belongs to the CPU resource block rather than to placement attributes because it is a
property of the compute resource being requested, participates in resource matching, and is
machine-detected on the provider side rather than operator-asserted.

### Provider Inventory

The inventory operator detects and reports each node's CPU architecture as part of its hardware
inventory. Architecture is a per-node property: a cluster may contain nodes of differing
architectures, and each is reported independently.

A node whose architecture is unreported or empty MUST NOT be treated as matching any architecture.
Absent information is not a wildcard — see [Fail-Closed Semantics](#fail-closed-semantics).

### Bid Matching

The bid engine's resource-matching logic compares the requested architecture against each candidate
node's reported architecture. A node whose architecture differs from the request is excluded from
consideration for that resource.

Where no node in the cluster satisfies the requested architecture, the provider does not bid. This is
the correct outcome and is not an error condition: the tenant sees no bid from that provider, exactly
as with any other unsatisfiable resource requirement, rather than receiving a bid that would fail
after lease creation.

### Fail-Closed Semantics

Two defaults are load-bearing, and both resolve ambiguity toward rejecting a bid rather than making
one:

| Condition                          | Behaviour             | Consequence of the opposite choice                                            |
| ---------------------------------- | --------------------- | ----------------------------------------------------------------------------- |
| SDL omits `arch`                   | Request `amd64`       | Pre-existing SDLs become eligible for arm64 nodes — the exact failure to avoid |
| Node reports empty/unknown `arch`  | Matches nothing       | A detection bug silently disables filtering for the entire cluster             |

The second case is not hypothetical. A node's reported architecture can be empty during provider
startup before inventory detection has populated it, and treating empty as a wildcard would allow
mismatched bids during that window while producing no error anywhere. Provider implementations MUST
ensure architecture is populated before a node is offered to resource matching.

## Rationale

### Why default to `amd64` rather than "unconstrained"

An unconstrained default would treat an SDL with no `arch` as willing to run anywhere. Since every
SDL written before this proposal has no `arch`, that interpretation makes the entire existing corpus
of deployments eligible for arm64 hardware — reintroducing precisely the mismatch this AEP exists to
prevent, and doing so silently.

Defaulting to `amd64` instead encodes the assumption those SDLs were actually written under. It
preserves current behaviour exactly while all providers are amd64, and it protects existing
deployments the moment any are not.

The cost is that a tenant with a genuinely architecture-agnostic workload (an interpreted runtime, or
a multi-arch image) must opt in explicitly to be scheduled on arm64. That is the correct trade: the
tenant who knows their workload is portable can say so, whereas the tenant who does not know cannot
be asked to have anticipated a capability that did not exist when they wrote their SDL.

### Why the CPU resource block rather than placement attributes

Placement attributes are operator-asserted key/value strings. Architecture is machine-detectable, is
a property of the compute resource rather than of the deployment's placement policy, and needs a
default that applies to SDLs which never mentioned it. Attributes provide none of those: they carry
no defaulting semantics, and an attribute-based approach would leave every pre-existing SDL
unprotected.

### Alternative approaches considered

**1. Express architecture as a placement attribute.** Providers publish an `arch` attribute; tenants
match on it in the placement block.

- *Pros*: requires no schema change and works with the existing attribute mechanism.
- *Cons*: self-reported and unverified; no defaulting, so every SDL predating arm64 capacity remains
  exposed; places a resource property in the placement policy layer. Rejected.

**2. Infer architecture from the container image manifest.** The provider inspects the image at bid
time and matches its declared platform.

- *Pros*: no SDL change at all; the requirement is derived from the artifact that actually carries it.
- *Cons*: requires the provider to reach the registry during bid evaluation, including for private
  images requiring credentials; multi-arch manifests are ambiguous as a requirement signal; adds
  network latency and an external dependency to the bid path. Rejected.

**3. Treat unknown node architecture as matching any request.** Simplifies provider upgrade ordering.

- *Cons*: converts any detection failure into silent mis-scheduling across the whole cluster, with no
  error surfaced to either party. Rejected in favour of fail-closed. See
  [Fail-Closed Semantics](#fail-closed-semantics).

## Backwards Compatibility

This proposal is backwards compatible and introduces no consensus change.

- **Existing SDLs** are unaffected. An SDL with no `arch` requests `amd64`, which every current
  provider satisfies, so matching behaviour is identical to today.
- **The `arch` field is optional**, so no SDL requires modification.
- **Providers running earlier builds** do not report architecture and do not filter on it. Because the
  ecosystem they operate in is amd64, their bids remain correct for the default request.
- **Mixed-version fleets** behave correctly: an updated arm64 provider filters and declines mismatched
  orders, while an un-updated amd64 provider continues bidding on the amd64 requests it can satisfy.

No migration is required. Tenants opt into arm64 by adding the field.

## Test Cases

### SDL validation

1. `arch: amd64` and `arch: arm64` parse successfully.
2. An unrecognised value is rejected at validation with a clear error.
3. An SDL omitting `arch` parses and resolves to a request for `amd64`.

### Bid matching on a single-architecture cluster

Against a provider whose nodes are arm64:

1. An SDL with no `arch` does **not** receive a bid.
2. An SDL with explicit `arch: amd64` does **not** receive a bid.
3. An SDL with `arch: arm64` receives a bid, and the resulting lease deploys a workload that starts
   and is reachable.

The first two cases are the ones that matter: they confirm the default is applied and is enforced,
rather than being nominally present but bypassed.

### Node architecture reporting

1. A node's reported architecture matches the host's actual architecture.
2. A node whose architecture is unpopulated matches no request, including a request that would
   otherwise be satisfiable.
3. Architecture is populated before the node becomes available to resource matching, including
   immediately following provider restart.

### Mixed-architecture cluster

1. A cluster containing both amd64 and arm64 nodes bids on both `arch: amd64` and `arch: arm64`
   orders, and each lease is placed on a node of the requested architecture.

## Security Considerations

**Architecture is a provider-reported signal.** It is machine-detected rather than operator-asserted,
which makes it more reliable than a placement attribute, but a provider still controls its own
inventory reporting. A provider reporting an architecture it does not have gains nothing: the
resulting workload fails to start, which costs the provider a failed lease and the associated
completion-rate consequences. There is no privilege escalation or data exposure path here.

**Fail-closed defaults limit the blast radius of implementation faults.** Both defaults resolve toward
not bidding. A detection bug therefore removes a provider from consideration rather than causing
mismatched leases, so the failure mode of this feature is lost bids rather than broken tenant
workloads.

**Interaction with inventory verification.** Where provider inventory becomes independently attested,
reported architecture is a natural candidate for inclusion in that attestation, at which point the
signal moves from provider-asserted to auditor-verified. This proposal does not depend on that and
does not block on it.

## Implementation

### chain-sdk

SDL schema and inventory types carry the `arch` attribute on the CPU resource block, with validation
constraining it to the permitted values. Merged and tagged as `go/v0.4.4` and `go/sdl/v0.4.4`.

### provider

Three changes:

1. **Inventory operator** — detects each node's CPU architecture and reports it as part of the
   hardware inventory.
2. **Bid engine** — resource matching filters candidate nodes against the requested architecture.
3. **Startup ordering** — ensures a node's reported architecture is populated before the node is
   offered to matching, so that a startup race cannot leave architecture empty and silently disable
   filtering.

Complete on a feature branch and validated end-to-end against an arm64 provider; not yet merged
upstream.

Detection specifics, validation procedure, and rollout guidance are in the
[Implementation Guide](./IMPLEMENTATION.md).

## References

- [AEP-3: Stack Definition Language Specification](https://akash.network/roadmap/aep-3/)
- [Implementation Guide](./IMPLEMENTATION.md)

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).