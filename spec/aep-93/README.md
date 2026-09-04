---
aep: 93
title: "Console: Async Actions & Notification Center"
description: "Run blockchain actions as persistent server-side jobs and show their progress, together with alerts, in an in-app notification center instead of a blocking modal"
author: Maxime Beauchamp (@baktun14)
status: Draft
type: Standard
category: Interface
created: 2026-09-04
estimated-completion: 2026-12-31
discussions-to: https://github.com/orgs/akash-network/discussions
roadmap: minor
requires: 63, 84
---

## Abstract

Every blockchain action in Akash Console (create, update, close or fund a deployment, create a lease, reclaim escrow) opens a full-screen modal that cannot be dismissed and holds the user until the transaction is confirmed. If they refresh or navigate away, the action disappears from the UI even though the server may still complete it.

This AEP makes those actions asynchronous and persistent. The browser enqueues an action, the API signs, broadcasts and confirms it as a background job, and the user keeps working. Progress and outcome appear in a notification center: a bell with an unread badge, a dropdown of recent items, toasts on completion, and a full activities page. Alerts that today go out by email only (deployment balance, wallet balance, deployment closed) are folded into the same feed.

Async mode is explicit opt-in through `?async=true` on the existing transaction endpoint. The synchronous response stays the default for every caller, permanently. The work is confined to `apps/api`, `apps/deploy-web`, `apps/tx-signer` and `apps/notifications` in `akash-network/console`. No chain or provider changes.

## Motivation

The blocking modal is the classic foreground process that should be a background one. Closing a deployment takes a few seconds of chain time; the user does not need to watch it, but the modal makes them, and a page refresh loses the in-flight action from view.

What makes the change tractable now is that Console is managed-wallet only (AEP-84). Every transaction is already signed and broadcast entirely server-side, keyed by user: the browser posts to `POST /v1/tx`, the API hands the messages to the transaction signer, and the signer talks to the chain. Nothing about that needs the browser to stay open. So this is a server-orchestration and UI refactor, not a change to how anything is signed. If self-custody ever returned to Console its browser-must-sign flow would be excluded from backgrounding.

The change also fits two adjacent efforts. Console Redesign v2 wants low-level mechanics out of the way. The escrow-abstraction work ([AEP-91](../aep-91)) produces automatic funding events that have no natural home in the UI today; an activity feed is exactly that home.

## Specification

### Scope

In scope:

- A persistent execution record for every asynchronous transaction and a unified activity feed, both in `apps/api`.
- A two-phase prepare and broadcast split in `apps/tx-signer`.
- An opt-in asynchronous mode on `POST /v1/tx` and new feed endpoints.
- The notification center UI in `apps/deploy-web`: bell, badge, dropdown, activities page, toasts, optimistic pending state, migration of every transaction call site.
- An in-app sink for fired alerts from `apps/notifications`.

Out of scope:

- Email, push or WebSocket delivery. The feed is polled; that matches every existing pattern in the app.
- Server-side bid selection or a full deployment saga. Bid selection stays an interactive client-side page.
- Any change to the signer's custody model or the funding-wallet flow.
- Flipping the synchronous default. `POST /v1/tx` without `?async=true` behaves as it does today, permanently.

### Goals

- Blockchain actions are non-blocking and survive a page refresh; the server completes them with the browser closed.
- One notification center shows async action status and alert firings side by side.
- A toast and the right query invalidation fire on completion; every item deep-links to the right page.
- Correctness under retries and crashes: no double broadcast, and no false failure that invites a user to retry a transaction that already executed.
- Nothing breaks for an existing caller: API keys, unmigrated browser bundles and third parties see no change unless they opt in.

### A. Architecture

```
Browser ── POST /v1/tx?async=true ──▶ TxActionService.enqueue
                            │  (one DB transaction, outbox pattern)
                            ├─ insert tx_action   (status=QUEUED, deadlineAt=null, gas=null)
                            ├─ insert activity    (kind=ACTION, status=PENDING, link)
                            └─ enqueue TxBroadcastJob         ◀ same transaction
                            ▼
                 202 { activityId }          |   default: today's sync body

  tx-broadcast queue (retryLimit 0, at most once)
    lock tx_action FOR UPDATE, require status == QUEUED
    re-validate (balances, fee grant, trial rules)
    tx-signer prepare  →  { signedTxBytes, txHash, gas, timeoutTimestamp }   (sign only)
    persist txHash, gas, signedTx, deadlineAt; status=BROADCASTING           ◀ hash known before any broadcast
    tx-signer broadcast { signedTxBytes }  →  status=BROADCAST
    enqueue TxConfirmJob   (deterministic pre-mempool rejection → FAILED; ambiguous → stays BROADCASTING)

  tx-confirm queue (idempotent, retryable)
    poll getTx(persisted txHash) until found or now > deadlineAt
    found, code 0   → CONFIRMED, fire idempotent post-tx effects, activity → SUCCESS
    found, code 11  → one gas-recovery cycle (re-prepare with gasUsed × multiplier; prior tx is terminal on chain)
    found, code N   → FAILED (on-chain);  not found past deadline → FAILED (EXPIRED)

Frontend ◀── react-query polls GET /v1/activities (5 s while anything is pending, 45 s idle)
             status diff → toast + invalidate related queries
             bell, badge, /activities page; each item deep-links
```

The producer runs in the REST app and the workers in the existing background-jobs app, which already share the pg-boss schema. Enqueueing a job already joins the ambient database transaction, so the outbox insert is one transaction with no new infrastructure.

### B. Data model

Two tables rather than one with nullable columns. The execution record is a hot, locked, mutating state machine; the feed is a hot, paged, mostly-append read model. Separating them keeps the worker's `SELECT ... FOR UPDATE` writes from contending with the bell's polling reads, keeps alert rows free of action-only columns, and gives alerts a uniform append target later with no schema change.

**`tx_action`**: the execution state machine and the source of truth for idempotency.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid, primary key | Equals the activity id |
| `userId` | uuid → users, cascade | Owner and scoping |
| `walletId` | integer → user wallets | Derived wallet used to sign |
| `type` | enum | `CREATE_DEPLOYMENT`, `CREATE_LEASE`, `UPDATE_DEPLOYMENT`, `CLOSE_DEPLOYMENT`, `DEPOSIT_DEPLOYMENT`, `RECLAIM_ESCROW`, `OTHER`; intent-level, passed by the caller |
| `status` | enum | `QUEUED`, `VALIDATING`, `BROADCASTING`, `BROADCAST`, `CONFIRMING`, `CONFIRMED`, `FAILED` |
| `messages` | jsonb | Base64 `{ typeUrl, value }[]` as received |
| `signedTx` | text, nullable | Base64 signed `TxRaw`, persisted at prepare, before any broadcast. A re-broadcast only ever resends these exact bytes |
| `deadlineAt` | timestamp, nullable | The signer's returned `timeoutTimestamp`. Set at prepare, not at enqueue, so a queue backlog cannot expire an action before it is broadcast |
| `gas` | integer, nullable | From prepare; replayed only in the gas-recovery cycle |
| `txHash` | varchar, nullable, unique | Known at prepare, before broadcast |
| `code`, `rawLog` | integer, text | On-chain result |
| `errorCode` | varchar, nullable | `INSUFFICIENT_FEE`, `TRIAL_LIMIT`, `EXPIRED`, `ONCHAIN_ERROR`, ... |
| `broadcastJobId`, `confirmJobId` | varchar | pg-boss job ids |
| `attemptCount` | integer | Prepare and broadcast cycles; gas recovery caps it at 2 |
| `createdAt`, `updatedAt` | timestamp | |

Indexes on `userId`, `status`, unique `txHash`, `(userId, createdAt, id)`, and a partial index over the non-terminal statuses for the reaper.

**`activity`**: the unified feed read model, shaped after the existing alert table.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | |
| `userId` | uuid → users, cascade | |
| `kind` | enum | `ACTION` or `ALERT` |
| `category` | varchar | Frontend grouping and icon |
| `status` | enum, nullable | `PENDING`, `SUCCESS`, `ERROR`, `INFO`; projected from the action status, `INFO` or null for alerts |
| `severity` | enum | `INFO`, `SUCCESS`, `WARNING`, `ERROR`; toast styling |
| `title`, `body` | text | Human-readable |
| `link` | text, nullable | Server fallback; the frontend overrides it |
| `meta` | jsonb, nullable | `{ dseq?, dseqs?, gseq?, oseq?, txHash?, amountUdenom?, denom?, alertId? }` |
| `actionId` | uuid → `tx_action`, nullable | Set when `kind = ACTION` |
| `source` | varchar | `api` or `notifications` |
| `dedupeKey` | varchar, nullable | Unique per `(userId, dedupeKey)`; idempotent alert writes |
| `seenAt`, `readAt` | timestamp, nullable | Badge and opened state |
| `createdAt`, `updatedAt` | timestamp | |

Indexes on `(userId, createdAt, id)` for the feed page, `(userId, seenAt)` for the unseen count, `actionId`, unique `(userId, dedupeKey)`, and a GIN index on `meta`.

Both repositories extend the shared base repository and are row-scoped through the existing ability layer, with two new subjects scoped to the owning user. Workers run with an empty ability and call repositories unscoped, as the existing reload-check handler does; HTTP paths are always scoped.

### C. Status lifecycle

`QUEUED → VALIDATING → BROADCASTING → BROADCAST → CONFIRMING → CONFIRMED`, with `→ FAILED` from validation or a deterministic pre-mempool rejection. An ambiguous broadcast outcome (timeout, 5xx) leaves the row in `BROADCASTING`; the confirm job resolves it by polling the persisted hash, so no separate reconcile state or job exists. On-chain code 11 (out of gas) triggers at most one gas-recovery cycle: re-prepare with the actual `gasUsed` times a multiplier, and the new hash replaces the old. That is safe because the prior transaction is terminal on chain. The activity projection collapses every non-terminal status to `PENDING`, `CONFIRMED` with code 0 to `SUCCESS`, and any `FAILED` to `ERROR`. Each transition updates both rows so the poll shows live progress.

### D. Jobs

**`TxBroadcastJob`**, registered with `retryLimit: 0`. Loads and locks the row, requires `status == QUEUED` (otherwise a no-op), re-validates, then runs two phases: signer prepare (sign only) → persist `{ txHash, gas, signedTx, deadlineAt }` and `status = BROADCASTING` → signer broadcast of the stored bytes → `status = BROADCAST` → enqueue the confirm job. A deterministic pre-mempool failure becomes `FAILED`. An ambiguous failure leaves `BROADCASTING` and still enqueues confirm; the handler never rethrows, so the queue never blindly retries a broadcast.

**`TxConfirmJob`**, retryable and idempotent. Polls `getTx(persisted hash)` until found or the deadline passes. Found with code 0: `CONFIRMED`, then the post-transaction effects that today run inline in the managed signer service (trial lease created event, alert enablement, wallet limit refresh, immediate funding) fire idempotently. Found with code 11: the gas-recovery cycle. Not found past the deadline: `FAILED (EXPIRED)`, which is unconditionally safe because the exact hash was checked. It may re-broadcast the same stored bytes once if the transaction is not found mid-window; the chain deduplicates identical hashes.

**Reapers**, self re-arming like the existing reload check. A row stuck in `QUEUED` past its creation plus a grace period is re-enqueued for broadcast (safe: `BROADCASTING` is persisted before any broadcast call, so `QUEUED` proves nothing was sent). A row stuck in `BROADCASTING`, `BROADCAST` or `CONFIRMING` gets confirm-by-hash re-enqueued. A row past its deadline plus grace becomes `FAILED (EXPIRED)`. A retention reaper purges activity rows and terminal action rows older than 90 days.

The managed signer service's execute method is refactored into callable steps: validation (reused at enqueue for an instant `402` and in the worker as the source of truth), the two-phase broadcast, and the post-confirm effects, which move to the confirm handler. The job queue gains per-queue `retryLimit` and `retryBackoff` on the handler definition; today every queue shares a hard-coded retry limit.

### E. Transaction signer

Additive endpoints; the existing combined sign-and-broadcast endpoint stays for the legacy synchronous path until cleanup.

- `POST /v1/tx/derived:prepare`: sign only, no broadcast. Returns `{ signedTxBytes, txHash, gas, timeoutTimestamp }`. Accepts an optional pinned `gas` for the gas-recovery cycle. The signer computes the unordered-transaction TTL and returns it; the API persists it as the deadline. Signed bytes are stored and never re-derived, so byte stability across calls is not assumed anywhere.
- `POST /v1/tx/derived:broadcast`: accepts `signedTxBytes`, runs a synchronous broadcast, returns immediately.
- `getTx(hash)` for the confirm handler.

The API's signer client gains matching methods, and its timeout for these fast calls drops from 60 seconds to about 15.

### F. API

- `POST /v1/tx` is unchanged by default: today's `{ code, transactionHash, rawLog }` body, any authentication method. With `?async=true` it returns `202 { activityId, status: "QUEUED" }` after the outbox enqueue; the instant `402` from validation still applies. API keys may use it for scripted bulk operations and poll the activity. An optional server-side feature flag on honouring `?async` serves as an operational kill switch.
- New feed endpoints, ability-guarded and row-scoped: `GET /v1/activities` (cursor-paged, filter by kind and status), `GET /v1/activities/summary` returning the unseen count, `POST /v1/activities/seen` with ids or a `before` cursor, `PATCH /v1/activities/:id` for `readAt`, and `GET /v1/activities/:id`.
- New internal endpoint `POST /internal/activities`, behind the private middleware, reading the user id from a header. This is the alert sink; it upserts on `(userId, dedupeKey)`.

Once the engine has been hardened (section K, milestone 9), the synchronous default is served by the same engine: enqueue, wait a bounded time (about 60 seconds) on the confirm stage, and degrade to `{ activityId }` past the window. The response shape does not change. Until then the legacy direct path serves it.

### G. Frontend contract

The wallet-context function `signAndBroadcastTx` is renamed to `startTx` rather than silently changing its return type: every call site checks the result's truthiness, and an object is always truthy, so a silent change would turn failures into apparent successes.

```
startTx(msgs, { type, meta }): Promise<{ activityId: string; dseq?: string }>
  always calls POST /v1/tx?async=true
  resolves on 202
  rejects on 402 → existing add-credits snackbar; on 400/5xx → existing error snackbar
```

Callers pass `type` and `meta`; nothing parses title strings. Terminal toasts and query invalidation move into the polling provider. The loading state machine and the modal are deleted at cleanup. During rollout each migrated call site branches on the feature flag, so both entry points coexist until the legacy path is removed.

### H. Polling provider

An `ActivityPollingProvider`, modelled on the existing payment-polling provider and mounted next to it inside the wallet provider. One query over `GET /v1/activities` with a refetch interval of 5 seconds while any item is non-terminal and 45 seconds when idle (to catch alert firings), and no background refetch. It keeps a map of the last seen status per id; on each fetch it diffs, and for any id entering a terminal state fires the success or error toast and the invalidation for that type. It exposes the items, the unseen count, `markSeen`, `markAllSeen`, a `pendingByDseq` map, and `registerOptimistic`, which prepends a synthetic pending item on enqueue so the bell reacts before the first poll.

Three maps keyed by action type carry the per-action knowledge:

- `linkFor(item)`: create → the new-deployment page at the create-leases step with the `dseq`; lease, update, deposit, reclaim → deployment details; close → the deployment list; alert → deployment details on the alerts tab.
- `invalidationFor(type)`: for example close invalidates the deployment list, detail, leases and balances (looping `meta.dseqs` for a bulk close); deposit invalidates detail and balances.
- `copyFor(type)`: the started, success and failure strings, replacing the scattered constants used by the modal today.

### I. Notification center

A bell icon button with an unseen-count badge (capped at "9+"); opening it marks everything seen. A dropdown shows the eight most recent items, a "See all activities" footer and an empty state. Each item shows an icon, title and body, relative time, and a status badge generalised from the existing alert status component; a non-terminal item shows a spinner, a failed one shows the error with "Contact support" and a "Try again" link that deep-links to re-initiate the action and never replays the chain transaction blindly. Clicking marks the item seen and navigates via `linkFor`. The bell sits in the top navigation before the account menu, and in the legacy navigation for parity while both exist. An `/activities` page mirrors the alerts list pages with filters for all, in progress and failed, and by type. Actions started by API keys appear in the feed once the sync default rides the engine; that is intended, since it is the user's activity.

`pendingByDseq` drives "Closing...", "Depositing..." and "Updating..." badges on list rows and the detail top bar and disables conflicting buttons. Cross-tab consistency comes for free because every tab polls.

### J. Guided create, bid and lease

The deployment sequence number is generated client-side before broadcast, so the resume URL exists at enqueue time. The create activity's link is `?step=create-leases&dseq=...`, which the new-deployment container already reconstructs, so resume after refresh costs nothing extra. On enqueue the manifest is persisted locally and the router moves to the create-leases step; while the create activity is non-terminal the page shows "Deployment is being created on-chain...", then falls through to the existing waiting-for-bids UI. Each transaction is its own action and feed row; the frontend glues the steps and bid selection stays client-side.

The manifest push to the provider after a lease or update confirms is not a chain action. It starts page-local, driven by the confirmed status, and is then centralised into a `ManifestSyncProvider`. That also fixes a latent bug where closing the tab between confirmation and the manifest send leaves a lease with no manifest. A true server-side manifest send becomes possible once [AEP-92](../aep-92) moves stored definitions into the database.

Call-site migration order: reclaim escrow first as the tracer (already-dead deployments, zero live-workload risk); then the fire-and-forget actions (three close buttons, close-in-create, deposit), which navigate or clear on enqueue with refresh centralised on confirmation; then the guided flows (create, onboarding, create-lease, update), which navigate on enqueue, auto-advance on the watched activity, and resume via URL. Ten call sites across nine files; there are no certificate, fee-grant or authorization broadcasts in the web app. The dead mint path that predates managed-wallet-only is deleted along the way.

### K. Idempotency

Two root causes make naive retries dangerous. First, the signer sets the unordered-transaction `timeoutTimestamp` at sign time (default TTL 30 seconds) and the chain deduplicates unordered transactions by hash within that TTL. A blind re-sign behind a retrying queue produces a new hash the chain cannot deduplicate, and the transaction executes twice. Second, if a worker crashes after broadcasting but before persisting the hash, the transaction may have landed while the row knows no hash; marking it failed invites the user to retry something that already executed, which is a double spend through the UI.

The defence is layered:

1. **Outbox enqueue.** Action row, activity row and job commit in one transaction. No job without a row, no row without a job.
2. **Persist before broadcast.** Prepare signs without broadcasting; hash, gas, signed bytes and deadline are persisted before the broadcast call. The exact hash is therefore known for any row that could have reached the mempool, so every ambiguous state resolves by looking up that hash: adopt if landed, expire if not. No sender-window scans, no on-chain memo, no hash re-derivation.
3. **Split broadcast and confirm, with no retries on broadcast.** All retry-prone waiting lives in the idempotent confirm poll; the only unretried step is prepare, persist, broadcast, which is sub-second.
4. **Status guard and row lock.** Concurrent or duplicate workers cannot both proceed past `QUEUED`.
5. **Definitive expiry.** Once the deadline passes the transaction is permanently unincludable, so "not found past deadline" for the exact persisted hash is a safe terminal failure, and the user's retry with a fresh action is unconditionally safe.
6. **Re-broadcast means the same bytes.** Any resend uses the stored signed transaction; the identical hash is deduplicated by the chain. The only path that creates a new hash is the explicit gas-recovery cycle after a definitive on-chain out-of-gas failure, when the prior transaction is terminal.
7. **Idempotent post-transaction effects.** Domain event publication is gated on the confirm handler's own status transition, and downstream jobs are keyed or singleton.

Fault-injection tests that crash the worker between prepare, persist, broadcast and confirm must show neither a double broadcast nor a false failure before the flag is ramped.

### L. Alert integration

The notifications service has one dispatch seam, the notification router, which today sends email only. Its payload carries only a summary and a description, so it is extended with `alertId` and a `dedupeKey` (for example the alert id and block height), populated in the alert-condition services where both are in scope. An unconditional in-app sink posts each fired alert to the API's internal activities endpoint (a new notifications-to-API client with a shared secret and a user-id header, the reverse of today's API-to-notifications call), upserting on `(userId, dedupeKey)`. It is best effort, retry then swallow, so a feed outage never blocks email. The feed table and endpoints ship first; this phase only adds a writer.

### M. Milestones

| # | Milestone | Apps | Depends on |
|---|---|---|---|
| 0 | Per-queue `retryLimit` and `retryBackoff` on job handlers | api | |
| 1 | Data model, repositories, ability subjects, migration, feed read API, generated client | api | 0 |
| 2 | Notification center shell, flagged and read-only: polling provider, bell, dropdown, activities page | deploy-web | 1 |
| 3 | Signer prepare/broadcast split and `getTx`; API client | tx-signer, api | (parallel) |
| 4 | Async happy path behind `?async=true`: outbox enqueue, broadcast and confirm jobs, signer service refactor, `startTx`, completion engine; migrate reclaim as the tracer | api, deploy-web | 1, 2, 3 |
| 5 | Full lifecycle, post-transaction effects moved to confirm, live status, gas recovery | api | 4 |
| 6 | Idempotency hardening: reapers and fault-injection tests | api | 5 |
| 7 | Migrate fire-and-forget actions, optimistic pending badges, delete dead mint code | deploy-web | 4 |
| 8 | Migrate guided flows, resume via URL, centralise the manifest push | deploy-web | 7 |
| 9 | Serve the sync default through the engine façade | api | 6 |
| 10 | Alert fold-in: payload extension and in-app sink | notifications, api | 1 |
| 11 | Cleanup and rollout: delete the legacy path, the combined signer endpoint and the modal; ramp the flag; dashboards | all | 8, 9 |

## Rationale

**Opt-in async, sync default forever.** The value of async is UI-centric; a headless client has no bell and wants to block. An explicit parameter cannot break anyone at deploy time, stale cached bundles keep working through every phase, and there is no authentication-type sniffing to get wrong. One execution engine, two response modes, one record either way.

**Two tables.** Explained in section B: lock contention and shape. The alternative of one table with nullable action columns would have every alert row carry a state machine it does not use and every bell poll compete with worker locks.

**Persist before broadcast, rather than reconcile after.** Reconciliation by scanning a sender's recent transactions, or by stamping a memo, is heuristic and lags the chain. Knowing the exact hash before anything is sent turns every ambiguous case into a lookup.

**No retries on broadcast, all retries on confirm.** The one step that can double-execute is the one that is not retried; everything that can safely repeat is.

**Rename, do not overload.** Changing the return type of a function whose callers test for truthiness would compile and silently report success on failure.

**Polling over WebSockets.** Every existing live surface in the app polls with react-query; adding a socket layer for one feature would be new infrastructure for a marginal latency gain.

**Client-side bid selection stays.** A server-side saga that picks bids would change what users get charged without them looking. The guided flow glues the steps and leaves the choice on screen.

**Ninety-day retention.** Long enough to answer "what happened last month", short enough that the feed table does not become an archive.

## Backward Compatibility

- `POST /v1/tx` without `?async=true` returns exactly today's body for every caller, in every phase, with no planned flip. When the engine façade takes over in milestone 9 the response shape is unchanged; only its production changes.
- The combined sign-and-broadcast signer endpoint remains until milestone 11 so the legacy path can serve the sync default in the meantime.
- New endpoints are additive. The feed and its tables are guarded by the ability layer and visible only to their owner.
- The frontend feature flag lets migrated call sites branch between the legacy modal flow and `startTx`, so a partial rollout leaves unmigrated actions exactly as they are.
- API-key clients that opt in receive a `202` with an activity id and must poll; clients that do not opt in notice nothing.
- Alerts continue to send email exactly as today; the in-app sink is an additional writer and its failure never blocks email.
- Deposit is migrated unless AEP-91 has removed the deposit UI first; either way it is off the legacy path by cleanup, which is never held hostage to that project.

## Test Cases

- Backend unit and functional: every state transition; a simulated crash between prepare, persist, broadcast and confirm yields neither a double broadcast nor a false failure and adopts a landed transaction by hash; `FAILED (EXPIRED)` only after the deadline; outbox atomicity (no row without job, no job without row); ability row-scoping on every feed endpoint; the gas-recovery cycle re-prepares at most once and replaces the hash.
- Backend integration: the migration applies cleanly; repositories against a real database.
- Local API smoke: boot the REST interface and a background-jobs worker on scratch databases; `POST /v1/tx?async=true` with a bearer token returns `202` and the rows transition as the jobs run; `POST /v1/tx` with an API key and no parameter returns the classic body unchanged.
- Frontend: the polling provider turns a status diff into a toast plus invalidation; `startTx` rejects on `402` and shows the add-credits snackbar; bell, badge and seen state; a close from the list is non-blocking, shows "Closing..." and resolves on poll; a refresh mid-create re-attaches through the activity link and auto-advances to bids.
- Pre-push in every affected app: `npm test`, `npm run lint -- --quiet`, `npx tsc --noEmit`.

## Implementations

Not started. The reference implementation lands in `akash-network/console` following the milestones in section M, each as one or more small PRs, with milestone 4 (reclaim as the tracer) as the first user-visible change behind the flag. The AEP moves to Last Call once milestone 6 has proven the idempotency properties with fault-injection tests and milestone 7 has migrated the fire-and-forget actions.

## Security Considerations

- **Double spend through retries.** The whole of section K exists for this. The two failure modes, a re-signed duplicate that the chain cannot deduplicate and a false failure that invites a user retry, are both closed by knowing the hash before broadcast and never re-signing except after a definitive on-chain failure.
- **Custody unchanged.** Signing stays in the transaction signer with the same derived wallets and the same funding-wallet grants. The API stores signed bytes, which are already public once broadcast, and never key material.
- **Validation twice.** Balance, fee-grant and trial rules are checked at enqueue, for an immediate `402`, and again in the worker as the source of truth, so a queued action cannot bypass a limit that changed in between.
- **Row scoping.** Feed endpoints filter by the requesting user through the ability layer. Workers run unscoped and never take user input for the row they act on.
- **Internal sink.** The notifications-to-API endpoint sits behind the private middleware with a shared secret and takes the user id from a header, mirroring the existing API-to-notifications trust contract. The header guard that rejects client-supplied identity headers must cover it.
- **"Try again" never replays.** A failed item's retry link re-initiates the action through the UI, creating a fresh action with fresh validation, and never re-broadcasts stored bytes on the user's behalf.
- **Kill switch.** A server-side flag can stop honouring `?async` without a deploy; the default sync path is unaffected by it.

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
