# Creekdog — Plain-Language Glossary

Jargon demystified, in the order it tends to come up. Each entry: what it is, and
whether Creekdog *needs* it.

### RDF / Linked Data
The **data model** at the heart of the "standards" goal: represent information as
**triples** — tiny facts of the form *(thing, property, value)*, e.g.
*(report-123, category, illegal-dump)*. Many triples form a **graph**. This is the
W3C-standard way to make data interoperable across systems.
**Need it?** Yes — as the *model* our data follows. (You don't need a special
database to follow it; see Triplestore.)

### JSON-LD
A **file format** for writing RDF that looks like ordinary JSON. Lets a Web
Component read plain JSON while the data is still valid Linked Data.
**Need it?** Yes — this is our wire format.

### schema.org / SKOS / GeoJSON / SOSA / Darwin Core
**Vocabularies** — agreed sets of property names so different systems mean the same
thing. schema.org (general), SKOS (category lists), GeoJSON (locations), SOSA
(sensor readings, *later*), Darwin Core (species records, *later*).
**Need it?** Yes for the first few; the rest are for later modules.

### SPARQL
The **query language** for RDF — "SQL, but for triples/graphs." Its standout trick
is **federation**: one query can reach across many independent datastores at once.
**Need it?** Not for v1. Only valuable if/when we want cross-watershed
(consortium) querying, and it can be added later.

### Triplestore
A **database** whose native unit is the fact-triple (instead of the table-row). It
stores RDF directly and answers SPARQL. Examples: Oxigraph (Rust), Apache Jena
Fuseki (Java), GraphDB.
**Need it?** **No.** Postgres can store our data and emit JSON-LD with equal
standards-compliance. A triplestore only adds native SPARQL + easy federation, and
can be bolted on later as a mirror if the consortium vision materializes.

### Postgres / PostGIS
A mainstream **relational database** (stores tables, answers SQL). **PostGIS** is
its geospatial extension — excellent at map queries ("within 2 km of here", "inside
this watershed boundary").
**Need it?** It's the recommended operational store, chosen for great geo + easy
install. (Not mandatory — the point is a simple, boring, well-supported DB.)

### Solid
A set of W3C specs for storing data **on the web**, decoupled from apps, using RDF +
HTTP. Being standardized by the **Linked Web Storage (LWS)** W3C working group.
**Need it?** We adopt it *pragmatically* — Solid-compatible data/URLs — not
necessarily a full Solid server. See `backend-options.md`.

### Pod
In Solid, a **Pod** is a personal web datastore (an HTTP server holding your RDF
resources). In Creekdog, the data steward is the **watershed group**, so "the pod"
(if any) belongs to the group, not to individual citizens.
**Need it?** Only if we go the full-Solid route; the concept (a per-tenant data
space) applies regardless.

### WebID / Solid-OIDC
**WebID** = an identity that is a URL. **Solid-OIDC** = the login protocol for it.
The standards-pure way to authenticate. Citizens are anonymous, so this only
matters for **staff** identity — and email/passkey is a lighter alternative.
**Need it?** Undecided; an open question for staff auth.

### LDP (Linked Data Platform)
A W3C spec for reading/writing Linked Data resources over plain HTTP
(GET/PUT/POST/PATCH). It's how Solid servers expose data.
**Need it?** Only relevant if we adopt a Solid server; we can expose similar clean
JSON-LD endpoints without it.

### Community Solid Server (CSS)
The main open-source **Solid server** (Node.js/TypeScript). Reference implementation
of the Solid specs.
**Need it?** Only on the "Solid-pure" backend path — see `backend-options.md`.

### Web Components / Custom Elements / Shadow DOM / Lit
**Web standards** for building UI from reusable HTML elements without a framework.
**Lit** is a thin helper (templating) on top of these standards — no Vue/React.
**Need it?** Yes — this is our chosen frontend approach.
