# Ytel Daily Monitor — TAX

A standalone call-monitoring dashboard for the TAX company's Ytel dialer data.
Vanilla JS, no build step — SheetJS 0.18.5 + Chart.js 4.4.0 via CDN. Full feature
parity with the sibling repo's `command-center` dashboard (same section set),
plus a couple of TAX-specific additions (see below).

**Independent from `Ytel_Daily_Monitor_ADP`** (a different company/repo) — no shared
code, no shared CRM, no shared branding, own GitHub repo. This company has **no
CRM merge**: it's raw Ytel call-detail records only, so there's no Enrolled/Debt/
Conversion tracking anywhere in this dashboard, same reasoning as that repo's
`command-center` dashboard.

## Data source

**Manual upload for now** — drag/drop or click the dropzone in the top bar to load
one or more `.csv`/`.xlsx` Ytel exports (multi-file merges into one dataset), pick
a date range, click **Run Analysis**. `autoSetDate()` sets the date pickers to the
min/max date found across whatever you uploaded.

A live BigQuery data source (like the sibling repo's `command-center` and v2
dashboards use) can be added later the same way — a `/api/calls` serverless
function plus a `GCP_SERVICE_ACCOUNT_KEY` env var — once a BigQuery table exists
for this company's call log. Ask for that when the table's ready; the frontend's
`normalize()` already expects the same column names the BigQuery route would
return, so wiring it in is a small, additive change.

## Expected columns (Ytel export)

```
call_date, direction, phone_number_dialed, status, user, full_name, campaign_id,
call_list_id, vendor_lead_code, source_id, list_id, gmt_offset_now, phone_code,
phone_number, title, first_name, middle_initial, last_name, address1, address2,
address3, city, state, province, postal_code, country_code, gender, date_of_birth,
alt_phone, email, security_phrase, comments, length_in_sec, talk_sec, dispo_sec,
dead_sec, user_group, alt_dial, rank, owner, lead_id, list_name, list_description,
status_name, recording_location
```

Only a subset is currently read by the dashboard: `call_date`, `direction`,
`phone_number_dialed`/`phone_number`, `status`, `status_name`, `user`, `full_name`,
`campaign_id`, `source_id`, `length_in_sec`, `recording_location`. The rest (lead
demographics, `list_name`, `talk_sec`/`dispo_sec`/`dead_sec`, etc.) are harmless if
present — just unused for now — and available to wire up later if a report needs
them (e.g. `list_name` for campaign-readable labels, or `talk_sec` if `length_in_sec`
ever proves unreliable).

## Sections

Same set as `command-center` in the sibling repo, in nav order:

- **Overview** — Total Calls, Unique Numbers Received (inbound-unique), Contact
  Rate (outbound, >30s), **Unique Calls >2/15/30 Min** (TAX-specific addition, see
  below), Avg Talk Time, Drops, DNC Calls, Dialed After DNC. Tiles are styled like
  `Ytel_Daily_Monitor_v2`'s KPI grid (light cards, colored top accent bar,
  label/value/sub) by request — the one deliberately light-themed spot on an
  otherwise dark page; the rest of the dashboard stays dark.
  - The three "Unique Calls >X Min" tiles use `phoneBest` — each distinct
    phone's single longest call in the range, **any direction** (not just
    outbound) — compared against `uniquePhonesAll` (every distinct phone
    touched in range, any direction). Same logic v2 uses for its own
    `>2 Min`/`>15 Min`/`>30 Min`/`>45 Min` tiles (`>45 Min` omitted here,
    not requested). This denominator is intentionally different from the
    Contact Rate tile's, which is outbound-only (`dialedUnique`).
  - **Direction filter** (TAX-specific addition): All / Inbound / Outbound
    buttons above the KPI grid (`setKpiDir()`/`computeKpiData()`/
    `renderKpisForDir()`) re-filter just the Overview tiles by call direction.
    Nothing else on the page (Issues, hour charts, Agent Performance,
    Rankings, Scorecard, Campaign Breakdown, etc.) is affected — those always
    reflect the full, all-direction date range.
- **Issues Detected** — compliance/ops alerts: dialed-after-DNC (critical), DNC
  call volume, drop rate, dead-call rate, agent short-call rate, excessive redials
  — thresholds editable in Settings
- **Call Volume & Compliance by Hour** — Calls / DNC / Drops by hour, bar charts
- **Disposition Breakdown** — status counts + %
- **Agent Performance** — per-agent call counts, talk-time brackets (Short ≤30s,
  <2m, 5–10m, 10–15m, 15–20m, 20–30m, 30m+), Avg/Total Talk, Drops, Redials
  (3+ outbound calls to the same number)
- **Agent Rankings** — By Volume / By Contact Rate / By Drop Rate
- **Agent Scorecard** — one card per agent: call count, avg talk, drop rate,
  talk-time bracket bar, color-coded verdict badge (opener verdict uses
  transfer rate; everyone else uses the Long Calls No Follow-up rate at a
  fixed 20-min threshold)
- **Campaign / Queue Breakdown** — Contact% per campaign
- **Top 5 Dialed Numbers**
- **VDCL / VDAD Analysis** — dispositions dialed by the predictive dialer itself
- **Missed Callbacks** — inbound `TIMEOT` with no follow-up
- **DPC — Dropped, Never Called Back** — per-event drop-with-no-follow-up flagging
- **Transfers** — Incomplete Transfers (CLtrns with no inbound follow-up) +
  Correct Transfers Received by Agent
- **Openers — Transfer Breakdown** — by talk-time bracket, plus a Lowest
  Transfer Rate (>2min) ranking. Empty/hidden until agents are tagged as
  Openers in Settings (see below)
- **Long Calls, No Follow-up** — closer-scoped, selectable 20/25/30-min
  threshold, CSV export
- **Settings** — Agent role editor (Closers/Openers/Retention) + alert
  thresholds

### Agent roles

Unlike the sibling repo's dashboards, **no agent names are pre-seeded** —
`DEFAULT_ROLES` ships empty (`{closer:[], opener:[], retention:[]}`) since this
company's roster isn't known in advance. Until you fill in Settings, every agent
is untagged (no Closer/Opener/Retention pill) and treated the same everywhere —
Agent Scorecard's non-opener branch and the Long Calls No Follow-up logic apply
to everyone, and the Openers Transfer Breakdown card just stays empty. Add names
via the Settings role editor (persists to `localStorage`, same pattern as the
sibling repo) whenever you're ready to split them out.

## What's deliberately NOT here

Same reasoning as `command-center` in the sibling repo: no CRM data means no
ground truth for "did this lead close," so Enrolled/Debt/Conv%, CA-escrow
pending-deal logic, and Ytel Discrepancy are all dropped rather than approximated.

## Deploy

Static site, no build step — works from any static host (Vercel, GitHub Pages,
etc.). Just serve `index.html`.
