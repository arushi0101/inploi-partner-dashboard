# inploi Partner Dashboard

**Live:** https://inploi-partner-dashboard.vercel.app

Static dashboard for the inploi partner programme. No build step, no dependencies.

| File | What it is |
|---|---|
| `index.html` | The whole dashboard — Board, Management and Miscellaneous views, self-contained |
| `vercel.json` | No-cache headers so a refresh shows immediately |

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

**Do not add a `frame-ancestors` CSP here.** An earlier version set
`frame-ancestors 'self' https://*.hubspot.com ...` intending to permit HubSpot
embedding. It did the opposite: with no CSP a page is framable by default, and
`frame-ancestors` applies to the *whole* ancestor chain — HubSpot nests the embed
inside a sandboxed wrapper whose origin is `null`, which matches no host entry. The
result was "refused to connect" in HubSpot's embed preview.

Even with framing allowed, **link to the URL rather than embedding it**. HubSpot frames
external content with `sandbox="allow-scripts"` only, so single-click navigation out of
the page is impossible — tested across all four sandbox configurations.

## Access

This deployment is public: anyone with the URL can read it, including partner revenue,
deal values and customer names. Vercel deployment protection would close that, but
protection and HubSpot embedding are mutually exclusive — a protected page cannot be
framed. Pick one deliberately.
