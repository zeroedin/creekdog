# Creekdog — Planning Document

> Status: **first draft**, actively evolving. This is a working plan, not a spec.
> Last substantive update: 2026-07-08.

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
3. **Agnostic & self-hostable.** Anyone can run the backend. No proprietary cloud
   lock-in. MIT-licensed code; open data licensing for the scientific record.
4. **Right-sized modeling.** Use the lightest standard that fits. Don't force
   heavy sensor/biodiversity ontologies onto a simple citizen report.
5. **Multi-tenant from the start.** One install can host many watershed groups.

## 3. Actors & data ownership

| Actor | Role | Identity |
|---|---|---|
| **Citizen reporter** | Submits reports/observations over the plain web. Often anonymous. | None required (optional contact for follow-up) |
| **Watershed group (tenant)** | Owns and stewards the data space ("the pod"). Configures categories, agency routing, map. Reviews/verifies reports. | Authenticated (org accounts) |
| **Host / operator** | Runs the install; may host one or many watershed groups. Could be an individual, an org, or a consortium. | Server admin |

**Key decision:** the **data steward is the watershed group, not the individual
citizen.** Citizens interact through the web and are frequently anonymous, so we do
**not** require per-person Solid pods. Each watershed group gets a
**Solid-compatible datastore** it owns; citizen submissions land there.

## 4. Architecture (current direction)

```
  Citizen (browser, anonymous)                Watershed staff (browser, authed)
        │  JSON-LD report                             │  review / verify / config
        ▼                                             ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  Creekdog API  (per-install, multi-tenant)                    │
  │  - Public submission endpoint (anti-spam, no auth)            │
  │  - Solid-compatible read/write of report resources (LDP)      │
  │  - Query endpoint (filtered feeds; later: SPARQL/GeoJSON)     │
  │  - Agency-routing engine (report → notify correct agency)     │
  │  - Tenancy: data partitioned per watershed group              │
  └──────────────────────────────────────────────────────────────┘
        │
        ▼
  Datastore (Linked Data / RDF quad store, per-tenant graphs)
        │
        ├─► Public verified feed (JSON-LD, GeoJSON, RSS/Atom)
        └─► Export adapters (later): Darwin Core → GBIF, chemistry → WQX/SOSA
```

- **Native mobile apps come later** (iOS/Android) and will handle offline field
  capture. The API must therefore be a **clean, public, documented HTTP+JSON-LD
  API** that those apps can consume — this is a design constraint on v1 even though
  we ship web-only first.
- **Offline:** v1 web is online-only, but the **schema is offline-aware**
  (client-generated stable IDs, explicit capture timestamps) so later sync is clean.

### Solid / Linked Web Storage
Solid (and the W3C **Linked Web Storage** WG standardizing it) is the model for the
tenant datastore: reports as LDP resources, access control via WAC/ACP, WebID for
staff identity. We adopt Solid **pragmatically**: the watershed group's storage is
Solid-shaped, but citizen submission is a simple public POST, not a per-user pod
write. Candidate server to evaluate: **Community Solid Server (CSS)**.

## 5. Standards stack (right-sized)

**Citizen report / fish catch (v1–v2):**
- **JSON-LD** wire format.
- **schema.org** for general, discoverable properties + web crawlability.
- **GeoJSON** (+ WGS84 Geo) for location — friendlier than GeoSPARQL for points.
- **Per-watershed SKOS concept scheme** for report categories (illegal dump,
  sewage, …) — extensible per tenant.
- **PROV-O / Dublin Core** (light) for provenance: who/when/verified-by.

**Structured monitoring module (later):**
- **SOSA/SSN** for water-chemistry observations, **QUDT** for units.
- **Darwin Core** for fish/macroinvertebrate occurrences → **GBIF** export.
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
- Only **verified** data enters the public scientific feed; raw submissions stay
  in a moderation view.

## 8. Roadmap (phased)

- **Phase 0 — Foundations (now).** This plan; data-model sketch; retire the old
  Vue deploy; decide server/runtime; scaffold repo (API + web-components frontend).
- **Phase 1 — Incident report MVP.** Single tenant. Submit a pollution report
  (category + location on a map + description + photo + optional contact) →
  stored as JSON-LD → public verified feed → basic agency-routing (email/webhook).
  Web Components frontend (Lit). Public read API.
- **Phase 2 — Multi-tenancy + fish catches.** Tenant config (categories, routing,
  boundary); second report type (angler fish catch); moderation UI.
- **Phase 3 — Public API hardening for native apps.** Documented, versioned
  HTTP+JSON-LD API; auth for staff; offline-friendly submission contract.
- **Phase 4 — Structured monitoring module.** SOSA/QUDT chemistry + Darwin Core
  fish/macroinvertebrate; SPARQL/GeoJSON query; GBIF/WQX export.
- **Phase 5 — Consortium tooling.** Shared hosting ops, billing/cost-sharing,
  governance.

## 9. Open questions

1. **Server/runtime:** adopt Community Solid Server, or build a lean custom
   Solid-compatible API (more control, less standards surface for free)?
2. **Storage engine:** RDF quad store (e.g. for native Linked Data) vs. a
   conventional DB with a JSON-LD mapping layer. Trade-off: purity vs. ops ease.
3. **Staff identity:** WebID + Solid-OIDC (pure) vs. email/passkey (easier
   onboarding) vs. both.
4. **Agency routing:** how are agencies + jurisdictions modeled per watershed?
   Static config table first; geospatial jurisdiction lookup later?
5. **Data licensing:** CC0 vs CC-BY for the public scientific record.
6. **Consortium governance:** legal/financial structure for shared hosting.
7. **Map tooling:** Leaflet (framework-agnostic lib) vs. another approach for
   picking/showing locations, within the "no framework" spirit.

## 10. Decisions log

| Date | Decision |
|---|---|
| 2026-07-08 | Client uses Web Components + **Lit** (thin helper allowed). |
| 2026-07-08 | Tenant/data steward = **watershed group**, not individual citizen; no per-person pods. |
| 2026-07-08 | v1 vertical slice = **pollution incident report + agency routing** (not lab chemistry). |
| 2026-07-08 | Offline handled by future native apps; web v1 online-only but schema offline-aware. |
| 2026-07-08 | Standards right-sized: JSON-LD + schema.org + GeoJSON + SKOS for citizen reports; SOSA/Darwin Core reserved for later monitoring module. |
