# Partner Dashboard — Hourly Refresh Runbook

**Owner:** Arushi Mittal · **Portal:** 25735731
**Repo:** `arushi0101/inploi-partner-dashboard`, branch `main`
**Live:** https://inploi-partner-dashboard.vercel.app — Vercel project `inploi-partner-dashboard` under scope `hey-1646s-projects`, auto-deploys on every push to `main`
**Cadence:** hourly · **Created:** 14 October 2026

This is the procedure the hourly scheduled task follows. It is deliberately narrow:
refresh the figures that move, patch them in place, commit only when something
actually changed, and report anything it cannot safely patch rather than guessing.

Definitions, field names and traps live in `Partner_Dashboard_Report_Spec.md`. Read it
before changing anything here.

---

## 0. Before anything else

1. `git clone --depth 1 https://github.com/arushi0101/inploi-partner-dashboard` — attach
   it first with `add_repo` (owner `arushi0101`, repo `inploi-partner-dashboard`,
   access `push`), then clone exactly once.
2. Read `data.json` from the clone. That is the last known state, and the thing to diff
   against. Never diff against the HTML.
3. If the clone or the HubSpot connector is unavailable, **stop and report**. Do not
   partially patch.

---

## 1. The queries

All via `query_crm_data` (SQL dialect). `FY` = calendar 2026; roll the dates each January.

| # | Figure | SQL |
|---|---|---|
| Q1 | Partner revenue YTD | `SELECT dealname, amount_in_home_currency, closed_won_date FROM DEAL WHERE discovery_meeting_channel = 'Partner' AND closed_won_date BETWEEN '2026-01-01' AND '2026-12-31'` |
| Q2 | Pipeline created FY26 | `SELECT dealname, amount_in_home_currency, dealstage, hs_is_closed, hs_is_closed_won FROM DEAL WHERE discovery_meeting_channel = 'Partner' AND createdate BETWEEN '2026-01-01' AND '2026-12-31' LIMIT 200` |
| Q3 | Open partner deals | Q2 filtered to `hs_is_closed = 'false'` |
| Q4 | Closed partner deals, rolling 12m | `SELECT dealname, amount_in_home_currency, closedate, COMPANY.name, COMPANY.company_type FROM DEAL WHERE discovery_meeting_channel = 'Partner' AND hs_is_closed = 'true' AND closedate BETWEEN '<today−12m>' AND '<today>'` |
| Q5 | Partner companies by stage | `SELECT name, partner_stage, agreement_in_place, partner_manager, partner_status FROM COMPANY WHERE company_type = 'Partner' LIMIT 200` |
| Q6 | Deals by year | `SELECT dealname, amount_in_home_currency, createdate FROM DEAL WHERE discovery_meeting_channel = 'Partner' LIMIT 300` |

**Two rules that must not be relaxed:**

- **Revenue filters on `closed_won_date`, never deal stage.** Won deals move to the
  Implementation pipeline, where `hs_is_closed_won` reads *false*. A stage filter
  returns £0 and silently hides every win.
- **`closed_won_date` was restamped by the pipeline migration on historic deals.** It is
  safe for partner deals (all post-date the field) and unsafe portal-wide. Never widen
  Q1 beyond `discovery_meeting_channel = 'Partner'`.

---

## 2. What is refreshed, and what is not

**Refreshed automatically** — a number changing here is normal and gets patched:

- Board tiles 1, 3, 4, 5 (partner revenue, pipeline created, open pipeline, producing of signed)
- Partner overview: days since last deal, deal count, win rate, open pipeline, YTD revenue
- Lifecycle stage counts
- Records-to-fix counts
- Deals by year, deals by month

**Not refreshed — left alone, and flagged if the underlying data moves:**

- **The new-business denominator (£273,030)** comes from the Secure bookings sheet, not
  HubSpot, because five of the six 2026 new logos carry Deal Source `Customer` or blank.
  Share of new customer revenue is recomputed as `partner revenue ÷ 273030` when the
  numerator moves. If a new partner-sourced new-business win appears, **flag it and stop**
  — the denominator needs a human to update it from the sheet.
- **Narrative prose, flags and the insight panel.** If a number inside a sentence is
  contradicted by new data, flag it; do not rewrite the sentence.
- **Win rate `n` values.** These stay low because 12 partner-sourced losses carry no
  partner company. Do not "fix" this by counting unassociated deals.

---

## 3. Patching

1. Recompute every field in `data.json` from the queries.
2. Diff against the committed `data.json`. **No differences → stop here. Commit nothing.**
3. For each changed field, patch `index.html` by exact-string replacement, anchored on
   the surrounding markup. Each figure appears in more than one place — the scorecard
   tile, the card body, the dropdown rows, sometimes the prose — so replace *all*
   occurrences of a figure, and verify the count of replacements matches expectation
   before writing.
4. Table rows carry `data-v="<raw number>"` for sorting. Patch the attribute and the
   visible text together or sorting breaks.
5. Update the header stamp to **GMT and London local time — never IST**:
   `<span class="asat" id="asat">live &middot; refreshed <D Month YYYY>, <HH:MM> GMT &middot; <HH:MM> London</span>`
   London is GMT in winter and BST (GMT+1) from late March to late October, so the two
   differ for most of the year. Print both; do not collapse them when they match.
6. Write the new `data.json`.

**Verify before committing.** Open the patched file headlessly and confirm: no JS errors,
the view switcher still moves between all three views, every KPI dropdown opens, and
every anchor still points at `app-eu1.hubspot.com`. If any check fails, **discard the
patch and report** — a broken dashboard is worse than a stale one.

---

## 4. Committing

Commit `index.html` and `data.json` together on `main`, with a message naming what moved:

```
Refresh: open pipeline £77,100 → £91,400, 11 → 12 deals

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
```

Push to `main`. Vercel redeploys automatically. **Never force-push**; never rewrite history.

---

## 4a. Slack — every run, without exception

After the commit decision is made, DM the result to Arushi on Slack (`channel_id` is her
user id `U0BJTG9SECB`). **Send one every hour whether or not anything moved** — she asked
for the heartbeat, so silence is not an acceptable outcome.

Three shapes:

**Nothing moved**
> *Partner dashboard · 14 Oct, 09:00 GMT · 10:00 London*
> No change. Partner revenue £87,000, open pipeline £77,100 across 11 deals.

**Something moved and was patched**
> *Partner dashboard · updated*
> Open pipeline £77,100 → £91,400, 11 → 12 deals. New: Acme Corp via Beamery, £14,300 at Discovery.
> Live: https://inploi-partner-dashboard.vercel.app · commit `abc1234`

**Stopped — needs a person** (any §5 case)
> *Partner dashboard · not updated, needs you*
> <what was found, and why it was not patched>
> Nothing was committed; the dashboard still shows <date> figures.

Keep it to a few lines. Name figures, not process. Always include the live URL when
something changed. If the Slack send fails, say so in the run's final line — never let a
failed notification pass as a quiet run.

---

## 5. When to stop and report instead of patching

Stop, commit nothing, and report:

- A new partner-sourced **new-business win** — the denominator needs the bookings sheet.
- A **new partner company** appearing in the overview set, or an existing one dropping out.
- A deal named `Test`, `TEST`, `Demo`, `Dummy` or similar, or one attached to a company that
  is not `company_type = Partner`, carrying Deal Source = Partner. These are test records.
  **Never put one on the board.** Report it for deletion or re-sourcing.
- A deal that was partner-sourced in the last snapshot and now has a blank or changed Deal
  Source. That silently removes it from every partner report and can drop a partner out of
  the productive count.
- Any verification check in §3 failing.
- A figure moving more than 50% in one hour — almost always a data-entry error or a
  migration in progress, not a real change.
- The HubSpot connector returning an error, or a query returning zero rows where the
  baseline had many.

A stale dashboard with an explanation beats a confidently wrong one.

---

## 6. Known limits

- **Hourly is faster than this data moves.** Over the fortnight to 13 October the partner
  deal count changed three times. Most runs will find nothing and commit nothing, which
  is the intended behaviour.
- **Embedding the page in HubSpot breaks single-click navigation.** HubSpot frames
  external content with `sandbox="allow-scripts"` only. Link to the Vercel URL instead.
- **Reports and dashboards cannot be created through the HubSpot connector** — read only.
