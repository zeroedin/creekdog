# Creekdog — Agency Routing (design exploration)

> Status: **exploration**, not settled. This is the topic that shapes Phase 1 most.

## Problem statement
A citizen files a concern. Creekdog must get that concern to the **right agency**,
in whatever form that agency accepts. Two things vary independently:

1. **Who** a category routes to — a **per-watershed business decision**.
2. **How** a report physically reaches that agency — a **per-agency mechanism**.

Categories are *not* pre-seeded by Creekdog; each watershed defines its own and
attaches routing to them.

## Step 0 — the in-bounds jurisdiction gate (before any routing)
Routing only makes sense once a report is confirmed to be *this watershed's to
handle*. So the first check on submission is **point-in-polygon against the watershed
boundary**:

- **In-bounds** → continue to routing below.
- **Out-of-bounds** → outside the group's jurisdiction. Handle per a **per-watershed
  policy**:
  - `reject` — hard block at submission.
  - `flag` *(recommended default)* — accept but tag **out-of-area** so the human
    reviewer decides (the boundary check becomes an input to review, not a silent wall).
  - `accept` — no gating.

This gate is *one point vs one polygon per submission* — cheap in app code on any
node (no SpatiaLite required; see `hosting-and-cost.md`). Defining the watershed
boundary is therefore a **prerequisite to accepting reports**.

**Future federation payoff:** the flagship knows every peer's boundary, so an
out-of-bounds report at one node is often in-bounds for a neighbor — the network can
eventually **hand a misdirected report to the correct watershed**. (A node alone
knows only its own boundary; the flagship enables the handoff.)

## Two-layer model

### Layer 1 — Routing map (business config, per watershed)
Core relationship: **category → agency**.
- May be **one category → many agencies** (notify several parties).
- May later be refined by **location / jurisdiction**: the same category can route
  differently depending on where in the watershed the report is (municipal vs.
  county vs. state). Start with `category → agency`; add spatial rules later.

```
RoutingRule:
  watershed
  category            # this watershed's SKOS concept
  area?              # optional GeoJSON / jurisdiction filter (later)
  agencies[]         # one or more targets
```

### Layer 2 — Delivery adapter (per agency)
Each agency has a **delivery mechanism**. Model it as a pluggable adapter so new
mechanisms drop in without touching core logic.

| Mode | Mechanism | Notes |
|---|---|---|
| **manual** | Report enters a staff queue; a human phones/emails/forwards. | No integration; most reliable; default fallback. |
| **email** | Auto-composed templated email (details, photos, map link) to intake address. | Sweet spot — nearly every agency accepts email. |
| **prefilled-form** | Deep-link / pre-fill the agency's online complaint form; human submits. | Resilient to form changes; semi-automated. |
| **open311** | POST to an **Open311 / GeoReport v2** endpoint. | Fully automated where municipal 311 supports it; rare. |
| **webhook** | POST to any endpoint the watershed configures. | Generic automation. |

```
Agency:
  name, jurisdiction
  contactChannels     # email, phone, form URL, api endpoint...
  delivery:
    mode              # manual | email | prefilled-form | open311 | webhook
    config            # address / URL / credentials as needed
    autoSend          # auto-transmit AFTER human approval? (review is always required)
```

## Human review is mandatory (settled)
The original CreekDog auto-delivered every submission with **no spam protection**.
Creekdog's rule going forward: **every submission is reviewed by a person before it
leaves the system.** The human review *is* the spam/false-report filter — we
deliberately avoid any machine-review/classification layer, which keeps the system
**lightweight and easy to self-install**.

- Flow: `submitted → (staff review) → approve & send → delivered → closed`
  (or `rejected` — spam/duplicate/out-of-scope, never delivered).
- `autoSend` therefore means **"auto-compose and transmit *after* a human approves,"**
  not "send without review." There is no path that skips review.
- Post-approval **delivery** may be automated (`email`, `open311`, `webhook`) or
  manual (`manual`, `prefilled-form`) per agency. `manual` + `email` cover ~95% of
  real agencies with almost no integration effort — the right Phase-1 default.

## Audit & closing the loop
Every delivery attempt writes a **RoutingEvent**:
```
RoutingEvent:
  report, agency, mode
  sentAt, sentBy        # staff user or "auto"
  payload               # exactly what was transmitted
  agencyReference?      # case/ticket number returned by the agency
  outcome               # queued | sent | failed | acknowledged | closed
```
This gives accountability and lets Creekdog report status back to the (opted-in)
reporter: "your report was forwarded to X on DATE, case #NNN."

## Open questions
- How is **jurisdiction** determined per watershed — static table, or spatial lookup
  against agency service-area boundaries? (Spatial version = point-in-polygon; needs
  Rung-3 SpatiaLite or Rung-4 PostGIS — see `hosting-and-cost.md`. Static table works
  on any node, incl. serverless.)
- Do we need **escalation** (no acknowledgement in N days → notify next party)?
- Where do agency **credentials** (API keys, form quirks) live and who maintains them?
- Should reporters be able to **opt in to status updates** while staying anonymous
  (e.g. a claim code rather than an email)?
