# Creekdog — Data Model Sketch

> Status: **first draft**. Illustrative JSON-LD, not a frozen schema.

## Design goals
- Look like plain JSON to a Web Component; **be** RDF (JSON-LD).
- Lightweight for citizen reports; extensible per watershed; export-ready later.
- **Offline-aware**: client-generated stable IDs + explicit capture time, so
  future native apps can create records offline and sync without collisions.

## Core type: `PollutionReport` (v1)

A citizen's concern report. Keep it small.

```jsonc
{
  "@context": "https://creekdog.org/context/v1.jsonld",
  "@type": "PollutionReport",
  "id": "urn:uuid:9f1c...   ",          // client-generated (UUID/ULID), offline-safe
  "watershed": "deckers-creek",          // tenant
  "category": "cd:illegal-dump",         // SKOS concept (per-watershed scheme)
  "description": "Tires and drums dumped at the bank.",
  "location": {                          // GeoJSON Point
    "type": "Point",
    "coordinates": [-79.9553, 39.6295]   // [lon, lat]
  },
  "observedAt": "2026-07-08T14:12:00Z",  // when the citizen saw it
  "submittedAt": "2026-07-08T14:20:00Z",
  "photos": ["https://.../evidence/1.jpg"],
  "reporter": {                          // optional; omit for anonymous
    "contact": "steven@example.org",
    "consentToContact": true
  },
  "status": "submitted",                 // submitted|triaged|routed|verified|closed
  "routing": {                           // filled by the routing engine
    "agency": null,
    "notifiedAt": null
  }
}
```

### Report categories (SKOS, per-watershed — NOT pre-seeded)
Creekdog ships **no default categories.** Each watershed defines its own
`ConceptScheme`. The first-class thing is not the label but the
**category → routing-agency relationship** (see `agency-routing.md`): every
category a watershed creates carries the decision of which agency it routes to.

A local category carries three things — label (local), `routesTo` (routing), and
`broadMatch` (federation mapping to the shared core). See `vocabulary.md`.

```jsonc
{
  "@type": "skos:Concept",
  "id": "https://creekdog.org/w/deckers-creek/category/orange-water",
  "prefLabel": "Orange water / iron staining",              // local, citizen-facing
  "inScheme": "https://creekdog.org/w/deckers-creek/categories/",
  "routesTo": ["https://creekdog.org/agency/wv-dep"],        // routing (local decision)
  "broadMatch": "https://creekdog.org/vocab/concern/mining"  // → shared core (federation)
}
```

## Second type: `FishCatch` (v2)

Angler citizen-science observation — same "report" shape, different payload.

```jsonc
{
  "@type": "FishCatch",
  "id": "urn:uuid:...",
  "watershed": "deckers-creek",
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

| Concern | v1 (citizen) | Later (interop/export) |
|---|---|---|
| Envelope | JSON-LD | JSON-LD / Turtle |
| General props | schema.org | schema.org |
| Location | GeoJSON | GeoSPARQL, USGS HUC |
| Categories | per-watershed SKOS | — |
| Provenance | PROV-O / Dublin Core (light) | PROV-O |
| Chemistry | — | SOSA/SSN + QUDT → WQX |
| Fish / biota | schema.org | Darwin Core → GBIF |

## Open modeling questions
- Photo/evidence storage: inline URLs vs. LDP non-RDF resources in the pod.
- Identifier scheme: UUID vs. ULID (ULID sorts by time — nice for feeds).
- How much of the report is public vs. staff-only (reporter contact must be
  private; location may need coarsening for sensitive reports).
