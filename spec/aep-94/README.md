---
aep: 94
title: "Console: Organizations and Projects"
description: "Organization-owned billing and managed wallet, with projects as the grouping and permission boundary for shared team workloads"
author: Maxime Beauchamp (@baktun14)
status: Draft
type: Standard
category: Interface
created: 2026-09-04
estimated-completion: 2027-03-31
discussions-to: https://github.com/orgs/akash-network/discussions
roadmap: major
requires: 63, 68, 84
---

## Abstract

Akash Console is single-tenant per user. Every owned row hangs off the user, and a user has exactly one managed wallet. Because a deployment's on-chain identity is `(owner_address, dseq)` and that address is the managed wallet, the wallet is the tenancy boundary: two people cannot share a workload, and billing is attached to a person rather than a company.

This AEP introduces organizations, a team boundary that owns billing and the managed wallet, and projects, a grouping and permission boundary inside an organization. After the change a company pays once, its engineers share the workloads they deploy under one on-chain address, and those workloads are filed into projects that individual members can be granted or denied access to. Every user gets a personal organization, invisible until they create or join a team, so nothing changes for someone working alone.

The scope is `apps/api`, `apps/deploy-web` and `apps/notifications` in `akash-network/console`. There are no chain changes.

## Motivation

Every authorization rule in the Console API is `userId = <current user>`, and the hard constraint underneath is that `user_wallets.user_id` is unique. Three consequences follow:

- Two people cannot see or manage the same workload. A teammate cannot restart a service their colleague deployed.
- Billing is attached to a person. A company cannot pay for its engineers' usage, and an engineer leaving takes the payment relationship with them.
- There is no way to separate staging from production, or one client's workloads from another's, beyond naming conventions held in browser local storage.

[AEP-63](../aep-63) gave managed-wallet users an API, [AEP-68](../aep-68) gave them billing and usage, and [AEP-84](../aep-84) made Console managed-wallet only. All three assume one person per wallet. Teams are the next unit of adoption and the current model has no place to put them.

## Specification

### Scope

In scope: data model and backfill; a rewrite of the ability engine; default-on tenant scoping at the data layer; active-organization resolution; moving the wallet and the Stripe customer to the organization; organization and project management UI; invitations; deployments filed into projects and listed through the API; organization-scoped API keys; trial and first-purchase-bonus caps; notification scoping; per-project spend attribution and budgets.

Out of scope: chain changes; one wallet per project (see Rationale); per-key permissions narrower than the owner's role (a follow-up); SSO and directory sync.

### Concepts

**Organization.** The billing and ownership boundary. Owns the managed wallet, the Stripe customer, payment methods, transactions and credit. One wallet per organization, so all members deploy under one on-chain owner address.

**Project.** A grouping inside an organization and a permission boundary. Every deployment belongs to exactly one project. Every organization has a `Default` project that cannot be deleted.

**Personal organization.** Auto-created for every user, containing only them. It preserves today's behaviour exactly and stays invisible in the UI until the user creates or joins a team.

**Roles.**

| Organization role | Can |
|---|---|
| `OWNER` | Everything, including billing and deleting the organization |
| `ADMIN` | Everything except billing and deleting the organization; cannot grant or remove `OWNER` |
| `MEMBER` | Deploy and manage within granted projects |
| `BILLING` | Payment methods, transactions, invoices; no deployment access |
| `VIEWER` | Read-only within granted projects |

`OWNER` and `ADMIN` implicitly reach every project. `MEMBER` and `VIEWER` reach only projects they are explicitly granted, through project roles `PROJECT_ADMIN`, `PROJECT_MEMBER` and `PROJECT_VIEWER`. An organization always has at least one `OWNER`.

### What this is not

The words "organization" and "project" carry expectations from AWS and GCP that this design does not meet, so three boundaries are stated plainly and must be reflected in product copy.

**A project is not a confidentiality boundary.** All of an organization's deployments live under one public on-chain owner address. Anyone, member or not, can enumerate them from the chain. What the project boundary enforces is mutation (closing, funding and updating a deployment all route through the API) and what the app shows. It does not make a deployment's existence secret.

**A project is not a spend boundary.** The on-chain deposit authorization is organization-wide and cannot carry a project id. Any member who can deploy in any project can spend the organization's entire balance, including on projects they cannot see. Per-project budgets are a cooperative guardrail against accidental overspend and a basis for chargeback, not isolation. A true spend boundary would require one wallet per project, which is the trade consciously made in exchange for shared workloads and a single credit pool.

**An organization is not a security boundary against its own admins.** An API key carries its owner's full organization role, so an `OWNER`'s key can delete the organization. Scoped keys are follow-up work.

### A. Data model

New tables:

| Table | Purpose |
|---|---|
| `organizations` | `name`, `slug`, `type` (personal or team), `stripe_customer_id`, `created_by_user_id`, `deleted_at` |
| `organization_members` | `(organization_id, user_id)` unique, `role` |
| `organization_invitations` | `email`, `role`, `project_grants`, `token_hash`, `status`, `expires_at` |
| `projects` | `(organization_id, slug)` unique, `is_default`, `deleted_at` |
| `project_members` | `(project_id, user_id)` unique, `role` |
| `trial_grants` | One row per granted free trial; unique on `user_id` and on `organization_id` |

Three structural decisions:

- `organizations` and `projects` are soft-deleted. An organization owns a funded wallet whose on-chain address is derived from a database row; a cascading hard delete would orphan real funds.
- Composite foreign keys. Every project-bearing table carries `(organization_id, project_id) → projects(organization_id, id)`, which makes "a row whose project belongs to a different organization" impossible at the database level rather than merely unlikely at the query level. Likewise `project_members(organization_id, user_id) → organization_members(organization_id, user_id) ON DELETE CASCADE`, so removing someone from an organization drops all their project grants.
- A partial unique index on `created_by_user_id WHERE type = 'PERSONAL'` is the database-level guarantee that every user has exactly one personal organization, and it is what makes the backfill idempotent.

Existing tables gain `organization_id` (and `project_id` where relevant): `user_wallets`, `wallet_settings`, `payment_methods`, `stripe_transactions`, `deployment_settings`, `api_keys`, `template`. The Stripe customer id moves from the user to the organization.

The existing `user_id` columns are kept but demoted to an actor and audit role: who created this, not who owns this. Dropping them is irreversible and cannot be rolled back mid-deploy; keeping them is free and preserves history. They are removed from every authorization condition and repository filter.

**Invariants.**

- `user_wallets.id` is the BIP-44 derivation index. The transaction signer derives the signing key from it and historical address resolution depends on it. The migration is therefore a pure column addition plus an `UPDATE`: no row recreation, no re-keying, no sequence reset. The signing service requires zero changes, and a CI guard asserts `(count, min(id), max(id))` on `user_wallets` is unchanged across the migration chain.
- The `ON DELETE CASCADE` from users to wallets and deployment settings must go. Under organizations, deleting a member would otherwise destroy the only record of the index-to-address mapping and silently disable automatic funding on live deployments: a billing incident, not a UX bug.
- Stripe idempotency key formulas do not change. Keys are opaque namespaces; changing one mid-flight makes a client retry across a deploy boundary produce a different key and therefore a duplicate charge. New organizations get a new namespace instead.
- The payer is never derived from ambient context. A refund for a purchase made in organization A must debit organization A, read off the transaction row, even if the person who made it has since moved to organization B.

### B. Authorization

Three layers, in order of trust:

1. **Tenant predicate.** A new organization-scoped repository base class ANDs `organization_id = <active organization>` into every query it builds, unconditionally. Today's ability filtering is opt-in per call site, which means a repository method that forgets to opt in returns everything. This layer is what actually prevents cross-organization leakage. Writes are attributed to the active organization automatically, and a write naming a different organization is rejected. Deliberately unscoped queries are individually marked so the full set of exceptions can be listed, and a test fails if a table carrying an organization is not covered. Three existing gaps close with it: bulk updates by id apply no access filtering at all, one repository bypasses its own access check when creating rows, and templates can be read by id by any user, including private ones.
2. **Ability rules.** Role → action → subject, with project scoping.
3. **Route guards.** Coarse action gates at the controller boundary.

**Active organization resolution.** Per request, in order: the API key's bound organization → the `x-organization-id` header → the user's default organization, if still a member → their personal organization. An API key combined with a differing header is rejected rather than silently resolved either way. Non-membership returns `403`. An optional `x-project-id` header may narrow the scope to one project; it can never widen it. Membership is deliberately not cached to begin with, since a cache here means a removed member keeps access until it expires. Background jobs and CLI commands run outside any request and keep their current unfiltered behaviour.

**Ability rules.** The current rule table is compiled by string-templating a JSON blob. That is replaced by typed rule factories, for three reasons:

- Project scoping needs `projectId IN (...)`. An array interpolated into a JSON string literal does not error; it silently becomes the string `"id1,id2,id3"` and yields a predicate that matches nothing.
- The translation layer from rules to queries has no `in` operator at all, so such a rule would throw at query time.
- The current approach carries a latent production bug independent of this work: a user whose email contains a double quote produces malformed JSON and a `500` on every request.

Rules become:

```ts
const inOrg = c => ({ organizationId: c.organizationId });
const inScope = c =>
  c.projectScope.kind === "ALL"
    ? { organizationId: c.organizationId }
    : { organizationId: c.organizationId, projectId: { $in: c.projectScope.ids } };
```

The project scope is a discriminated union rather than an optional array, so that "admin, all projects" and "context not set" cannot be confused; otherwise the data layer fails open. Wallet, billing and payment subjects scope to the organization; deployment, template, alert and notification subjects scope to the projects the member can reach. A rule naming a field that does not exist on its table fails loudly. Feature-flag changes to rules take effect without a restart. The notifications service carries a near-identical copy of the rule-to-query layer, so it is extracted to a shared package rather than fixed in one place.

Two rules SQL cannot express are enforced in services, both with a locking read: an `ADMIN` may not grant or remove `OWNER`, and the last `OWNER` may not be demoted or removed, including when two admins attempt it at the same time.

### C. Deployments and projects

A deployment has no database record today; the list is read straight from the chain by owner address, and deployment names live in browser storage. With one owner address per organization that no longer works: the API must decide which deployments a given member may see.

The per-deployment settings record becomes the authoritative per-deployment row, gaining `organization_id`, `project_id`, `owner_address`, `name`, `created_height` and `state`. Rows are created eagerly rather than lazily; a deployment with no project is invisible to every restricted member while still billing the organization. Three layers guarantee a row exists:

1. **Write-ahead at creation.** The deployment sequence number is generated before the transaction is broadcast, so the record is written first and marked active on success. Writing after the broadcast would mean a deployment that exists on chain but not in the database, which would then reconcile into the default project, invisible to the very member who created it.
2. **A safety net at the signing chokepoint**, catching any path that bypasses the primary flow.
3. **Reconciliation**, periodically diffing the on-chain set against the database, filing strays into the default project and closing records whose deployments have ended.

Listing becomes database-first: an ability-filtered query with server-side paging, search by name and a project filter, hydrating only the current page from the chain. The alternative, fetching everything from the chain and filtering client-side, cannot paginate a project and would fetch hundreds of deployments to show a member the three they can see. Each row shows its project. A deployment the user has no access to is indistinguishable from one that does not exist. Opening a link to a deployment in another of the user's organizations switches to it.

Owners and admins can move a deployment between projects. A member deploying into a project they cannot see is refused. A project holding deployments cannot be deleted until they are moved or closed. Deployment names move from browser storage onto the record and are migrated once from local storage the first time a user loads the list; search matches on the stored name. An organization with a single project does not surface project UI where it would be noise.

### D. Organization lifecycle

**Create.** A user creates an organization from the account menu, giving it a name. They become its `OWNER`, it gets a default project and a wallet, and they are switched into it, landing on the members screen ready to invite. The new wallet is provisioned immediately and is never a trial wallet. The number of organizations one person can create is rate-limited (see F).

**Invite.** An `OWNER` or `ADMIN` invites by email, choosing an organization role and optionally project grants. An `ADMIN` cannot invite as `OWNER`; a `MEMBER` or `VIEWER` cannot invite. Pending invitations are listed with members and can be revoked; they expire. Re-inviting the same address does not create a duplicate. The link is a secret; only its hash is stored, so it cannot be reconstructed from anything on the server. An organization still on a free trial cannot invite.

**Accept.** Opening the link shows who invited them and to which organization before asking them to sign in or sign up. After authenticating they return to the invitation, join with the intended role and project access, and land in the organization. A signed-in user whose email differs from the invited address must confirm rather than silently joining as the wrong identity. An expired or revoked invitation explains itself. Accepting twice, or two people racing one link, cannot produce duplicate membership. An invitee with no wallet of their own is not diverted into first-deployment onboarding; joining an organization that already deploys makes them onboarded, since they inherit its wallet.

**Members.** A members screen lists everyone with their role. Owners and admins change roles and remove members; members and viewers can see the list. A removed member immediately loses access to the organization and every project in it, and their deployments remain with the organization and keep running, because they belong to the organization's wallet.

**Projects.** Owners and admins create, rename and delete projects and grant or revoke project access with a project role. Revocation takes effect immediately. A member with no grants sees an explanation rather than an empty list that looks like a bug.

### E. Billing

The credit layer itself is unchanged: balances, spending authorization and signing are already keyed by wallet or address, not by user. The change is routing.

- The Stripe customer moves from the user to the organization, including all webhook reverse lookups, with a temporary fallback to the legacy user link so that webhooks already in flight during the migration still resolve (Stripe retries for three days).
- Top-ups, refunds, coupons, auto-reload, transaction history and CSV export become organization-scoped while recording the acting member separately.
- Wallet provisioning moves from user registration to organization creation, staying idempotent so existing self-healing paths still work.
- Analytics keeps the acting user as the identity and adds the organization as a property. Using an organization id as the analytics identity would silently fork every user's identity graph.
- Members joining a team do not bring their payment methods; cards belong to the organization that attached them.
- Reporting that counts "paying users" by counting non-trial wallets now counts paying organizations, and is renamed accordingly.

### F. Trial and bonus abuse

This is the highest-risk consequence of the change and must ship in the same release as organization creation.

Today the ceiling is one free trial per verified email, held up by the unique wallet per user. Once users can create organizations it becomes one trial per organization, unbounded per person. The existing fingerprint checks all exclude by user id, so against many organizations owned by one user they are blind. The same hole applies to the first-purchase bonus: 10% of a first top-up up to $100, per organization, which is real money.

The mechanism is the `trial_grants` table, whose two unique indexes are the enforcement: an atomic insert replaces a check-then-act race. It is backfilled from existing activated wallets so current users cannot claim a fresh trial. On top of it, as policy rather than mechanism: trials only on personal organizations (behind a flag, so relaxing it later is a one-line change), owner or admin only, and trial organizations cannot invite members. The first-purchase bonus gains a matching per-person cap, backed by a partial unique index, which is deliberately not cleared on refund. Rejected claims are recorded so the abuse rate is visible. Organization creation is rate-limited per user, which caps every organization-multiplied abuse vector at once, including ones not yet thought of.

### G. API keys

Every API key is bound to one organization at creation, and optionally to one project for CI use. A key acts only within that organization regardless of any header the caller supplies. Existing keys are bound to their owner's personal organization. A member's key cannot exceed what that member can do. The key list shows each key's organization and project. Per-key permissions narrower than the owner's role are left as a column that is cheap to add now and a breaking change later.

### H. Frontend

The active organization is held in a cookie and translated into an `x-organization-id` header at the API proxy. A `?org=<slug>` query parameter acts as a one-shot override that writes the cookie and is stripped from the URL, which preserves deep-link sharing without restructuring the router. Organization administration pages carry the slug in the path; application pages keep their flat URLs.

Two details that are easy to get wrong:

- Server-side rendering calls the API directly rather than through the proxy, so without a per-request client it would silently read the user's personal organization while the browser reads the team's. The fix is a request-scoped API client; mutating a shared client's headers would leak one request's organization into another's response.
- Switching organizations clears the query cache wholesale. The active organization travels in a header the client never sees, so it is not part of any cache key, and selective invalidation would mean hand-maintaining a list of every organization-sensitive endpoint.

A switcher in the header shows the current organization when the user has more than one and is hidden otherwise, in both header variants and on mobile. No data from the previous organization is shown after a switch, even briefly. Two tabs share one selection; that is accepted initially. The personal organization stays invisible until the user has a team, so an existing user's interface is unchanged apart from a "Create organization" entry in the account menu.

Two one-line omissions would break the feature outright: the onboarding gate must allow the invitation and organization routes, or an invitee without a wallet is redirected into the first-deployment funnel and the invite link is dead; and a newly created team must be provisioned a non-trial wallet immediately, or the same gate redirects its creator to onboarding.

### I. Notifications service

The notifications service is a separate database with no foreign keys to the main one; headers are the entire trust contract. The proxy gains organization, role and project-scope headers, and the guard that rejects client-supplied identity headers must be generated from the same list that mints them. The two lists drifting apart is exactly how a client would forge its own tenancy.

Alerts and notification channels gain organization scoping, and where relevant project scoping, so an alert created by one member is visible to others who can see the project. The "one default channel" constraint re-keys from user to organization, or every member of a team gets their own default and the organization has several. Existing alerts and channels are attributed to their owner's personal organization and keep firing. The backfill cannot be a migration, because that database cannot see users or organizations; it is a one-shot script holding both connections, run as an explicit deploy step between the schema change and the `NOT NULL` change.

### J. Per-project spend and budgets

Spend is reported per project over a date range, reconciling against the organization total with any remainder shown as unassigned rather than dropped. Owners and admins set a monthly budget per project. Exceeding a budget raises an alert first and then blocks new deployments in that project with a clear reason; stopping funding for running deployments is opt-in, never the default. A budget is a rate limiter on new commitments, not a spend cap: deployment funds are deposited in advance and consumed per block, so a project can overrun by whatever it has already deposited. Per-project spend is visible only to members who can see the project. Existing spend reporting is public and keyed by wallet address; per-project reporting must not inherit that, since project names and spend reveal organization structure.

### K. Migration and rollout

Three migrations:

1. **Structure.** New enums, tables and nullable columns on existing tables. Metadata-only; no table rewrite.
2. **Backfill.** Hand-written and idempotent: a personal organization and owner membership per user, a default project per organization, then `organization_id` onto every child table.
3. **Tighten.** Drop the old user-scoped uniques, set `NOT NULL`, add the new uniques and composite foreign keys.

Migrations 1 and 2 are backward compatible and ship a release early. Migration 3 must ship with the application code that requires organization context: in the window between them, an old instance still running would insert rows without an organization and migration 3 would then fail.

Pre-flight checks run before the backfill (duplicate Stripe customer ids, deployment sequence numbers shared across users, orphaned templates, and a baseline of the wallet id range). Post-backfill assertions run before tightening. Two operations cannot live inside a migration: concurrent index builds, which cannot run in a transaction, and `SET NOT NULL` on a large table, which needs the validated-check-constraint approach instead. Both are decided per table from live row counts. Stripe customer metadata sync and historical deployment-to-project attribution are background jobs, not migrations.

### L. Phasing

| Phase | Contents |
|---|---|
| 0 | Data model and backfill; ability engine rewrite; default-on tenant scoping; active-organization resolution; wallet and Stripe customer move to the organization. Nothing user-visible. |
| 1 | Behind a feature flag: create an organization, invite and accept, member and role management, project CRUD and grants, organization switcher, deployments filed into projects, deployment list on the API, deployment names persisted. |
| 1.5 | Must land before full rollout: organization-scoped API keys, trial and bonus caps, project picker in the deploy flow, billing and usage re-labelled to the organization. |
| 2 | Schema tightening; per-project spend attribution and budgets; notification scoping; organization-prefixed application routes; audit log. |

## Rationale

**One wallet per organization, not per project.** A wallet per project would make projects real spend and confidentiality boundaries, at the cost of one credit pool per project, one set of payment methods per project, and workloads that cannot move between projects without changing on-chain owner. Shared workloads and a single balance are what teams asked for; the boundaries that a wallet per project would have given are stated as non-goals instead of being half-implemented.

**A personal organization for everyone.** Backfilling one for every user means the rest of the code has exactly one path, with no nullable-organization branch to carry forever. It also makes the migration a no-op from the user's point of view.

**Default-on scoping at the data layer.** Opt-in filtering is survivable while each user only ever sees their own data through a handful of audited paths. Once one organization's rows sit beside another's, a single forgotten opt-in is a cross-tenant leak, so the safe query has to be the one that requires no thought.

**Typed rule factories.** String-templating rules cannot express set membership without silently producing a broken predicate, and it already breaks for a legal email address. A representation that carries real data fails loudly where the old one failed open.

**Write the deployment row before broadcast.** The default project is admin-visible. A member's deployment that reconciled into it after the fact would vanish from that member's own view. Generating the sequence number first makes the write-ahead possible.

**Soft delete and kept `user_id` columns.** Both are about the irreversibility of the alternative: a hard-deleted organization orphans funds, and a dropped column cannot come back mid-rollback.

**Abuse caps in the same release as creation.** The hole is not theoretical: the bonus pays out up to $100 per organization, and organization creation is otherwise free. Unique indexes are the enforcement because a check-then-act race is exactly what a determined abuser will find.

**Analytics keyed on the person.** Switching the identity to the organization would fork every existing user's history the day the migration ran.

## Backward Compatibility

- A user who never creates or joins an organization sees the product exactly as before. Their personal organization is invisible and their wallet address is unchanged.
- Every existing wallet keeps its exact on-chain address. The migration adds columns and updates rows; it never renumbers, recreates or re-keys a wallet. The signing service needs no change.
- Existing API keys keep working, bound to their owner's personal organization. Requests with no organization header resolve deterministically to the caller's default or personal organization.
- Stripe webhooks in flight during the migration resolve through a temporary fallback to the legacy user link. Idempotency key formulas are unchanged, so no client retry across the deploy boundary produces a second charge.
- Existing alerts and channels keep firing, attributed to their owner's personal organization.
- Users who already used a free trial cannot claim another after the migration, because `trial_grants` is backfilled from activated wallets.
- The `user_id` columns remain on every table as actor attribution; nothing that reads them for audit breaks.
- Any cacheable organization-scoped `GET` must vary on the organization header, or a shared cache would serve one organization's data to another. This is a required change to caching, not an optional one.

## Test Cases

- Data model: the backfill is idempotent (running twice changes nothing); every user has exactly one personal organization and is its `OWNER`; every organization has one default project; the wallet id `(count, min, max)` triple is identical before and after the migration chain.
- Scoping: a query with no explicit filter returns only the active organization's rows; a bulk update cannot touch rows outside it; a write naming another organization is rejected; a user cannot read another user's private template by id; a new organization-bearing table without default scoping fails the coverage test.
- Ability: a member granted two of four projects sees rows from exactly those two; a member with no grants sees nothing; owners and admins reach every project without a grant; a rule naming a nonexistent field fails loudly; a user whose email contains a double quote can use the app.
- Resolution: naming an organization the caller does not belong to returns `403`; an API key with a conflicting header is rejected; a caller who names nothing lands on a deterministic organization; a project header can narrow but never widen.
- Lifecycle: an admin cannot promote to `OWNER`; the last `OWNER` cannot be removed under concurrent attempts; a removed member loses access immediately and their deployments keep running; an invitation cannot be accepted twice or by a race; an invitee with a differing email must confirm; an invitee without a wallet is not sent to onboarding.
- Deployments: the row exists before broadcast; a deployment created by a bypassing path is filed into the default project; a member cannot see, close or fund a deployment in an ungranted project; paging is server-side; an inaccessible deployment reads as not found.
- Billing: a top-up in a team charges the organization's card and credits its wallet; a refund debits the organization that paid even after the buyer left; in-flight webhooks resolve; no duplicate charge from the migration.
- Abuse: a second trial through a new organization is refused; two simultaneous trial claims cannot both succeed; a team organization cannot start a trial; the bonus is granted at most once per person and refund does not restore eligibility; organization creation is rate-limited.
- Frontend: the switcher is hidden with only a personal organization; a switch shows no stale data; a server-rendered page shows the same organization as the client; `?org=` opens the right organization and is stripped from the URL.
- Notifications: a client-supplied identity header is rejected; one default channel per organization; alerts fire for the right organization through the backfill.

## Implementations

Not started. The reference implementation lands in `akash-network/console` following the phases in section L, phase 0 first as a dark release. The AEP moves to Last Call once phase 1 is available behind its flag and the phase 1.5 caps are in place.

## Security Considerations

- **Cross-tenant leakage** is the new failure class this AEP introduces, and section B's default-on tenant predicate is the primary control. Composite foreign keys make a row in the wrong organization unrepresentable; the scoping coverage test makes a forgotten table a test failure rather than a leak.
- **Header trust.** The organization, role and project-scope headers are minted by the proxy and must be rejected when supplied by a client, on both the API and the notifications service, from one shared list. An API key's bound organization always wins over a header, and a conflict is a rejection rather than a choice.
- **Deployment enumeration is public.** Nothing here hides the fact that a deployment exists; the chain publishes every deployment under the organization's address. The project boundary governs mutation and display. Product copy must not imply confidentiality.
- **Spend is organization-wide.** Any member who can deploy anywhere can spend the whole balance. Budgets are cooperative. Owners should grant `MEMBER` with care and treat `VIEWER` as the default for people who do not deploy.
- **API keys carry full role.** An `OWNER`'s key can delete the organization. Until per-key permissions exist, keys for automation should be created by a `MEMBER` bound to a single project.
- **Trial and bonus multiplication** is closed by unique indexes, not by checks, and by the per-user creation rate limit as a backstop for vectors not yet identified.
- **Wallet safety.** The wallet id is the derivation index. The migration is additive only, verified by the id-range guard; any future change that renumbers wallet rows would orphan real funds.
- **Deleting an organization that still holds credit.** Credit is an on-chain grant to an address only that organization's key can spend, and there is no path to refund it to members. Deletion is blocked while a balance or active leases remain.
- **Invitation links** are bearer secrets. Only a hash is stored, links expire, and acceptance is idempotent.
- **Cached responses.** Any shared cache in front of organization-scoped endpoints must key on the organization, or it becomes the leak the data layer prevents.

## Open questions

Carried from the design and to be settled during phase 0 and 1:

- Rolling deploy or a brief write freeze for the tightening migration and the actor-column rename.
- Who receives deployment alerts in a team. The phase 1 answer is whoever deployed.
- Trial state on organization switch. Trial status is per wallet, so it flips when moving between a trialing personal organization and a paid team, changing the wallet banner and the provider allow-list.
- Whether scoped API key permissions should land in phase 1.5 rather than later, given that adding the column is cheap now and breaking later.

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
