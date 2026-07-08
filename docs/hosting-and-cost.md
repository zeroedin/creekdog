# Creekdog — Hosting & Cost

> Watershed groups are typically small nonprofits with little money. Keeping the cost
> of participating at or near **$0** is a first-class design constraint. This is
> achievable because **federation lives in the publishing contract, not the database**
> (`federation.md`), so each node uses the cheapest storage that works.

## Principle: storage is a per-node choice behind one contract
A node federates by *publishing* standard JSON-LD, not by running any particular
database. So the storage engine is swappable per node:

- **SQLite is the default.** It's a single file — no separate database server, no
  managed-DB bill. Plenty for a watershed's volume (dozens–hundreds of reports/yr).
- **Postgres/PostGIS is an optional upgrade** for large/funded nodes that want heavy
  geospatial power or scale. **Not required.**

This revises the earlier "every node runs Postgres/PostGIS" assumption.

## Hosting tiers

| Tier | Who | Runs | Cost |
|---|---|---|---|
| **Colocated** (default) | most / poor groups | nothing — a tenant on the flagship | **$0** |
| **Self-host lite** | wants own install, no budget | app + SQLite, or free-tier serverless | **$0–5/mo** |
| **Self-host full** | larger / funded groups | app + Postgres/PostGIS | market rate (opt-in) |
| **Flagship** | Creekdog / consortium | aggregator + host + real DB | shared / grant-funded |

**Default answer for a poor group: colocate — run nothing, pay nothing.** The
flagship hosts them as a tenant; self-hosting is opt-in for sovereignty/capacity.

## The near-$0 self-host recipe
For a group that wants its own install without a budget:
- **Static public site** (map + published reports) → free on GitHub/Cloudflare Pages.
- **Dynamic bits** (accept submission, staff approve) → small serverless functions on
  free tiers.
- **Data** → free-tier serverless SQLite/Postgres (e.g. Cloudflare D1, Turso, Neon,
  Supabase — examples; watershed volume won't exceed free limits).
- Realistically **$0/mo**.

### Radical option: Git as the datastore
Because data is small, append-mostly, and human-reviewed, reports can live as
**JSON-LD files in a Git repo**: review = merging a pending file, publishing = the
repo served as static Linked Data on free hosting. $0, transparent, version-
controlled, and federation-native (published files *are* the contract). Weaker at
querying/geo, but fine at this scale. An option, not the default.

## Geospatial is not a blocker for cheap nodes
PostGIS power is only needed at scale. A node climbs this ladder only as far as it
needs — and Rung 1 alone is enough for v1 at watershed scale.

| Rung | Capability | Needs | Runs where |
|---|---|---|---|
| **1. Naive** | lat/lon columns, bbox `WHERE`, Haversine distance, app-side point-in-polygon | nothing | **everywhere**, incl. serverless |
| **2. + R\*Tree** | fast bbox/viewport via SQLite's built-in R\*Tree index | standard SQLite | almost everywhere |
| **3. SpatiaLite** | full OGC GIS — boundaries, jurisdiction point-in-polygon, projections, HUC/GeoJSON import | native extension | VPS / Docker / local |
| **4. PostGIS** | everything, at scale | Postgres server | flagship / funded nodes |

**Baseline = Rung 1** — runs literally anywhere (including the $0 serverless path)
and is plenty at a few hundred points. **SpatiaLite (Rung 3)** is "PostGIS for
SQLite": real GIS in a single file, no server — the upgrade a *self-hosted* node
reaches for to get **watershed-boundary** and **jurisdiction-routing** point-in-
polygon queries (see `agency-routing.md`) without running Postgres.

**Two node geo profiles** (proper watershed boundaries are wanted, `federation.md`):
- **Self-managed** (e.g. **FODC**): **Rung 3 SpatiaLite** — proper boundary geometry
  + point-in-polygon, still a single-file install on a ~$5 VPS. This is the profile
  the FODC self-hosted federation test uses.
- **Colocated / serverless**: **Rung 1** — store & display the boundary, do
  point-in-polygon in app code. No native extension needed; stays $0.
Either way the boundary is *published as coverage metadata*; only in-node
point-in-polygon differs by profile.

**SpatiaLite caveat:** it's a *native extension*, so it needs a runtime that allows
loading extensions — fine on VPS/Docker, but **many serverless SQLite platforms
(e.g. Cloudflare D1, Turso) don't allow it**. On the serverless path, stay on Rung 1
(lat/lon math). So SpatiaLite is a **self-hosted-only** capability, not a serverless
one. The **flagship** likely runs **PostGIS** since it aggregates all peers.

## Where cost concentrates — and how it's paid
The only node needing real capacity is the **flagship** (it aggregates all peers and
serves the cross-watershed map). That is the correct place for cost to land, because
it is the node with a funding mechanism:
- **Consortium cost-sharing** across member watersheds.
- **Nonprofit cloud credits** (Google for Nonprofits, AWS/Azure nonprofit programs,
  GitHub free for open source).

Net: cost concentrates where there's money to pay it and approaches $0 for the poor
peers who most need that.

## Open questions
- Default **colocation onboarding**: how does a group become a flagship tenant (self-
  serve vs. manual)?
- Which **free-tier serverless stack** to document as the blessed near-$0 recipe.
- Consortium **funding model** specifics (dues, grants, fiscal sponsor).
