# Creekdog — Agency Routing (design exploration)

> Status: **exploration**, not settled. This is the topic that shapes Phase 1 most.

## Problem statement
A citizen files a concern. Creekdog must get that concern to the **right agency**,
in whatever form that agency accepts. Two things vary independently:

1. **Who** a category routes to — a **per-watershed business decision**.
2. **How** a report physically reaches that agency — a **per-agency mechanism**.

Categories are *not* pre-seeded by Creekdog; each watershed defines its own and
attaches routing to them.

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
    autoSend          # false = require staff approval before send (recommended)
```

## Human-in-the-loop (recommended for v1)
Because submission is **anonymous**, spam and false reports are real. Recommendation:
**nothing auto-sends to a government agency without a staff "approve & send" click.**
- `manual` + `email` cover ~95% of real agencies without brittle automation.
- Automation (`open311`, `webhook`, auto-`email`) becomes an opt-in per agency once
  a watershed trusts its intake pipeline.

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
  against agency service-area boundaries?
- Do we need **escalation** (no acknowledgement in N days → notify next party)?
- Where do agency **credentials** (API keys, form quirks) live and who maintains them?
- Should reporters be able to **opt in to status updates** while staying anonymous
  (e.g. a claim code rather than an email)?
