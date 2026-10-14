# inploi Partner Dashboard

Static dashboard for the inploi partner programme. No build step, no dependencies.

| File | What it is |
|---|---|
| `index.html` | The whole dashboard — Board, Management and Miscellaneous views, self-contained |
| `vercel.json` | No-cache headers so a refresh shows immediately; `frame-ancestors` so the page can be embedded in HubSpot |

## How it stays current

An hourly scheduled task re-runs the HubSpot queries in the refresh runbook, patches
any figure that moved, and commits to `main`. Vercel redeploys on push. When nothing
has moved, nothing is committed — so the commit history is a log of what actually
changed, not a heartbeat.

Every definition, filter and field name behind these numbers lives in
`Partner_Dashboard_Report_Spec.md` in the inploi project docs. Two rules matter most:

- Partner revenue filters on **Closed Won Date**, never deal stage — won deals move to
  the Implementation pipeline, where `hs_is_closed_won` reads false.
- The new-business denominator comes from the **bookings sheet**, not HubSpot, because
  five of the six 2026 new logos carry Deal Source `Customer` or blank.

## Embedding in HubSpot

Link to the deployed URL rather than embedding it in an iframe. HubSpot frames
external content with `sandbox="allow-scripts"` only, which blocks single-click
navigation out of the page — tested across all four sandbox configurations.
