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

**The in-bounds gate is required on every node — and it's cheap everywhere.**
Validating that a submission falls inside the watershed boundary (the jurisdiction
gate, see `agency-routing.md`) is *one point vs one polygon, once per submission* —
trivial in plain app code on **any** node, including serverless/Rung 1. So this
mandatory check does **not** force SpatiaLite.

**Two node geo profiles** (proper watershed boundaries are wanted, `federation.md`):
- **Self-managed** (e.g. **FODC**): **Rung 3 SpatiaLite** — proper boundary geometry
  for the *heavier* spatial work (bulk queries, sub-HUC labeling, agency-jurisdiction
  routing), still a single-file install on a ~$5 VPS. The FODC federation test uses this.
- **Colocated / serverless**: **Rung 1** — store & display the boundary, run the
  in-bounds gate (and any point-in-polygon) in app code. No native extension; stays $0.
Either way the boundary is *published as coverage metadata* and the in-bounds gate
runs; only the heavier spatial queries differ by profile.

**SpatiaLite caveat:** it's a *native extension*, so it needs a runtime that allows
loading extensions — fine on VPS/Docker, but **many serverless SQLite platforms
(e.g. Cloudflare D1, Turso) don't allow it**. On the serverless path, stay on Rung 1
(lat/lon math). So SpatiaLite is a **self-hosted-only** capability, not a serverless
one. The **flagship** likely runs **PostGIS** since it aggregates all peers.

## Photo storage

**Processing on upload (always, non-negotiable):**
- **Strip EXIF.** Phone photos embed **GPS coordinates** and device identifiers.
  Since submission is anonymous, publishing an unstripped photo would leak exactly
  what the rest of the privacy model protects. Strip metadata before storing.
- **Resize/recompress** to web size. A ~12 MB phone photo becomes ~300 KB — roughly
  40× smaller, which makes every storage option cheap.

**Photos arrive with the submission — there is no separate upload step.** Files are
saved as part of saving the report, so a stored file can never exist without a
report (no orphans, no sweep needed), and the **submission rate limit covers uploads
too**. Resize **client-side before upload** so a citizen on weak mobile signal isn't
waiting on a 12 MB transfer; still strip EXIF **server-side**, since anything done in
the browser can be bypassed.

**Lifecycle — photos are private until the report is accepted:**
1. **Submitted / pending review** — stored but **not publicly served**. Visible to
   staff in the review queue only; access is **enforced by the server**, never by an
   unguessable URL. On the `s3` adapter this means objects are **private by default**
   (no public-read ACL), served via the app or a signed URL.
2. **Accepted & published** — the photo becomes public at its stable URL and appears
   in the published report.
3. **Rejected** — the report **and its files are deleted**: original plus every
   resized derivative, including from object storage.

**Storage is a pluggable adapter** (same pattern as maps and the database):

| Adapter | Use | Notes |
|---|---|---|
| **filesystem** *(default)* | self-hosted VPS/Docker node | A folder next to the app. Simplest thing that works. Back up the folder alongside the DB. |
| **s3** | serverless nodes, or any node wanting offloaded storage | Any S3-compatible service — Cloudflare R2, Backblaze B2, MinIO (self-hosted). Generous free tiers. |

*(SQLite blob storage was considered and **rejected** — keeping photos out of the
database keeps the DB small and nimble.)*

**Abuse limits (required).** Submission is anonymous and unauthenticated, so it needs
a **max file size**, a **max photos per report**, and **rate limiting on submission**
— which, because photos only arrive with a report, is also the upload limit.

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
