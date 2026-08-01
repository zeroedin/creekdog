# Creekdog — Planning Document

> Status: **first draft**, actively evolving. This is a working plan, not a spec.
> Last substantive update: 2026-08-01.

## 0. TL;DR — the simple version (read this first)

Creekdog v1 is **one web app + one backend + one database**:

1. Web app (Web Components + Lit): pin a spot on a map, pick a category, describe
   it, add a photo, submit.
2. Backend service saves it to a small database (**SQLite by default** — a file, no
   DB server; Postgres/PostGIS optional for large nodes).
3. Staff review each submission (the review *is* the spam filter) and click
   **approve & send** → it emails the right agency.
4. Data is published as **JSON-LD** → this is what makes Creekdog *open* and
   *standards-compliant*. Nothing more is required for either.

**Federation is the core value proposition** — Creekdog is a *network* of watershed
installs whose data interoperates, not a set of silos. Crucially, federation is
achieved by **how each node publishes its data** (Linked Data / JSON-LD, a shared
vocabulary, stable web addresses), **not** by any particular database. So each
install stays "one app + one database" *and* is a full federation citizen, because
the federation lives in the **publishing contract + an aggregator**, not inside
every node. See `federation.md`.

What that means for v1: build the four steps above **with two disciplines from day
one** — (a) give everything a **stable URL as its identifier**, and (b) map each
watershed's categories onto a **shared core vocabulary**. These are cheap and are
the foundation federation stands on. The only *heavy* federation piece — an
aggregator that gathers many nodes into one cross-watershed view — is centralized
(likely creekdog.org) and never imposed on individual watersheds. Live federated
SPARQL and triplestores remain optional, aggregator-side, later.

**"Open" = MIT code anyone can run + open data in a standard format** — that plus the
publishing contract is what makes the federation possible.

---

## 1. What Creekdog is

Creekdog is an **open-source, self-hostable API and datastore for citizen-science
watershed data**. Its proven core — as built by Friends of Deckers Creek — is a
lightweight **citizen incident reporter**: a "911 for the creek" where anyone can
report a pollution concern and have it routed to the responsible agency. Around
that core, the platform generalizes to other citizen observations (e.g. angler
fish catches) and, later, to structured monitoring data.

The point of the project is to be **free, standards-native, and organization-
agnostic**, so any watershed group can run the backend — self-hosted or colocated
on a shared install — and, if a community forms, share costs and development
through a consortium.

### Origin / ground truth
- Built by **Friends of Deckers Creek (FODC)** for the Deckers Creek watershed (WV).
- Creekdog itself = **anonymous pollution/concern reports → routed to the correct
  government agency.** Report types seen in the original tool: *illegal dumps,
  untreated sewage, suspicious drilling, stream or wetland dredge and fill, and
  "any other concern or suspicion."*
- FODC *also* runs a separate scientific monitoring program (quarterly water
  chemistry — pH, conductivity, flow, iron, aluminum, manganese, calcium; annual
  macroinvertebrate and fish sampling). Deckers Creek's signature pollution is
  **acid mine drainage (AMD)**. This structured program is a **later module**, not
  the citizen-facing v1.

## 2. Principles (non-negotiables)

1. **Web standards only on the client.** No Vue/React/Angular. UI is built from
   **Web Components (Custom Elements + Shadow DOM)**, using **Lit** as a thin,
   standards-aligned templating helper (no VDOM, no framework runtime).
2. **W3C / Linked Data native.** Data is Linked Data expressed as **JSON-LD** so
   simple clients see plain JSON while the data remains interoperable RDF.
3. **Agnostic, self-hostable & lightweight.** Anyone can run the backend, and it
   must be **easy to install** and cheap to operate — favor simple, few moving
   parts over heavyweight infrastructure. No proprietary cloud lock-in.
   MIT-licensed code; open data licensing for the scientific record.
   *Corollary:* a **human reviewer is the spam filter** — no machine-review layer.
4. **Right-sized modeling.** Use the lightest standard that fits. Don't force
   heavy sensor/biodiversity ontologies onto a simple citizen report.
5. **Multi-tenant from the start.** One install can host many watershed groups.

## 3. Actors & data ownership

| Actor | Role | Identity |
|---|---|---|
| **Citizen reporter** | Submits reports/observations over the plain web. Often anonymous. | None required (optional contact for follow-up) |
| **Watershed group (tenant)** | Owns and stewards its data. Configures categories, its agency list, and boundary. Reviews reports and selects the agency on accept. | Authenticated (staff accounts) |
| **Host / operator** | Runs an install; may host one or many watershed groups. Could be an individual, an org, or a consortium. | Server admin |
| **Flagship (creekdog.org)** | The aggregator + registry + reference install + optional host for colocated peers. | Project operator (Steven) |
| **Peer** | A watershed group's node in the federation — self-hosted or colocated. **FODC = peer #1.** | Per-node |

**Key decision:** the **data steward is the watershed group, not the individual
citizen.** Citizens interact through the web and are frequently anonymous, so we do
**not** require per-person Solid pods. Each watershed group owns its own datastore;
citizen submissions land there.

## 4. Architecture (current direction)

```
  Citizen (browser, anonymous)                Watershed staff (browser, authed)
        │  submit report                              │  review → accept/reject
        ▼                                             ▼  → pick agency → send
  ┌──────────────────────────────────────────────────────────────┐
  │  Creekdog node  (Node.js/TypeScript, multi-tenant capable)    │
  │  - Public submission endpoint (no auth)                       │
  │  - In-bounds gate: point-in-polygon vs. watershed boundary    │
  │  - Review queue (the spam filter; rejected = deleted)         │
  │  - Agency delivery: reviewer selects agency, then send        │
  │  - Publishes the node contract (JSON-LD) for harvesting       │
  └──────────────────────────────────────────────────────────────┘
        │
        ▼
  Small database — SQLite by default (a file); Postgres/PostGIS optional
        │
        └─► Public feed: accepted reports as JSON-LD (no PII, no status, no routing)
                 │
                 ▼  harvested by
            Flagship aggregator (creekdog.org) → cross-watershed map
```

- **Native mobile apps come later** (iOS/Android) and will handle offline field
  capture. The API must therefore be a **clean, public, documented HTTP+JSON-LD
  API** that those apps can consume — this is a design constraint on v1 even though
  we ship web-only first.
- **Offline:** v1 web is online-only, but the **schema is offline-aware**
  (client-generated stable IDs, explicit capture timestamps) so later sync is clean.

### Solid / Linked Web Storage — where we actually landed
Solid (and the W3C **Linked Web Storage** WG standardizing it) inspired the approach,
but we adopt it **pragmatically, not literally**:
- ✅ **Kept:** Linked Data as JSON-LD, stable dereferenceable URLs, open publishing —
  the parts that make federation work.
- ❌ **Not adopted:** running a Solid server (CSS), per-citizen pods, WAC/ACP, and
  WebID/Solid-OIDC login. Each was full cost for no benefit at our shape — citizens
  are anonymous, the watershed group is the data steward, and staff log into exactly
  one app. See `backend-options.md`.

All of it remains addable later as an additive layer; nothing here forecloses it.

## 5. Standards stack (right-sized)

> **Scope discipline:** Creekdog models **only what it actually collects, in its own
> terms.** We do *not* map every field onto external ontologies — that is ceremony
> with no payoff here. The **only** thing that must be shared and curated is the
> **core concern list** (`vocabulary.md`), because that is what makes cross-watershed
> queries meaningful. Everything else is Creekdog's own small, flat field set.

**Citizen report (v1):**
- **JSON-LD** wire format, over a **small Creekdog-owned `@context`** that mostly
  names our own fields.
- **SKOS** for the category/concern machinery — the one place a standard vocabulary
  genuinely earns its place (federation depends on it).
- **GeoJSON** for geometry — see below.

**Geometry is deliberately tiny:**
- **Report location = a lat/long point.** Nothing else.
- **Watershed boundary = the only non-point geometry**, uploaded once at tenant
  setup (not per report). Accept **KML or GeoJSON**; convert and **store GeoJSON**.

**Later modules only (not v1, do not pre-build):**
- Fish catches → **Darwin Core** for GBIF export.
- Structured monitoring → **SOSA/SSN + QUDT**, EPA WQX export.
- US watershed identity via **USGS HUC** codes.

## 6. Multi-tenancy, hosting & consortium

- A **watershed** is a tenant: its own data partition/graph, its own category
  scheme, its own agency-routing table, its own map boundary (GeoJSON / HUC).
- One install hosts **N watersheds**. Hosting modes:
  1. **Self-host** — a group runs its own install.
  2. **Colocate** — a group's tenant lives on a shared install (e.g. Steven's).
- **Consortium**: a shared install run collectively; costs/dev shared. Governance
  model TBD (see open questions).

## 7. Trust, spam & verification

- Anonymous submission invites abuse → need **anti-spam** (rate limiting,
  challenge, moderation queue) without forcing accounts.
- Reports have a **lifecycle**: `submitted → triaged → routed → verified/closed`.
- **In-bounds jurisdiction gate** (before routing): point-in-polygon vs. the
  watershed boundary. Out-of-bounds reports are **accepted and tagged `out-of-area`**
  for the reviewer to dismiss or handle (no config knob — review already covers the
  strict case). Cheap app-side check on any node. See `agency-routing.md`.
- Only **verified** data enters the public scientific feed; raw submissions stay
  in a moderation view.

## 8. Roadmap (phased)

- **Phase 0 — Foundations (now).** This plan; data-model sketch; retire the old
  Vue deploy; decide server/runtime; scaffold repo (API + web-components frontend).
- **Phase 1 — Incident report MVP.** Single tenant. Define the **watershed
  boundary** (import HUC/GeoJSON, display on map, in-bounds check). Submit a
  pollution report (category + location on a map + description + photo + optional
  contact) → stored as JSON-LD → public verified feed → basic agency-routing
  (email/webhook). Web Components frontend (Lit). Public read API.
- **Phase 2 — Federation proof (FODC self-hosted).** *Federation is the core value,
  so prove it early.* Stand up **FODC as a self-hosted, off-flagship node** using
  SpatiaLite for its boundary; it publishes the **node contract** (`node-contract.md`:
  descriptor + paged reports feed + report resources); the **flagship harvests** it
  into the aggregate cross-watershed view. Real external peer = the real end-to-end
  test of the thesis.
- **Phase 3 — Multi-tenancy + fish catches.** Tenant config (categories, routing,
  boundary); second report type (angler fish catch); moderation UI.
- **Phase 4 — Public API hardening for native apps.** Documented, versioned
  HTTP+JSON-LD API; auth for staff; offline-friendly submission contract.
- **Phase 5 — Structured monitoring module.** SOSA/QUDT chemistry + Darwin Core
  fish/macroinvertebrate; SPARQL/GeoJSON query; GBIF/WQX export.
- **Phase 6 — Consortium tooling.** Shared hosting ops, billing/cost-sharing,
  governance.

## 9. Open questions

*(Resolved and moved to the decisions log: server/runtime, storage engine, staff
identity, data licensing, report fields, geometry, data migration.)*

*(Also done: `@context` + JSON Schema — see `spec/`; agency selection — reviewer-chosen.)*

1. **Flagship photo caching:** the node stores and serves its own photos (decided);
   still open is whether the **flagship copies them** when harvesting, or hot-links
   the node and risks link rot if a peer disappears.
2. **Registration auth:** how the flagship verifies a registrant controls the node
   domain (e.g. a challenge file), to prevent spoofed nodes.
3. **Consortium governance:** legal/financial structure for shared hosting.
4. **Address geocoding** — if citizens should *search an address* rather than drop a
   pin, free geocoders (OSM Nominatim) have usage limits and Google's is notably
   better. Pin-drop + device GPS likely covers the real "I'm standing at the creek"
   case; decide whether search is needed at all.
5. **creekdog.org migration:** retire the defunct Vue app + fix the broken
   `gh-pages` deploy workflow (it currently nests `…temp-deployment-folder/`
   directories); decide what the domain serves during the rebuild.

## 10. Decisions log

| Date | Decision |
|---|---|
| 2026-07-08 | Client uses Web Components + **Lit** (thin helper allowed). |
| 2026-07-08 | Tenant/data steward = **watershed group**, not individual citizen; no per-person pods. |
| 2026-07-08 | v1 vertical slice = **pollution incident report + agency routing** (not lab chemistry). |
| 2026-07-08 | Offline handled by future native apps; web v1 online-only but schema offline-aware. |
| 2026-07-08 | Standards right-sized: JSON-LD + schema.org + GeoJSON + SKOS for citizen reports; SOSA/Darwin Core reserved for later monitoring module. |
| 2026-07-08 | **No pre-seeded categories.** Each watershed defines its own; the first-class relationship is category → routing-agency. See `agency-routing.md`. |
| 2026-07-08 | Routing = two layers: per-watershed map (category→agency) + per-agency delivery adapter (manual/email/prefilled-form/open311/webhook), with a staff approval gate before any outbound send. |
| 2026-07-08 | **Every submission is human-reviewed before delivery** (review = the spam filter; no machine-review layer). Original auto-delivery is replaced. |
| 2026-07-08 | **Lightweight / easy-to-install** is an explicit goal, reinforcing few-moving-parts choices throughout. |
| 2026-07-08 | **Federation is THE core value proposition** — Creekdog is a network of interoperating watershed nodes. Achieved via the data-*publishing contract* (Linked Data, shared vocab, stable URLs), not via any specific database. See `federation.md`. |
| 2026-07-08 | **Topology RESOLVED — flagship, not P2P.** creekdog.org = flagship (aggregator + registry + reference install + optional host). Peers = watershed nodes, self-hosted or colocated. **FODC = peer #1.** Discovery = simple registry at the flagship. |
| 2026-07-08 | **Shared vocabulary is a core artifact.** Local categories (per-watershed wording + routing) each `broadMatch` onto a small, flagship-curated, additive-only **core concern scheme** (~10 concepts, `vocabulary.md`). Local usability + global comparability. Report **status** is a companion shared list. **Core scheme accepted as v0** (additive-only henceforth). |
| 2026-07-08 | **Backend RESOLVED (simple node):** each install = one backend service, *publishing JSON-LD as its federation contract*. A node stays simple; it's a full federation citizen by publishing, not by running heavy infra. |
| 2026-07-08 | **Storage is a per-node choice; SQLite is the DEFAULT** (a file, no DB server, no bill). Postgres/PostGIS is an **optional upgrade** for large/funded nodes — not required. Revises the earlier "every node runs Postgres" assumption. See `hosting-and-cost.md`. |
| 2026-07-08 | **Cost model:** most/poor groups **colocate on the flagship and pay $0** (run nothing); self-host lite (SQLite / free-tier serverless) is ~$0–5/mo; real cost concentrates at the **flagship**, funded by consortium + nonprofit cloud credits. Geo at small scale needs no PostGIS (SpatiaLite / bbox math suffices). |
| 2026-07-08 | **Day-one federation foundation (cheap, required):** (a) stable **URL identifiers** for everything; (b) a **shared core vocabulary** that per-watershed categories map onto. |
| 2026-07-08 | **Deferred to the aggregator, not each node:** live federated **SPARQL**, triplestores. The aggregator (likely creekdog.org) harvests nodes' published data; individual watersheds never need this. |
| 2026-07-08 | **Watershed boundary is first-class.** Each watershed/peer has a boundary polygon (HUC-sourced or custom): defines coverage, validates/labels reports (in-bounds, sub-HUC), drives the map — **and is published as node metadata** so the flagship maps coverage/overlap. |
| 2026-07-08 | **Two node geo profiles.** *Self-managed* (e.g. FODC) uses **SpatiaLite** (Rung 3) for proper boundary + point-in-polygon, still a single-file simple install; *colocated/serverless* renders the boundary + does app-side point-in-polygon (Rung 1). |
| 2026-07-08 | **FODC = the self-hosted (off-flagship) reference peer** — the real end-to-end federation test (publish → flagship harvest). Resolves the colocated-vs-self-hosted question for peer #1. Colocation stays the default for capacity-poor *other* groups. |
| 2026-07-08 | **In-bounds jurisdiction gate is REQUIRED** — point-in-polygon vs. the watershed boundary precedes routing; out-of-bounds = outside the group's jurisdiction. Out-of-bounds reports are **accepted and tagged `out-of-area`** for review (no config knob — strict groups just dismiss them in review). Cheap app-side on any node — does NOT force SpatiaLite. Boundary is a prerequisite to accepting reports. Future: flagship hands misdirected reports to the correct peer. |
| 2026-07-08 | **Scope discipline:** model **only what Creekdog collects, in its own terms** — no forcing external ontologies onto our fields. The **core concern list is the only shared/curated vocabulary** (that's what federation needs). Shrinks the `@context` to mostly our own terms. |
| 2026-07-08 | **Report = five citizen-supplied fields:** category, location (lat/long point), description, photo, optional contact. Expandable later. Everything else is system-managed. |
| 2026-07-08 | **Geometry is tiny:** report location is a **lat/long point only**; the **watershed boundary is the sole non-point geometry**, uploaded once at tenant setup. Accept **KML or GeoJSON**, convert and **store GeoJSON** (Leaflet/modern maps consume it natively; not locked to Google Maps). |
| 2026-07-08 | **Runtime = Node.js/TypeScript** — one language across backend and the Lit frontend; largest web contributor pool; deploys anywhere incl. serverless free tiers. |
| 2026-07-08 | **Published data license = CC-BY** — free reuse with attribution to the watershed group. (Code stays MIT.) Recorded in each node's descriptor. |
| 2026-07-08 | **Staff auth (admin side only; citizens never log in):** **password set at account creation** as the baseline, with **optional magic-link and passkey** sign-in and **TOTP 2FA** available. Rejects WebID/Solid-OIDC for now — full cost, no benefit, since staff log into exactly one app and there are no per-citizen pods. Additive later if wanted. |
| 2026-08-01 | **Identifiers = UUIDv7** (RFC 9562). Supersedes the UUID-vs-ULID question: it *is* a UUID (universal DB/tooling support) *and* sorts by creation time, which suits our cursor-paginated change feed and gives better index locality than random v4. Client-generatable, so the offline mobile apps can mint IDs at the creek. Clock skew is harmless — the feed orders by the server-stamped `modified`. |
| 2026-08-01 | **Photos.** Arrive **with the submission** (no separate upload endpoint → no orphans; submission rate limiting covers uploads). **EXIF stripped + resized server-side, always** — phone photos embed GPS/device IDs that would leak an anonymous reporter's position; also resize client-side for weak mobile signal. **Private until accepted** (staff-only, server-enforced; S3 objects private by default), public on acceptance, **deleted with the report on rejection** (all derivatives + object storage). Storage is a **pluggable adapter**: `filesystem` default, `s3` (R2/B2/MinIO) optional; **SQLite-blob rejected**. Requires size caps, per-report photo limits, and submission rate limiting. |
| 2026-08-01 | **No auto-routing in v1 — the reviewer selects the agency** when accepting a report, from the watershed's agency list. **Categories therefore carry no routing** (label + core-concern mapping only) — *supersedes* the 2026-07-08 "category → agency is the first-class relationship" decision. No routing-rules table, no jurisdiction engine. **Future:** auto-*suggest* an agency via per-agency bounding boxes/areas drawn on the map — a suggestion the reviewer can change, never an automatic send. |
| 2026-08-01 | **Published data is minimal.** The public feed carries **no status** (presence = accepted), **no routing** (who was notified stays internal), and **no reporter PII**. **Rejected reports are deleted outright** — never published, no trace. Only an already-published report needs a small `RemovedReport` tombstone so harvesters drop their copy. |
| 2026-08-01 | **Maps: open by default, configurable per node.** Library = **Leaflet** (framework-agnostic, no key). Default basemap = **USGS National Map** (`USGSTopo` + the `USGSHydroCached` overlay so the creek network is drawn) — public domain, **no API key, no billing account**, authoritative, and avoids OSM's production usage-policy limits. Note it's an ArcGIS service: tile path is `{z}/{y}/{x}` (y before x). A node **MAY** configure another provider (incl. Google) if it wants better geocoding/familiarity. **Google rejected as the default**: requires a per-node billing account + key (breaks $0/easy-install), ToS friction with redistributing open data, and contradicts the open/agnostic ethos. |
| 2026-07-08 | **No data migration — fresh start.** The new system begins empty; the old closed-source Creekdog data stays archived. No importer needed in Phase 1. |
| 2026-07-08 | **Node publishing contract specified** (`node-contract.md`). A node = 3 URLs (descriptor + paged JSON-LD reports feed w/ opaque-cursor incremental harvest + dereferenceable report resources) + registration; flagship harvests via registry + polling. Published = **verified only, PII-stripped**; node resolves `category → concern` and publishes both; versioned by `contractVersion` + `vocabulary`. Plain HTTP + JSON-LD, no triplestore/SPARQL. |
