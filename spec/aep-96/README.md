---
aep: 96
title: "Console: Redesign v2"
description: "Rebuild Akash Console around a top navigation, one Configure flow for every new deployment, a card-based deployments list and a deployment page that leads with status, endpoints and cost"
author: Maxime Beauchamp (@baktun14)
status: Draft
type: Standard
category: Interface
created: 2026-09-25
estimated-completion: 2026-12-31
discussions-to: https://github.com/orgs/akash-network/discussions
roadmap: major
requires: 84, 91
---

## Abstract

Akash Console grew one feature at a time, and by mid-2026 it showed. It had a sidebar meant for an app with many more pages, two ways to create a deployment that behaved differently, a deployments table that scrolled sideways on a laptop, and a deployment page that opened on a block of balance and escrow figures. Most screens assumed the user already knew how Akash works underneath: dseq, leases, escrow, bids, the chain.

This AEP describes Console Redesign v2, a rebuild of console.akash.network in a new design language, organised so that each screen answers the question the user opened it for. It covers the navigation shell, sign-in and onboarding, deployment creation, the deployments list, the deployment page, the account pages, and the wording used across all of them. Each surface ships behind its own feature flag next to the page it replaces, and the old page and its flag are deleted once the new one is on for everyone.

Most of it shipped between May and September 2026. What is left (the Logs, Events and Shell tabs, the Alerts and Usage pages, automatic provider selection, and taking blockchain wording out of the UI) is listed under Implementations. The work lives in `apps/deploy-web` in `akash-network/console`, with a few additive endpoints in `apps/api`. There are no chain or provider changes.

## Motivation

[AEP-84](../aep-84) made console.akash.network managed-wallet only, which took away the reason for much of the old UI: wallet connection, gas, per-deployment deposits and raw transactions. [AEP-91](../aep-91) then took per-deployment escrow out of the user's view. The layout built for self-custody was still there, and it got in the way:

- **Navigation.** A left sidebar suits an app with many pages that people switch between often. Console has four main destinations. The sidebar took width from the pages that need it most (logs, the configure form, the deployments list), and it did not match AkashML's navigation.
- **Two ways to create a deployment.** The Configure page created deployments through the Console API, which records each deployment on Console's side as it creates it. The older SDL Builder and new-deployment wizard broadcast the create transaction themselves and wrote no such record. Automatic funding works from that record, so a deployment made the old way stopped being funded once its first deposit ran out. Every new creation feature also had to be built and tested twice.
- **Pickers that offered things nobody could serve.** Region and GPU choices came from static catalogs. Measured on mainnet on 21 September 2026, 26 of the 38 region options had no online provider, and 47 of the 73 NVIDIA models were on no provider at all. Users found out only after requesting bids.
- **The deployments list.** It was a fixed-width table with six centred columns. The name was cut to about a dozen characters, the endpoint was hidden in a tooltip, `dseq` had a column of its own, and closed deployments sat behind a second tab.
- **The deployment page.** It opened on balance, cost, spent, status, time left and dseq as a grid of text, followed by a flat list of lease cards. People open that page to find out whether their app is running, where to reach it and what it costs.
- **Vocabulary.** Copy talked about the blockchain, chain messages and escrow, none of which a managed-platform user needs to know about.

## Specification

### Scope

In scope: the app shell and navigation; sign-in and onboarding; the new-deployment picker and the Configure page; the deployments list; the deployment page and its tabs; the account pages (Billing, API Keys, Usage, Alerts, Profile) and the Templates and Providers pages; product copy; in-app feedback prompts; and the flag-based rollout each surface ships through.

Out of scope: chain and provider changes; Console Air, a separate app ([AEP-84](../aep-84)); the funding model ([AEP-91](../aep-91)); how deployment definitions and secrets are stored ([AEP-92](../aep-92)); asynchronous actions and the notification center ([AEP-93](../aep-93)); organizations and projects ([AEP-94](../aep-94)); a replacement for deploying from a git repository ([AEP-85](../aep-85)); the Provider Console.

### Design principles

1. Lead with what the user came for: is it running, where do I reach it, what does it cost.
2. Keep one way to do each thing. A second path gets removed rather than hidden.
3. Offer only what can be delivered. A picker lists the options an online provider can serve, with a count of how many can.
4. Use platform terms. The UI does not say "blockchain", "chain" or "broadcast", and escrow appears only as the account-level figure on Billing ([AEP-91](../aep-91)).
5. Every screen works in both the light and dark themes.

### A. Navigation and shell

A top navigation bar replaces the sidebar. Its entries are Deployments, Providers and Templates, plus a Settings menu that holds Billing, API Keys, Usage and Alerts. The account menu behind the avatar holds Profile, the theme switch, Docs, the privacy policy and terms of service, and Contact us. The settings pages share one layout with the sections listed down the side. On narrow screens the navigation folds into a menu. During onboarding the navigation is hidden so that the only thing on screen is the next step.

### B. Sign-in and onboarding

Sign-in and sign-up share one screen: Continue with Google, Continue with GitHub, or Continue with email. Email sign-in is passwordless: the user types their address and enters the one-time code sent to it. Password sign-in still works for accounts that have a password, but the main screen no longer offers it.

A new account goes through onboarding before it reaches the rest of the app. The free trial starts on its own, the user picks a template or their own container, and the deployment runs through to a lease while a progress view shows each step. Users who would rather explore first can skip onboarding. Every deployment started from onboarding goes through Configure, like any other.

### C. Creating a deployment

**The picker.** `/new-deployment` only chooses what to deploy: a template, Run Custom Container, Launch Container-VM (a Linux VM reached over SSH), or Upload SDL (paste it or upload a file). A "Deploy with an agent" panel, behind a flag, points to the documentation for deploying through AI agents. Every option opens Configure.

**Configure.** One page with three panes. The configuration pane holds placements, each with a region, and within them services: image and registry, compute (CPU, memory, storage, GPU vendor, model, memory and interface), confidential compute, ports, environment variables, commands, and an optional runtime limit ([AEP-91](../aep-91)). The deployment pane summarises the resources and the cost. The marketplace pane shows provider quotes once they are requested, screened as described in [AEP-67](../aep-67). An SDL can be imported into the form or exported from it at any point. When few or no providers bid, the flow offers a direct way to contact the team.

**Placement options from live inventory.** The region and GPU pickers are filled from Console's provider inventory service, reached through the Console API; the browser never calls the inventory service directly. Offline and deleted providers contribute nothing. Each available option shows how many online providers can serve it, which also hints at how many quotes to expect. Unavailable options come after the available ones under an "Unavailable" heading. They are greyed out and cannot be selected, but search still finds them. A draft or imported SDL that pins an option nobody offers any more keeps showing it, listed under Unavailable. If availability cannot be loaded, both pickers fall back to the full catalog without counts. The option set must never be narrower than what online providers advertise, so nothing deployable disappears from a picker.

**One creation route.** Every deployment is created through the Console API, which writes Console's record at creation, so automatic funding covers it from the start. The SDL Builder and the old wizard are gone. `/sdl-builder` and `/deploy-linux` answer with permanent (308) redirects to Configure, and old links that pointed at a template, a deployment waiting for its lease, or a redeploy land on the matching Configure or deployment page.

**Automatic provider selection (planned).** Once quotes arrive, an "Auto deploy" action picks one provider per placement. The highest audit tier wins; among equals, 99% uptime or better beats anything lower; price breaks any tie left. The user sees each placement's provider and price and the deployment total before confirming, and every choice stays editable.

### D. Deployments list

Deployments render as cards in a grid that reflows to the width of the screen. A toggle switches to a dense list view for users with many deployments, and the choice survives a reload. Each card shows:

- the deployment name and one status badge, resolved the same way as on the deployment page so the two never disagree;
- how to reach it: named endpoints with their port, a disclosure when there are several, or plain text when there is nothing to reach ("private", "headless worker");
- a footer with the GPU model, vCPU, memory and storage;
- a countdown when the deployment is being reclaimed.

Closed deployments fold into an Archive section at the bottom of the same page, with a count, instead of a second tab. Search by name or dseq sits in the page header next to the New deployment action. The Console API serves the list, the search and the deployment names with server-side paging, so the browser no longer reads a user's whole deployment history to draw the page. Per-deployment escrow, "Add funds" and time left do not appear on the list.

### E. Deployment page

The header shows the status, the deployment name (editable in place), badges for confidential compute, GPU interconnect and trial deployments, and a Visit control for the live URL. Below it a summary lists the number of services, the cost per hour with a breakdown, the runtime limit when one is set, and the total GPU, vCPU, memory and storage. There is no escrow balance and no "Add funds".

| Tab | Content |
|---|---|
| Details | The deployment as placements, each with its service count, total resources and status. A placement expands into its services: status, resources, image and endpoints (URIs, forwarded ports and IPs, each with copy and open actions). Confidential compute and reclamation states stay visible. |
| Update | A structured editor for what can change on a running deployment: image, registry, variables, command and arguments, and ports. Hardware is shown but fixed for the lease. It sits behind a flag, and the YAML editor stays until the flag is on for everyone. |
| Logs | Live logs, filterable by placement and service, with download. |
| Events | Events with severity and timestamps, the same filters, and download. |
| Shell | A terminal in the browser, attached to the placement and service the user picks. |
| Settings | Billing (the runtime limit: extend it, or remove it to keep the deployment always on), Notifications (balance-threshold and deployment-closed alerts with their recipients, replacing the old Alerts tab), and a Danger Zone for closing the deployment. |

Logs, Events and Shell already work on the redesigned page but still use the old layout; restyling them with the placement and service filters is still to do. A link naming a tab that no longer exists opens Details.

### F. Account and catalog pages

Billing opens on an overview of the account balance and holds adding credits (a sheet with separate purchase and coupon tabs), payment methods, the transaction history including refunds, and the credit reload settings. API Keys, Profile, Templates and Providers are rebuilt in the new design. Alerts and Usage already sit in the settings layout; their redesign is still to do.

### G. Language

UI copy avoids blockchain terms. The banner shown while the network is unavailable or being upgraded describes an outage of the platform, and alert types get readable names instead of "Chain Message" and "Chain Event". The Console API's documentation and error messages already follow the same rule.

### H. Feedback

Short in-app prompts ask for feedback after a user's first deployment and when they come back after a period of inactivity. Contact us is reachable from the account menu and from a deployment that gets few or no quotes.

### I. Rollout

Each surface ships behind its own feature flag and runs next to the page it replaces. With the flag off, the old page renders unchanged. When a new page needs end-to-end coverage before rollout, a second, preview flag serves the same component on a development-only route that the product never links to. Both routes render the identical component, so the tests exercise the page that ships. Once the rollout flag is on for everyone, the end-to-end specs move to the canonical route, and the old page, the preview route and both flags are deleted in the same change rather than left in the code unreachable.

## Rationale

**Top navigation over a sidebar.** With four destinations, horizontal space matters more than a permanent list of links. Matching AkashML's navigation also helps people who use both products.

**Remove the second creation path instead of fixing it.** The funding gap came from the old path going around the deployment API, and every later feature would have needed the same fix twice. The last flow on the old path, deploying from a git repository, was off in production and would have had to be rebuilt to be useful, so it was removed with the builder. [AEP-85](../aep-85) proposes its replacement.

**Placements, not leases.** Users build their deployment out of placements and services on Configure, so the deployment page shows it the same way. Leases are how the network fulfils what the user designed, and they stay in the background.

**Account-level money on the deployment page.** Since [AEP-91](../aep-91), funding is account-wide. Showing per-deployment escrow next to it would contradict the Billing page.

**Live options that are never too narrow.** An option nobody can serve costs the user a wasted round of quotes. Hiding an option somebody could serve costs a deployment. The rules lean towards showing too much rather than too little, and fall back to the full catalog when inventory cannot be read.

**Archive instead of a tab.** People rarely open the list to look at closed deployments, but those deployments should not vanish either.

**A structured editor over YAML.** Most updates change an image tag or a variable. The form shows up front what cannot change on a running lease, where the YAML editor let users submit changes that would then be refused.

**One flag per surface, deleted at the end.** Each slice ships as a small change and can be switched off if it regresses. Deleting the old page straight after rollout keeps the codebase from carrying two UIs.

## Backward Compatibility

- `/sdl-builder` and `/deploy-linux` redirect permanently to Configure. A builder link carrying a saved template id opens that template on Configure. Links into the old wizard resolve to the matching Configure or deployment page, and `/new-deployment?step=choose-template` still renders the picker.
- Links to a deployment's old Alerts tab open Details. Deployment-closed emails link to the Settings tab.
- Saved templates still open in Configure, but saving a new template or editing a saved one left with the SDL Builder and has no replacement in this AEP.
- Deploying from a git repository (GitHub, GitLab and Bitbucket) is removed. It was already off in production.
- Deployments created through the old path before it was removed need a one-time backfill of Console's record so that automatic funding covers them. That backfill is tracked in the console repository.
- API users see no change. The new endpoints (deployment list with search, deployment names, placement options) are additive.
- Console Air is unaffected.

## Test Cases

- With a surface's flag off, the page it replaces renders unchanged. With it on, the new page renders on the canonical route.
- A list card's status badge matches the deployment page for running, reclaiming and closed deployments.
- Every creation path (template, custom container, container-VM, uploaded SDL, onboarding and redeploy) creates through the Console API and writes Console's record at creation.
- The region and GPU pickers never offer less than online providers advertise, offline and deleted providers add nothing, and an inventory failure shows the full catalog without counts.
- `/sdl-builder`, `/deploy-linux` and old wizard links redirect to the right place with the template or deployment carried over.
- Every redesigned screen renders in both themes, and the list reflows without horizontal scrolling at laptop width.
- End-to-end specs cover the onboarding deploy, Configure (container-VM included), the deployments list and the deployment page.

## Implementations

All in `akash-network/console`, merged between 2026-05-13 and 2026-09-24.

| Area | Pull requests |
|---|---|
| Top navigation and removal of the sidebar | [#3448](https://github.com/akash-network/console/pull/3448), [#3564](https://github.com/akash-network/console/pull/3564) |
| Sign-in screen and passwordless email | [#3168](https://github.com/akash-network/console/pull/3168), [#3186](https://github.com/akash-network/console/pull/3186), [#3272](https://github.com/akash-network/console/pull/3272) |
| Onboarding | [#3222](https://github.com/akash-network/console/pull/3222), [#3400](https://github.com/akash-network/console/pull/3400), [#3402](https://github.com/akash-network/console/pull/3402), [#3458](https://github.com/akash-network/console/pull/3458) |
| Configure page | [#3251](https://github.com/akash-network/console/pull/3251), [#3443](https://github.com/akash-network/console/pull/3443), [#3474](https://github.com/akash-network/console/pull/3474) |
| Placement options from live inventory | [#4005](https://github.com/akash-network/console/pull/4005), [#4007](https://github.com/akash-network/console/pull/4007), [#4028](https://github.com/akash-network/console/pull/4028) |
| One creation route | [#3935](https://github.com/akash-network/console/pull/3935), [#3998](https://github.com/akash-network/console/pull/3998) |
| Deployment page | [#3580](https://github.com/akash-network/console/pull/3580), [#3586](https://github.com/akash-network/console/pull/3586), [#3617](https://github.com/akash-network/console/pull/3617), [#3659](https://github.com/akash-network/console/pull/3659), [#3661](https://github.com/akash-network/console/pull/3661) |
| Update tab structured editor (behind a flag) | [#4017](https://github.com/akash-network/console/pull/4017) |
| Deployments list | [#3922](https://github.com/akash-network/console/pull/3922), [#3951](https://github.com/akash-network/console/pull/3951), [#3961](https://github.com/akash-network/console/pull/3961), [#3976](https://github.com/akash-network/console/pull/3976) |
| Billing and settings layout | [#3403](https://github.com/akash-network/console/pull/3403), [#3583](https://github.com/akash-network/console/pull/3583) |

Still open at the time of writing: the Logs, Events and Shell tab layouts; the Alerts and Usage pages; automatic provider selection; the UI copy changes in section G; and rolling out the Update tab editor, then retiring the YAML editor. The AEP moves to Last Call once those have shipped and their flags are removed.

## Security Considerations

- **Sign-in.** Social, passwordless and password sign-in all go through the existing identity provider, and an email code can be used only once.
- **One creation route.** Every deployment is created through the same deployment API, so there is one set of creation rules to review and test instead of two.
- **Placement options** reach the browser only through the Console API. The provider inventory service is not exposed to it.
- **The preview route** used during a rollout is development-only, never linked from the product, and deleted along with its flag.
- **Endpoints on list cards** are the deployment's own public URIs, shown only to its owner, as on the deployment page.
- **The Update tab editor** writes through the deployment API and keeps the sealing rules for variables and secrets set out in [AEP-92](../aep-92).
- **"Deploy with an agent"** opens public documentation in a new tab and passes nothing from the user's session.

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
