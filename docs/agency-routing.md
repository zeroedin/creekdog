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
- **Out-of-bounds** → **accept, but tag `out-of-area`** so the human reviewer
  decides (dismiss, or handle anyway). The boundary check is an *input to review*,
  not a silent wall or a hard block. No per-watershed config knob: a strict group
  simply dismisses out-of-area reports in review, which the mandatory review step
  already covers.

This gate is *one point vs one polygon per submission* — cheap in app code on any
node (no SpatiaLite required; see `hosting-and-cost.md`). Defining the watershed
boundary is therefore a **prerequisite to accepting reports**.

**Future federation payoff:** the flagship knows every peer's boundary, so an
out-of-bounds report at one node is often in-bounds for a neighbor — the network can
eventually **hand a misdirected report to the correct watershed**. (A node alone
knows only its own boundary; the flagship enables the handoff.)

## Two-layer model

### Layer 1 — Agency selection: **the reviewer chooses** (v1)
There is **no automatic routing in v1.** When a reviewer accepts a report, they
**select the agency** (or agencies) from the watershed's list. Nothing is derived
from the category.

This means:
- **Categories carry no routing.** A category is just a label + its core-concern
  mapping for federation (`vocabulary.md`). *(Supersedes the earlier
  "category → agency is the first-class relationship" decision.)*
- A watershed maintains a simple **list of agencies**; the reviewer picks from it.
- No routing-rules table, no spatial lookup, no jurisdiction engine to build.

```
Watershed:
  agencies[]          # the list the reviewer picks from

Report (on accept):
  selectedAgencies[]  # chosen by the reviewer, not computed
```

**Future step:** auto-*suggest* the agency by drawing **geographic bounding boxes /
areas on the map** per agency — the report's location falls in an area, that agency
is pre-selected. Still a suggestion the reviewer can change, never an automatic send.

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
- ~~How is jurisdiction determined?~~ **DECIDED:** the **reviewer selects the agency**
  at accept time (v1). Future: auto-*suggest* via per-agency bounding boxes/areas
  drawn on the map — a suggestion, never an automatic send.
- Do we need **escalation** (no acknowledgement in N days → notify next party)?
- Where do agency **credentials** (API keys, form quirks) live and who maintains them?
- Should reporters be able to **opt in to status updates** while staying anonymous
  (e.g. a claim code rather than an email)?
