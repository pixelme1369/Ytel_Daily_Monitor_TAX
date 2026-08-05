# Ytel Daily Monitor — TAX

A standalone call-monitoring dashboard for the TAX company's Ytel dialer data.
Vanilla JS, no build step — SheetJS 0.18.5 + Chart.js 4.4.0 via CDN.

**Independent from `Ytel_Daily_Monitor_ADP`** (a different company/repo) — no shared
code, no shared CRM, no shared branding. This company has **no CRM merge**: it's
raw Ytel call-detail records only, so there's no Enrolled/Debt/Conversion tracking
here, same reasoning as that repo's `command-center` dashboard.

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

- **Overview** — Total Calls, Unique Numbers Received (inbound-unique), Contact
  Rate (outbound, >30s), Unique Calls >2/15/30 Min, Avg Talk Time, Drops, DNC
  Calls, Dialed After DNC. Tiles are styled like `Ytel_Daily_Monitor_v2`'s KPI
  grid (light cards, colored top accent bar, label/value/sub) by request — the
  one deliberately light-themed spot on an otherwise dark page; the rest of the
  dashboard is unchanged.
  - The three "Unique Calls >X Min" tiles use `phoneBest` — each distinct
    phone's single longest call in the range, **any direction** (not just
    outbound) — compared against `uniquePhonesAll` (every distinct phone
    touched in range, any direction). Same logic v2 uses for its own
    `>2 Min`/`>15 Min`/`>30 Min`/`>45 Min` tiles (`>45 Min` omitted here,
    not requested). This denominator is intentionally different from the
    Contact Rate tile's, which is outbound-only (`dialedUnique`).
- **Issues Detected** — compliance/ops alerts: dialed-after-DNC (critical), DNC
  call volume, drop rate, dead-call rate, agent short-call rate, excessive redials
  — thresholds editable in Settings
- **Call Volume & Compliance by Hour** — Calls / DNC / Drops by hour, bar charts
- **Disposition Breakdown** — status counts + %
- **Agent Performance** — per-agent call counts, talk-time brackets (Short ≤30s,
  <2m, 5–10m, 10–15m, 15–20m, 20–30m, 30m+), Avg/Total Talk, Drops, Redials
  (3+ outbound calls to the same number). **No role split** — every dialer is
  treated as one undifferentiated pool (per explicit product decision: "everyone
  is closer" for this company), so there's no Closer/Opener/Retention tagging
  or role editor here, unlike the sibling repo's dashboards.
- **Settings** — alert thresholds only (no role editor, see above)

## What's deliberately NOT here

Same reasoning as `command-center` in the sibling repo: no CRM data means no
ground truth for "did this lead close," so Enrolled/Debt/Conv%, CA-escrow
pending-deal logic, and Ytel Discrepancy are all dropped rather than approximated.
Also trimmed for this first version (can be added later if wanted): Agent
Rankings, Agent Scorecard, Campaign/Queue Breakdown, Top Numbers, VDCL/VDAD
Analysis, Missed Callbacks, DPC Drops, Incomplete/Received Transfers, Openers
Transfer Breakdown, Long Calls No Follow-up.

## Deploy

Static site, no build step — works from any static host (Vercel, GitHub Pages,
etc.). Just serve `index.html`.
