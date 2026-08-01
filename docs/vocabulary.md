# Creekdog — Shared Vocabulary

> The vocabulary is what makes federation *mean* something. It is therefore a core
> design artifact, curated by the flagship. See `federation.md`.

## The tension it resolves
- **Local usability:** citizens should see plain, locally-worded choices
  ("Orange water / iron staining", "Straight pipe").
- **Global comparability:** the flagship must query "all mining-related reports
  across every watershed" and have it mean the same thing everywhere.

**Resolution:** each watershed defines its own categories, but every category must
**map onto a small shared *core* scheme.** Local wording for humans; shared core for
the federation.

## A category carries two things
1. **Label** — citizen-facing wording (local, per-watershed).
2. **`broadMatch` → core** — which shared concept it rolls up to (federation).

A category carries **no routing**. The reviewer selects the agency when accepting a
report (`agency-routing.md`), so classification and delivery stay independent.

## Shared core concern scheme (v0 — ACCEPTED, curated by flagship)
Stable URIs under `https://creekdog.org/vocab/concern/<id>`. Additive-only.

| Core concept | Covers |
|---|---|
| `dumping` | Illegal dumping / solid waste — trash, tires, drums, debris |
| `sewage` | Sewage & wastewater — untreated sewage, sanitary overflow, failing septic, straight pipes |
| `industrial-discharge` | Industrial/chemical discharge & spills — oil sheen, chemical odor, discharge pipe |
| `mining` | Mining impacts — acid mine drainage, mine discharge, orange/metallic staining |
| `oil-gas` | Oil & gas / drilling activity — well pads, brine, "suspicious drilling" |
| `sediment-erosion` | Sediment, erosion & construction runoff — muddy water, earth disturbance |
| `agricultural` | Agricultural runoff — manure, nutrients, livestock in stream |
| `stormwater` | Urban stormwater / illicit discharge — foam, illicit connections |
| `stream-alteration` | Physical alteration of stream or wetland — dredge & fill, channelization |
| `fish-kill-bloom` | Fish kill, algal bloom, or other biological alarm — dead fish, HABs |
| `other` | Other / unknown concern — the catch-all |

~10 buckets + catch-all: small enough to agree on and keep stable, broad enough to
cover most watersheds. `mining` and `oil-gas` are included because Deckers Creek
(AMD) and the original tool ("suspicious drilling") both require them.

> STATUS: **accepted as v0** (2026-07-08). Additive-only from here — future changes
> add concepts or deprecate, never rename/remove, so peer mappings never break. Can
> still be reconciled against FODC's real category list as peer #1 is configured.

## How a peer maps its categories

```jsonc
// Core concept (flagship-curated)
{
  "@type": "skos:Concept",
  "id": "https://creekdog.org/vocab/concern/mining",
  "prefLabel": "Mining impacts",
  "definition": "Pollution from active or abandoned mining, incl. acid mine drainage.",
  "inScheme": "https://creekdog.org/vocab/concern/"
}

// FODC local category — label + core mapping (no routing)
{
  "@type": "skos:Concept",
  "id": "https://fodc.example/category/orange-water",
  "prefLabel": "Orange water / iron staining",          // local, citizen-facing
  "inScheme": "https://fodc.example/categories/",
  "broadMatch": "https://creekdog.org/vocab/concern/mining"  // federation
}
```

A citizen sees "Orange water / iron staining"; the flagship counts it under
`mining`. An Iowa peer's "Manure in the creek" → `agricultural`. The two become
directly comparable in the aggregate.

`broadMatch` = local term is *narrower* than the core concept. Use `exactMatch` when
a local category is effectively identical to a core concept.

## Governance
- **Peers create local categories freely** — local autonomy, no gatekeeping.
- **Every local category MUST map to exactly one core concept** (default `other`).
  This mapping is the price of admission to the federation.
- **Flagship curates the core scheme** — versioned and **additive-only**: never
  rename or remove a concept (deprecate instead), so existing mappings never break.
- **Peers propose new core concepts** when `other` starts hiding a real pattern.

## Companion controlled lists (also shared, for the same reason)
- **Report status:** `submitted → triaged → routed → verified → closed` (+ `rejected`).
  The aggregate needs "verified" to mean the same thing on every node.
- (Later) shared units/vocabularies only if the structured-monitoring module lands
  (SOSA/QUDT, Darwin Core) — out of scope for the citizen report.

## Open questions
- Review the core scheme against **FODC's actual categories** — adjust before v0.
- **Flat vs. hierarchy — DECIDED: flat for v0.** The ~10 concepts stay as a flat
  list. Parent groupers (e.g. `extractive` → {`mining`, `oil-gas`}) can be added
  later via `skos:broader` with zero breakage — peers map to leaves, so parents are
  purely additive. Introduce a shallow hierarchy only if native rollups become
  painful in the aggregate dashboard.
- Where do **core URIs** live if a peer self-hosts — always `creekdog.org/vocab`
  (single source of truth), which we recommend, so every node maps to the same anchors.
- Multi-language `prefLabel`s (SKOS supports language tags) — worth planning for?
