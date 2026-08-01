# Creekdog — Federation Readiness: Gaps & Work Items

> Honest audit of how well federation — **the core value proposition** — is actually
> specified, and what's missing. Written 2026-08-01 after deciding not to adopt
> Solid's protocols. See `federation.md`, `node-contract.md`, `spec/`.

## First, the reframe
Dropping Solid did **not** weaken federation. Solid is a *storage protocol*: it
standardizes how one person's data server behaves. It provides **nothing** for what
federation actually requires — a shared vocabulary, an aggregation model, a registry,
a harvest protocol, coverage metadata. We had to specify all of those ourselves, and
a Solid-based Creekdog would have had *less* federation spec, not more.

## What is genuinely in place
- `federation.md` — topology (flagship + peers), harvest-vs-live-query model.
- `node-contract.md` — the wire spec: descriptor, paged feed with opaque cursors,
  report resources, removal tombstones, registration, flagship harvest loop.
- `spec/context/v1.jsonld` — the shared dictionary.
- `spec/schema/node-v1.schema.json` — validator; tested against 10 cases, including
  PII-leak rejection.
- `vocabulary.md` — shared core concern scheme (v0), which makes cross-node data
  *comparable*, not merely collectable.
- Stable URL identifiers as a day-one discipline; UUIDv7 record IDs.
- Published watershed boundaries as **coverage metadata**.
- **CC-BY** licensing, so aggregation is legally unambiguous.

**Assessment:** the *publishing* side is well specified. The *aggregator* side is
thin. Nothing is proven.

---

## Gap 1 — Nothing is validated (highest risk)
No two nodes have ever exchanged data. The contract is paper until Phase 2 stands up
FODC as a real self-hosted peer and the flagship harvests it.

**Work items**
- [ ] **Conformance test suite** *(highest value, buildable now)* — a script you point
      at any node URL that: fetches the descriptor, walks the feed through pagination,
      validates every document against `spec/schema/node-v1.schema.json`, checks
      cursor/`nextSync` behaviour, verifies tombstones remove records, and confirms no
      PII leaks. Turns "are you a valid Creekdog node?" into a command. Doubles as a
      peer's self-check before registering.
- [ ] End-to-end Phase 2 test: FODC publishes → flagship harvests → aggregate map.

## Gap 2 — The aggregator is barely specified
We defined what a node *publishes* in detail, and almost nothing about how the
flagship **stores, deduplicates, maps, and searches** the harvested result.

**Work items**
- [ ] Aggregator data model — how harvested reports from N peers are stored.
- [ ] Identity/dedup rules — reports are keyed by node-issued URL; define behaviour on
      re-harvest, edits, and a peer changing its base URL.
- [ ] Conflict/trust handling — what happens when a peer publishes something invalid,
      or floods the aggregate.
- [ ] The cross-watershed map/search itself — the actual user-facing payoff.
- [ ] Harvest failure handling — peer offline, partial page, malformed document.
- [ ] Retention — what the flagship keeps if a peer disappears entirely.

## Gap 3 — Registry mechanics undefined
`node-contract.md` §4 sketches `POST /registry` but not the substance.

**Work items**
- [ ] What the flagship stores per peer (URL, contact, harvest schedule, trust level).
- [ ] Approval workflow (manual for v1) and de-registration.
- [ ] **Anti-spoofing:** verify a registrant controls the node domain (e.g. a
      challenge file at a well-known path).
- [ ] Default harvest poll interval; whether the descriptor advertises a suggested one.

## Gap 4 — Contract versioning has no real story
There is a `contractVersion` field and nothing behind it.

**Work items**
- [ ] Define compatibility policy: what a v2 flagship does with v1 peers, and vice versa.
- [ ] Deprecation path — how long old versions are harvested.
- [ ] Whether the flagship reports version drift back to peers.

## Gap 5 — Vocabulary governance undefined
The core concern scheme is accepted as v0 and additive-only, but the *process* isn't.

**Work items**
- [ ] How a peer proposes a new core concept (issue template? registry API?).
- [ ] Who curates and on what cadence (flagship model → creekdog.org curates).
- [ ] How scheme versions are published and how peers learn of additions.
- [ ] Whether unmapped/`other`-heavy peers get flagged for review.

## Gap 6 — Smaller open items
- [ ] **Photo caching:** flagship copies harvested photos vs. hot-links the peer
      (link rot if a peer disappears).
- [ ] **Location coarsening** for sensitive reports before publishing.
- [ ] **Multi-language labels** (SKOS supports language tags).
- [ ] **Non-US peers** — HUC codes and USGS tiles are US-only; what an international
      peer uses instead.

---

## Suggested order
1. **Conformance suite** — makes the contract executable and testable immediately.
2. **Registry + anti-spoofing** — needed before any real peer registers.
3. **Aggregator model** — needed for Phase 2's payoff.
4. **Versioning & vocabulary governance** — needed before peer #3, not peer #1.
