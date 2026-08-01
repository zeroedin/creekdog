# Creekdog — Data Model Sketch

> Status: **first draft**. Illustrative JSON-LD, not a frozen schema.

## Design goals
- Look like plain JSON to a Web Component; **be** RDF (JSON-LD).
- Lightweight for citizen reports; extensible per watershed; export-ready later.
- **Offline-aware**: client-generated stable IDs + explicit capture time, so
  future native apps can create records offline and sync without collisions.

## Core type: `PollutionReport` (v1)

**The citizen fills in exactly five things** (confirmed against the real tool;
expandable later): **category, location, description, photo, optional contact.**
Everything else below is system-managed. Model only what we collect — no external
ontology mapping (see `PLAN.md` §5).

```jsonc
{
  "@context": "https://creekdog.org/context/v1.jsonld",
  "@type": "PollutionReport",

  // ── citizen-supplied (the whole form) ───────────────────────────────
  "category": "https://fodc.example/category/illegal-dump",  // watershed's own list
  "location": { "type": "Point", "coordinates": [-79.9553, 39.6295] },  // lat/long point
  "description": "Tires and drums dumped at the bank.",
  "photos": ["https://fodc.example/reports/123/photo/1.jpg"],
  "reporter": { "contact": "steven@example.org" },  // OPTIONAL; omit = anonymous

  // ── system-managed ─────────────────────────────────────────────────
  "id": "urn:uuid:0190f3a1-…",           // client-generated UUIDv7, offline-safe
  "watershed": "https://fodc.example/",  // tenant
  "observedAt": "2026-07-08T14:12:00Z",  // capture time (offline-aware)
  "submittedAt": "2026-07-08T14:20:00Z",
  "status": "submitted",                 // INTERNAL ONLY — never published
  "outOfArea": false,                    // INTERNAL — set by the in-bounds gate
  "routing": { "agency": null, "notifiedAt": null }  // INTERNAL — never published
}
```

### Internal vs. published
Only a subset ever leaves the node. The **published** form drops `reporter`,
`submittedAt`, `status`, `outOfArea`, and `routing` entirely — presence in the public
feed *means* the report was accepted, so no status field is needed, and who was
notified is nobody else's business. **Rejected reports are deleted**, never published.
See `node-contract.md`.

`location` is only ever a **lat/long point** — the watershed boundary is the sole
non-point geometry in the system, and it lives on the `Watershed`, not the report.

### Report categories (SKOS, per-watershed — NOT pre-seeded)
Creekdog ships **no default categories.** Each watershed defines its own
`ConceptScheme`.

A local category carries just **two** things — the citizen-facing label, and its
`broadMatch` mapping to the shared core concern (`vocabulary.md`). It carries **no
routing**: the reviewer selects the agency at accept time (`agency-routing.md`).

```jsonc
{
  "@type": "skos:Concept",
  "id": "https://fodc.example/category/orange-water",
  "prefLabel": "Orange water / iron staining",              // local, citizen-facing
  "inScheme": "https://fodc.example/categories/",
  "broadMatch": "https://creekdog.org/vocab/concern/mining"  // → shared core (federation)
}
```

## `Watershed` (the tenant / node identity)

Each watershed/peer is a tenant with a **boundary polygon** and its own config. The
boundary is both a local feature (validate/label/display reports) and **published
node metadata** (its geographic coverage in the federation — see `federation.md`).

```jsonc
{
  "@type": "Watershed",
  "id": "https://fodc.example/",             // stable URL (self-hosted peer's base)
  "name": "Deckers Creek",
  "huc": "05020004",                          // USGS Hydrologic Unit Code (optional)
  "boundary": {                               // GeoJSON Polygon — published as coverage
    "type": "Polygon",                        // uploaded as KML *or* GeoJSON at setup,
    "coordinates": [ [ [-79.99,39.60], [-79.90,39.60], /* … */ ] ]  // stored as GeoJSON
  },
  "categoryScheme": "https://fodc.example/categories/",
  "agencies": ["https://fodc.example/agency/wv-dep"]  // node-local list the reviewer picks from
}
```

Point-in-polygon (report ∈ boundary) uses **SpatiaLite** on a self-managed node, or
app-side code on a colocated/serverless node (`hosting-and-cost.md`).

## Second type: `FishCatch` (v2)

Angler citizen-science observation — same "report" shape, different payload.

```jsonc
{
  "@type": "FishCatch",
  "id": "urn:uuid:0190f3b2-…",           // UUIDv7
  "watershed": "https://fodc.example/",
  "species": "Salmo trutta",             // + optional common name
  "count": 1,
  "lengthCm": 31.5,
  "released": true,
  "location": { "type": "Point", "coordinates": [-79.95, 39.63] },
  "observedAt": "2026-07-08T09:00:00Z",
  "reporter": { "contact": null }
}
```

Later, `FishCatch` maps to **Darwin Core** (`dwc:scientificName`,
`dwc:individualCount`, `dwc:eventDate`, `dwc:decimalLatitude/Longitude`) for
**GBIF** export.

## Later module: structured monitoring
Not part of citizen v1. When built, model as **SOSA/SSN** observations with
**QUDT** units, e.g. a pH / iron / aluminum reading at a monitoring site, and
macroinvertebrate indicator counts (mayfly/caddisfly/stonefly) as biotic
observations. Export targets: **EPA WQX** (chemistry), **GBIF** (biota).

## Standards mapping (summary)

Per the scope discipline (`PLAN.md` §5), v1 uses **Creekdog's own terms** — we do not
map our fields onto external ontologies. SKOS is the sole borrowed vocabulary.

| Concern | v1 (citizen) | Later (interop/export) |
|---|---|---|
| Envelope | JSON-LD (`spec/context/v1.jsonld`) | JSON-LD / Turtle |
| General props | **Creekdog's own terms** (`cd:`) | — |
| Location | GeoJSON (opaque `@json`) | GeoSPARQL, USGS HUC |
| Categories | per-watershed **SKOS** → shared core | — |
| Chemistry | — *(not collected)* | SOSA/SSN + QUDT → WQX |
| Fish / biota | — *(v2)* | Darwin Core → GBIF |

## Open modeling questions
- Whether **location needs coarsening** before publishing for sensitive reports
  (reporter contact is already never published).
