# Creekdog — Backend & Storage Options (decision brief)

> Status: **open decision.** Recommendation below, but the fork is real.

## Solid in one paragraph
**Solid** is a set of W3C specs (being formalized by the **Linked Web Storage**
Working Group) for storing data *on the web*, decoupled from apps. A **Pod** is a
personal web datastore — an HTTP server holding **Linked Data (RDF)** resources,
each at its own URL, read/written with plain HTTP via the **LDP** spec. Identity is
a **WebID** (a URL) with **Solid-OIDC** login; **WAC/ACP** documents control access
per resource. A **Solid server** (reference impl: **Community Solid Server / CSS**,
Node.js) hosts pods and implements all of this.

## The key insight
**Data-standard compliance and Solid-*protocol* compliance are separable.**
You can emit fully standards-compliant Linked Data (JSON-LD + schema.org + GeoJSON +
correct vocabularies) from a conventional database. Running a full **Solid server**
is a *stronger, separate* commitment — and Solid is shaped around **per-person pods
and per-resource permissions**, a partial mismatch for Creekdog (anonymous public
submissions, watershed-owned data, moderation queues, agency routing).

## Decision A — Solid server vs. lean custom API

### A1: Adopt Community Solid Server (full Solid)
- ➕ LDP, WAC/ACP, Solid-OIDC, WebID, JSON-LD/Turtle for free; maximal W3C/LWS
  alignment; instant interop with the Solid ecosystem.
- ➖ **Weak querying** — document-centric (fetch by URL); "unverified illegal-dump
  reports near X this week" needs bolt-on indexing/SPARQL. Painful for map+dashboard.
- ➖ Anonymous writes, moderation lifecycle, routing all live *outside* the pod
  anyway — you build a service layer regardless.
- ➖ Young project; steeper ops/learning curve; smaller contributor pool.

### A2: Lean custom API, Solid-*compatible* at the edges
- ➕ Fits the real model (anon submit, tenant-owned data, moderation, routing); you
  own querying; mainstream stack; easier contributors; can still expose clean
  JSON-LD / LDP-ish read endpoints for interop.
- ➖ Reimplements bits of Solid; partial interop; needs discipline to not drift.

## Decision B — native RDF store vs. Postgres + JSON-LD

### B1: Native triple/quad store (Fuseki, Oxigraph, GraphDB…)
- ➕ Natively Linked Data; **SPARQL**; per-tenant named graphs; best long-term
  interoperable/federatable record.
- ➖ **Geospatial support is uneven** (GeoSPARQL varies by store) — but Creekdog is
  fundamentally map-driven. ➖ Smaller talent pool; less mainstream ops; can be
  overkill at small volume.

### B2: Postgres + PostGIS, serializing JSON-LD
- ➕ **Best-in-class geospatial** (radius, boundary containment, maps) — central to
  this app. ➕ Mainstream, cheap, huge contributor pool. ➕ Still emits JSON-LD, so
  the *data* stays standards-compliant.
- ➖ Storage isn't natively RDF; maintain the JSON-LD mapping; no built-in SPARQL
  (add an RDF export/mirror later for federation).

### B3: Hybrid
Postgres/PostGIS as the operational store **+ RDF/SPARQL export** as the canonical
interoperable record. Best of both; more moving parts. Natural later evolution.

## "Heavier ops" — what that actually means
"Ops" = everything after the code is written: **installing, running, backing up,
upgrading, and troubleshooting** the live system, for years, often by a small
nonprofit or a volunteer. "Lightweight / easy to install" is an ops statement, and
it's where the paths diverge most.

| Ops dimension | Solid-pure (CSS + triplestore + app) | Pragmatic (API + Postgres/PostGIS, or SQLite) |
|---|---|---|
| Services to run | ~3 (CSS is storage only — you still build the app layer) | 1–2 |
| Install difficulty | High (Components.js DI config, JVM triplestore, wiring) | Low (compose file / single binary) |
| Footprint | Modest — modern stack available (see note) | Small; SQLite ≈ nothing |
| Backups | Two stores kept consistent | One database dump |
| Upgrades | 3 moving projects; Solid/LWS spec still evolving | Mainstream, predictable |
| Hosting / community | Niche; few managed options | Universal; managed everywhere |
| Who can realistically run it | Someone technical | A volunteer with a VPS |

**The Solid path is NOT a legacy/JVM stack** — that was an early mischaracterization.
CSS is **Node.js/TypeScript**; the JVM only appeared via one triplestore example
(Apache Jena Fuseki). You can run an all-modern stack: **CSS (Node/TS) + Oxigraph
(Rust)**, and CSS doesn't strictly require a separate triplestore (pluggable
filesystem/memory/SPARQL storage). What stays true regardless of how modern the
pieces are: (1) CSS is **storage only**, so you still build+run a separate app
service for anon submission / moderation / routing; (2) **Components.js DI config**
is fiddly; (3) the **Solid/LWS spec is still evolving** (upgrade churn); (4)
**geo/query still needs bolting on** — even Oxigraph doesn't do GeoSPARQL; (5)
smaller community / fewer managed hosts. So the real cost is *parts + config + a
moving spec + weak geo*, not "old runtime."

The extra weight buys **Solid-protocol interop + data portability**. But note: the
*data* is fully standards-compliant Linked Data (JSON-LD, correct vocabularies) on
**either** path — heavier ops only buys Solid *protocol/server* conformance, not
data compliance. Net tension: the Solid-pure path is hardest for exactly the small,
volunteer-run groups Creekdog most wants to enable.

## Recommendation
Given small-org scale, a **map-centric** app, anonymous writes, real business logic
(routing/moderation), and contributor accessibility:

**A2 + B2** — a lean **Solid-compatible API** over **Postgres/PostGIS** that always
serializes **JSON-LD**, with an **optional RDF/SPARQL export (B3)** as the upgrade
path to a pure Linked-Data record and cross-watershed federation.

- Keeps you **standards-facing** (compliant *data*) and geospatially strong now.
- Leaves a clean door to "more Solid / more RDF" as the **LWS** spec matures.

**Honest counter-argument:** if Solid *purity* and ecosystem interop matter more than
query power and ops-ease, **A1 + B1 (CSS + triplestore)** is the principled choice.
That is the actual fork to decide.

## What to decide next
1. Do we prioritize **Solid-protocol conformance** now, or **standards-compliant
   data + pragmatic ops** now with a Solid upgrade path?
2. Implementation language/runtime for the API (affects contributor pool).
3. Whether to stand up the **RDF export** in Phase 1 or defer to Phase 4.
