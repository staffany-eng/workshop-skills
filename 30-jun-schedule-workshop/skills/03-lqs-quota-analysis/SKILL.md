---
name: workshop-lqs-quota-analysis
description: Stage 3 of the Schedule Analysis workshop. Fetch Workspace API data for LQS quota analysis, write the LQS block, and refresh the final HTML report.
---

# Workshop LQS Quota Analysis

## Purpose

Add the LQS quota layer after attendees understand cost, SPLH, and MOM checks. LQS is not a compliance checker in this report; it estimates local-worker quota contribution and near-tier opportunities.

This skill owns the LQS stage end to end. Do not require a previous stage to have fetched `schedule-context.json`; fetch and format the data needed for LQS before writing the LQS block.

## Inputs

- `start`: visible report start date, inclusive, `YYYY-MM-DD`.
- `end`: visible report end date, inclusive, `YYYY-MM-DD`.
- Workspace API key in the workshop folder. It starts with `wks_` and may be in `.env`, notes, or another local text file.
- StaffAny API JSON in the workshop folder, such as `api.json` or `api-1.json`, for endpoint reference.
- `reportDir`: default `reports/<start>-to-<end>`.

Keep existing cost, SPLH, and MOM block files when present.

## Workspace API Fetch

Fetch the data needed by this stage directly:

- `/workspace/v1/orgs/current` for current organisation and timezone.
- `/workspace/v1/staff`, including `sgStatutory.isPayingCpf`.
- `/workspace/v1/roles` and `/workspace/v1/sections`.
- `/workspace/v2/shifts` and `/workspace/v2/shift-slots` for the visible report range.
- `/workspace/v2/compensation-snapshots` for visible-range dates needed by the report.
- `/workspace/v2/shifts` and `/workspace/v2/shift-slots` for the selected LQS analysis month.
- `/workspace/v2/compensation-snapshots` for dates needed by the LQS monthly projection.
- `/workspace/v1/timesheets` for month-to-date work-hour rows when available.

Select the calendar month containing the visible report range. If the range crosses months, use the month with the most visible days.

Use month-to-date timesheet or work-hour rows when available and scheduled rows for the remaining-month projection. If timesheets are unavailable or empty, use scheduled rows as fallback and state that in `LQS Requirement Notes`.

Write or refresh `reports/<start>-to-<end>/artifacts/schedule-context.json` with `analysisMonth`, `lqsMonthRows`, and `timesheets` metadata used by this stage.

## Output

Write or update:

```text
reports/<start>-to-<end>/artifacts/blocks/lqs-quota-analysis.json
reports/<start>-to-<end>/schedule-cost-report.html
```

## Required Cards

- `Quota-eligible local workers`: count of CPF-local workers.
- `Number of 1.0 quota`: count of local workers projected at full LQS.
- `Number of 0.5 quota`: count of local workers projected at half LQS.
- `Near next quota tier`: count of scheduleable local hourly workers close to the next quota tier.

## Count Logic

- `1`: CPF-local proxy and projected monthly gross is at or above full LQS.
- `0.5`: CPF-local proxy and projected monthly gross is at least half LQS but below full LQS.
- `0`: not CPF-local proxy, missing wage projection, or projected monthly gross below half LQS.

Thresholds:

- before `2026-07-01`: full count `$1,600`, half count `$800`
- from `2026-07-01`: full count `$1,800`, half count `$900`

Near-tier opportunities must include both `0 -> 0.5` and `0.5 -> 1.0` moves. Show the criteria:

```text
Criteria: gap is within $100 monthly gross or within 8 additional paid hours.
```

## Proxies And Assumptions

- Use `sgStatutory.isPayingCpf === true` as the available StaffAny API proxy for Singapore Citizen / PR local employee status.
- If `isPayingCpf === false`, quota count is `0`; pass type remains unknown unless explicitly tagged.
- Check the part-time hourly LQS wage requirement for locals under 35 hours/week, but keep it separate from monthly quota count.
- Include MOM's requirement context for firms hiring foreign workers: they must pay PWM wages to local employees covered by relevant Sectoral or Occupational PWMs, and pay at least LQS to all local employees not covered under PWMs.
- For the workshop, assume one PWM category applies: food services. This is a modelling assumption, not an API-proven classification.
- Report the quota-counted local employee count as the quota input; do not calculate final foreign-worker slots unless sector rules are provided.

## LQS Block

Write `lqs-quota-analysis.json` with:

- `summaryCards`: `Quota-eligible local workers`, `Number of 1.0 quota`, `Number of 0.5 quota`, `Near next quota tier`
- `lqsStaffChecklist`
- `lqsConditions`
- `findings`
- `assumptions`
- `sources`

Do not use compliance framing, quick-win slang, or hard-code demo staff names in user-facing output. Decide opportunities from the current data.

## Report Rendering

Refresh the HTML while preserving previous sections. The LQS heading must include a hover/focus tooltip explaining proxies, count logic, and assumptions.

## Validation

Before finishing:

1. Confirm `lqs-quota-analysis.json` exists.
2. Confirm the HTML contains `LQS Quota Analysis`, `LQS Staff Quota Detail`, and `LQS Requirement Notes`.
3. Confirm previous cost/SPLH/MOM sections are preserved when their block files exist.
4. Scan the HTML for stale labels: `LQS Compliance`, `low-hanging`, `1.0 local count staff`, and `0.5 local count staff`.
