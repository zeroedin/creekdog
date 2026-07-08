# Creekdog — Federation (the core value proposition)

> Creekdog is a **network of interoperating watershed installs**, not a set of
> silos. Federation is the point of the project. This document is therefore central.

## Topology (resolved): flagship + peers
Creekdog is a **flagship model**, not a peer-to-peer mesh.

- **Creekdog (creekdog.org) = the flagship.** It plays several roles at once:
  the **aggregator** (harvests all peers into one cross-watershed view), the
  **registry** that peers join, the reference install, and an optional **host** for
  groups that don't want to self-host.
- **Peer = a watershed group's node**, participating in the federation as either a
  **self-hosted** install or a **colocated** tenant on the flagship. Both federate
  the same way: by publishing the standard contract.
- **Friends of Deckers Creek (FODC) = peer #1** — origin of the original tool and
  the pilot that proves the model.

This simplifies discovery enormously: a peer **announces itself to the flagship
registry**; no peer-to-peer discovery is needed. It also unifies with multi-tenancy
— the flagship install is multi-tenant (hosts colocated peers) *and* aggregates
external self-hosted peers into the same view.

## The core idea
Federation is **data interoperability**: many independent installs whose data can be
combined, searched, and compared as if it were one dataset — **without** forcing them
onto one server or one database.

The mechanism is simple and is the whole thesis behind choosing Linked Data / W3C
standards: **every install publishes its data the same standard way.** If node A and
node B both publish reports as JSON-LD, using a shared vocabulary, at stable web
addresses, then their data is *already* federatable.

> **Federation is achieved by how data is PUBLISHED, not by what database is behind
> it.** This is what keeps each node simple.

## Why this keeps nodes simple
Because the federation contract is the *published format*, a node's internal storage
is an implementation detail. A watershed can run plain **Postgres/PostGIS** and be a
perfect federation citizen — as long as it publishes the standard Linked Data.
Federation lives in **(publishing contract) + (aggregator)**, not inside every node.

```
   Node: Deckers Creek           Node: Other Creek           Node: Third Creek
   [app + Postgres]              [app + Postgres]            [app + anything]
        │ publishes                   │ publishes                  │ publishes
        ▼ JSON-LD (shared vocab, stable URLs)                      ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  Aggregator (e.g. creekdog.org)  — harvests nodes' published data │
   │  → one cross-watershed map / search / open dataset               │
   └──────────────────────────────────────────────────────────────────┘
```

## The four disciplines federation-first requires (from day one, all cheap)
1. **Global stable identifiers.** Every report, watershed, category, agency has a
   real **URL** as its ID (dereferenceable), not just a local database number.
2. **A shared core vocabulary.** Categories stay per-watershed, but each maps onto a
   **common concept scheme** so cross-node queries are meaningful (A's "illegal
   dump" ≈ B's "dumping"). Shared core + local extensions.
3. **A publishing spec.** The documented contract for what any install MUST expose to
   *be* a Creekdog node — feed format, discovery, identifiers. A defining artifact.
4. **An aggregator role.** The component that makes federation visible and valuable:
   a combined cross-watershed view. Likely **creekdog.org** as the flagship.

## Two ways to federate — anchor on the simple one
- **Harvest / aggregate (default).** Nodes publish feeds/dumps; the aggregator
  periodically collects them into one store it can search and map. This is how
  **GBIF** aggregates biodiversity data and how feed readers aggregate RSS. Robust,
  simple, tolerant of nodes going offline. **Only the aggregator needs extra tech.**
- **Live federated SPARQL (optional, later).** Nodes expose SPARQL endpoints and the
  aggregator issues cross-endpoint queries live (`SERVICE`). More power, more ops.
  Purely additive; nothing in v1 depends on it.

## What each install must do (the node contract — v1 sketch)
- Store reports (Postgres/PostGIS is fine).
- Assign each report a **stable URL identifier**.
- Tag each report's category with a term from the **shared vocabulary**.
- **Publish** verified reports as **JSON-LD**, discoverable via a documented feed
  (e.g. a paginated collection endpoint) so an aggregator can harvest them.

That's it. A node is simple; publishing correctly is what makes it federatable.

## Open questions
- **Vocabulary governance:** who curates the shared core scheme, and how do
  watersheds propose additions? (Flagship model → creekdog.org likely curates.)
- **Registry mechanics:** how does a peer register with the flagship, and what does
  the flagship store about it (URL, contact, harvest schedule, trust level)?
- **FODC onboarding:** does peer #1 start **colocated** on the flagship or
  **self-hosted**? (Colocated is the faster pilot.)
- How much data is public vs. held back (reporter identity always private; sensitive
  locations possibly coarsened before publishing).
