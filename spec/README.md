# Creekdog Spec Files

Machine-readable artifacts for the node publishing contract
(see `../docs/node-contract.md` for the human explanation).

## `context/v1.jsonld` — the dictionary
JSON published by a node is just labeled values (`"description": "…"`). Those labels
mean nothing to another computer on their own. This file is the **dictionary that
defines what each Creekdog label officially means**, so the flagship can safely merge
data harvested from many independent nodes.

Nodes reference it from every document they publish:
```json
{ "@context": "https://creekdog.org/context/v1.jsonld", "…": "…" }
```

Notes:
- Almost every term is **Creekdog's own** (`cd:` = `https://creekdog.org/vocab#`).
  We deliberately do **not** map our fields onto external ontologies — see the scope
  discipline note in `../docs/PLAN.md` §5.
- The only borrowed vocabulary is **SKOS**, for the category/concern machinery — the
  one place a shared standard genuinely earns its place, because federation depends
  on categories meaning the same thing everywhere.
- Geometry (`location`, `boundary`) is carried as **opaque JSON** (`@type: @json`),
  so GeoJSON rides along untouched and map libraries consume it directly.
- **Versioned and additive-only.** `v1` is a permanent URL; consumers cache it. Add
  terms, never rename or remove them.

## `schema/node-v1.schema.json` — the validator
A **JSON Schema** so a node can check its own published output is correct *before*
the flagship rejects it. Validates the three published document shapes: the node
descriptor, a reports feed page, and a report (plus removal tombstones).

Beyond structural checks, it enforces the **privacy boundary**: a published report
containing `reporter`, `routing`, `status`, `outOfArea`, or `submittedAt` **fails
validation**. That makes accidental PII leakage a build error rather than a
production incident.

Validate with any JSON Schema 2020-12 tool, e.g.:
```bash
npx ajv-cli validate -s spec/schema/node-v1.schema.json -d your-feed-page.json --spec=draft2020
```

## Not included (deliberately)
**SHACL.** It would validate the RDF graph semantically (e.g. "is this `concern` a
real concept in the core scheme?"), but that single useful check is a couple of lines
of ordinary code at the flagship. Not worth the weight for node authors.
