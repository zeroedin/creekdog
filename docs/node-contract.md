# Creekdog — Node Publishing Contract

> The contract every peer implements to participate in the federation, and that the
> flagship harvests. It is the build spec for **Phase 2 (Federation proof)**. See
> `federation.md` for the why.
>
> Design stance: **plain HTTP + JSON-LD, harvestable by a dumb client.** No triplestore,
> no SPARQL, no exotic tech required of a node. A node is *three URLs + a harvest loop*.

## Overview — a node exposes three resources
1. **Node descriptor** — who I am, my boundary, my license, where my reports live.
2. **Reports feed** — paged JSON-LD collection of *verified* reports, incrementally
   harvestable via a cursor.
3. **Report resource** — each verified report, dereferenceable at its stable URL.

Plus: **registration** (tell the flagship your descriptor URL once) and the flagship's
**harvest loop**.

All resources are served as `application/ld+json`. Serving an HTML representation of
the same URLs (content negotiation) is encouraged for humans but optional.

---

## 1. Node descriptor
A single document at a stable, conventional URL: `https://<node>/.well-known/creekdog`
(the exact path is flexible — the *registered* descriptor URL is what's authoritative).

```jsonc
{
  "@context": "https://creekdog.org/context/v1.jsonld",
  "@type": "CreekdogNode",
  "id": "https://fodc.example/",
  "contractVersion": "1.0",
  "watershed": {
    "@type": "Watershed",
    "id": "https://fodc.example/",
    "name": "Deckers Creek",
    "huc": "05020004",
    "boundary": { "type": "Polygon", "coordinates": [ /* GeoJSON … */ ] }
  },
  "vocabulary": "https://creekdog.org/vocab/concern/v0",  // core scheme version it maps to
  "categoryScheme": "https://fodc.example/categories/",   // its local SKOS scheme
  "reports": "https://fodc.example/reports",              // the feed (entry page)
  "license": "https://creativecommons.org/licenses/by/4.0/",
  "publisher": { "name": "Friends of Deckers Creek", "url": "https://deckerscreek.org" },
  "updated": "2026-07-08T12:00:00Z"
}
```
The **boundary** here is the node's published *coverage* (used by the flagship to place
it on the national map and, later, to hand off misdirected reports — `agency-routing.md`).

---

## 2. Reports feed (paged JSON-LD collection, incremental)
Entry page: `GET https://fodc.example/reports`. Harvest incrementally with an **opaque
cursor**: `GET https://fodc.example/reports?since=<cursor>`.

```jsonc
{
  "@context": "https://creekdog.org/context/v1.jsonld",
  "@type": "Collection",              // AS2/Hydra-style paged collection
  "id": "https://fodc.example/reports?since=CURSOR_ABC",
  "updated": "2026-07-08T12:00:00Z",
  "orderedBy": "modified-asc",        // change-feed order: oldest change first
  "next": "https://fodc.example/reports?since=CURSOR_ABC&page=2",  // more of THIS harvest
  "nextSync": "CURSOR_XYZ",           // store this; pass as ?since= on the NEXT harvest run
  "items": [ /* PollutionReport and RemovedReport objects, inline */ ]
}
```

**Why cursors, not raw timestamps:** the node hands out opaque `next`/`nextSync`
cursors so it controls pagination and checkpointing, avoiding same-timestamp
tie bugs. The harvester treats them as blobs.

**Items are inline and full** (at watershed scale — hundreds of reports — no need for
summary+fetch). Each item is also dereferenceable at its own URL (§3).

---

## 3. Report resource (the PUBLISHED form)
`GET https://fodc.example/reports/123`

```jsonc
{
  "@context": "https://creekdog.org/context/v1.jsonld",
  "@type": "PollutionReport",
  "id": "https://fodc.example/reports/123",
  "watershed": "https://fodc.example/",
  "category": "https://fodc.example/category/orange-water",   // local category
  "concern": "https://creekdog.org/vocab/concern/mining",     // resolved CORE concept (denormalized)
  "description": "Orange discharge staining the bank.",
  "location": { "type": "Point", "coordinates": [-79.955, 39.629] },  // GeoJSON
  "observedAt": "2026-07-01T09:00:00Z",
  "modified": "2026-07-02T14:00:00Z",
  "photos": ["https://fodc.example/reports/123/photo/1.jpg"]
  // NOTE: no status field — presence in the feed MEANS accepted.
  // NOTE: no reporter contact / PII, and no routing info. See Privacy below.
}
```

**No `status` in the published form.** Everything in the feed is an accepted report,
so a status field would be a constant. The full review lifecycle stays internal.

**Removal of an already-published report** uses a small tombstone, so a flagship that
already harvested a copy knows to drop it:
```jsonc
{
  "@type": "RemovedReport",
  "id": "https://fodc.example/reports/123",
  "modified": "2026-07-05T10:00:00Z"
}
```
Scope: this applies **only** to reports that were previously published. A report
**rejected during review was never public** — it is simply deleted, with no tombstone
and no trace in the feed. A node MAY discard a tombstone once all harvesters have
seen it.

> The node **resolves `category → concern` and publishes both.** The flagship therefore
> aggregates on `concern` without crawling each node's category scheme.

> Geometry note: GeoJSON is used because maps consume it directly. For strict RDF, a
> node MAY additionally express geometry as a GeoSPARQL WKT literal; not required in v1.

---

## 4. Registration
A node announces itself to the flagship registry once:
```
POST https://creekdog.org/registry
{ "node": "https://fodc.example/.well-known/creekdog" }
```
The flagship validates the descriptor and (with operator approval — trust is admitted
at registration) schedules the node for harvest. Manual approval is fine for v1.

---

## 5. Harvest loop (flagship side)
For each registered node:
1. `GET` the descriptor (conditional — ETag/Last-Modified). Note boundary/license.
2. `GET reports?since=<savedCursor>`; follow `next` until exhausted.
3. **Upsert** each `PollutionReport` by its `id`; **delete** any `RemovedReport` by `id`.
4. Save the collection's `nextSync` as the node's new cursor.
5. Sleep until the next scheduled poll.

Idempotent and resumable: re-running from a saved cursor is always safe.

---

## 6. Privacy boundary (publishing is the gate)
- Published reports carry **no reporter PII** — contact/identity is stripped at publish.
- **Photos are EXIF-stripped on upload** — phone images embed GPS and device IDs, so
  an unstripped photo would leak the reporter's position regardless of the rules
  above. See `hosting-and-cost.md`.
- **Only accepted reports are published.** Nothing mid-review appears in the feed, and
  **rejected reports are deleted outright** (never published, no trace).
- **Routing is never published** — who was notified, when, and any agency case number
  stay internal to the node.
- The published form therefore carries **no status** and **no routing**: presence in
  the feed means accepted, and that is all the outside world learns.
- Sensitive locations **MAY** be coarsened before publishing (node's choice).
- This means the feed is safe to be fully public — which is what makes open federation
  and open data licensing possible.

---

## 7. Versioning & compatibility
- **`contractVersion`** on the descriptor — harvesters check they speak it.
- **`vocabulary`** pins the core concern-scheme version the node maps to. The core is
  **additive-only** (`vocabulary.md`), so a node mapping to v0 stays forward-compatible.
- The **`@context`** (`https://creekdog.org/context/v1.jsonld`) maps our terms onto
  schema.org / SKOS / GeoJSON; it is a flagship-published deliverable (see open items).

---

## 8. Optional later: push instead of poll
A node MAY notify the flagship on change (**WebSub** or **Linked Data Notifications**)
so harvest is near-real-time. v1 is polling on a schedule; push is a drop-in
optimization that doesn't change the feed format.

---

## Open items
- ~~Author the **`@context`** and a validation shape~~ — **DONE**, see `spec/`
  (`context/v1.jsonld`, `schema/node-v1.schema.json`). SHACL deliberately skipped.
- Decide default **harvest poll interval** (and whether the descriptor advertises a
  suggested one).
- **Photo hosting**: inline URLs on the node vs. copied by the flagship (link rot if a
  node disappears). Probably: flagship may cache copies of published photos.
- **Auth on registration**: how the flagship verifies a registrant controls the node
  domain (e.g. a challenge file), to prevent spoofed nodes.
