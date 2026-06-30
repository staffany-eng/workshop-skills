---
name: workshop-lqs-quota-analysis
description: Stage 3 of the Schedule Analysis workshop. Fetch Workspace API data for LQS quota analysis, write the LQS block, and render the isolated LQS HTML report.
---

# Workshop LQS Quota Analysis

## Purpose

Generate the LQS quota layer as a standalone stage. LQS is not a compliance checker in this report; it estimates local-worker quota contribution and near-tier opportunities.

This skill owns the LQS stage end to end. Do not require a previous stage to have fetched `schedule-context.json`; fetch and format the data needed for LQS before writing the LQS block.

This skill is isolated. Do not require, read, preserve, or render cost, SPLH, or MOM block files, even if those files already exist from a previous run. Do not overwrite another stage's HTML file.

## Employee Access-Level Scope

This workshop targets StaffAny users whose access level is `Employee`.

Exclude users whose access level is `Owner` or `Manager` from all LQS counts, wage projections, near-tier opportunities, findings, and report tables. Treat those users as workshop runners or participants, not target employees.

If scheduled, timesheet, or compensation rows belong to an excluded access-level user, ignore those rows for this report. If access-level evidence is missing for a user, do not infer employee status from role or name; exclude the user from LQS calculations and add a finding that access-level evidence is unavailable for that user.

## Inputs

- `start`: visible report start date, inclusive, `YYYY-MM-DD`.
- `end`: visible report end date, inclusive, `YYYY-MM-DD`.
- Workspace API key in the workshop folder. It starts with `wks_` and may be in `.env`, notes, or another local text file.
- StaffAny API JSON in the workshop folder, such as `api.json` or `api-1.json`, for endpoint reference.
- `reportDir`: default `reports/<start>-to-<end>`.

## Date Range Semantics

The visible report `start` and `end` are inclusive dates.

Workspace schedule endpoints use an exclusive end boundary. When fetching `/workspace/v2/shifts` and `/workspace/v2/shift-slots`, convert the visible `end` into `apiEnd = dayAfter(end)` and pass that as the endpoint `end` query value.

Example: for visible report range `2026-07-01` to `2026-07-31`, fetch schedule rows with `start=2026-07-01&end=2026-08-01`.

Keep the original inclusive `end` in output paths, report headings, and visible `schedule-context.json` metadata. If API fetch metadata is recorded, store the schedule API end separately as the exclusive `apiEnd`.

## Workspace API Fetch

Fetch the data needed by this stage directly:

- `/workspace/v1/orgs/current` for current organisation and timezone.
- `/workspace/v1/staff`, including access level and `sgStatutory.isPayingCpf`.
- `/workspace/v1/roles` and `/workspace/v1/sections`.
- `/workspace/v2/shifts` and `/workspace/v2/shift-slots` for the visible report range, using the exclusive API end date described above.
- `/workspace/v2/compensation-snapshots` for visible-range dates needed by the report.
- `/workspace/v2/shifts` and `/workspace/v2/shift-slots` for the selected LQS analysis month, using the exclusive API end date described above.
- `/workspace/v2/compensation-snapshots` for dates needed by the LQS monthly projection.
- `/workspace/v1/timesheets` for month-to-date work-hour rows when available.

Select the calendar month containing the visible report range. If the range crosses months, use the month with the most visible days.

Use month-to-date timesheet or work-hour rows when available and scheduled rows for the remaining-month projection. If timesheets are unavailable or empty, use scheduled rows as fallback and state that in `LQS Requirement Notes`.

Write or refresh `reports/<start>-to-<end>/artifacts/schedule-context.json` with `analysisMonth`, `lqsMonthRows`, and `timesheets` metadata used by this stage.

## Paid Hours For LQS Projection

When deriving paid hours from schedule rows or shift slots, use the same paid-hour rule as `workshop-cost-and-splh`:

- Join shift slots to their parent shifts.
- Calculate gross slot hours from `timeStart` to `timeEnd` in the organisation timezone.
- Break minutes come from shift data: use `slot.breakOverride` when present; otherwise use the parent shift `breakTime`.
- Paid hours are gross slot hours minus break hours from shift data; cap paid hours at zero.
- Use break-excluded paid hours, not gross hours, for LQS monthly wage projection, part-time hourly LQS checks, and near-tier additional paid-hour calculations.

If using timesheet or work-hour rows that already expose paid/worked hours, use the exposed paid/worked hours directly. When falling back to scheduled shift slots, always subtract shift break hours as described above.

## Output

Write or update:

```text
reports/<start>-to-<end>/artifacts/blocks/lqs-quota-analysis.json
reports/<start>-to-<end>/03-lqs-quota-analysis.html
```

## Required Cards

- `CPF-contributing Employees`: count of Employee access-level users where `sgStatutory.isPayingCpf === true`.
- `Number of 1.0 quota`: count of Employee access-level local workers projected at full LQS.
- `Number of 0.5 quota`: count of Employee access-level local workers projected at half LQS.
- `Near next quota tier`: count of scheduleable Employee access-level local hourly workers close to the next quota tier.

## Count Logic

- `1`: Employee access-level user, CPF-local proxy, and projected monthly gross is at or above full LQS.
- `0.5`: Employee access-level user, CPF-local proxy, and projected monthly gross is at least half LQS but below full LQS.
Apply count logic only after filtering to Employee access-level users. Excluded Owner and Manager access-level users are not counted as `0`.

- `0`: included Employee access-level user with no CPF-local proxy, missing wage projection, or projected monthly gross below half LQS.

Thresholds:

- before `2026-07-01`: full count `$1,600`, half count `$800`
- from `2026-07-01`: full count `$1,800`, half count `$900`

Near-tier opportunities must include both `0 -> 0.5` and `0.5 -> 1.0` moves. Show the criteria:

```text
Criteria: gap is within $100 monthly gross or within 8 additional paid hours.
```

## Proxies And Assumptions

- Use `sgStatutory.isPayingCpf === true` as the available StaffAny API proxy for Singapore Citizen / PR local employee status, after filtering to Employee access-level users.
- If `isPayingCpf === false`, quota count is `0`; pass type remains unknown unless explicitly tagged.
- In `LQS Staff Quota Detail`, explicitly show each included Employee's CPF contribution status, such as `CPF contributing: Yes` when `sgStatutory.isPayingCpf === true` and `CPF contributing: No` when it is false. This detail must make clear who is paying CPF and who is not.
- Check the part-time hourly LQS wage requirement for locals under 35 hours/week, but keep it separate from monthly quota count.
- Include MOM's requirement context for firms hiring foreign workers: they must pay PWM wages to local employees covered by relevant Sectoral or Occupational PWMs, and pay at least LQS to all local employees not covered under PWMs.
- For the workshop, assume one PWM category applies: food services. This is a modelling assumption, not an API-proven classification.
- Report the quota-counted local employee count as the quota input; do not calculate final foreign-worker slots unless sector rules are provided.

## LQS Block

Write `lqs-quota-analysis.json` with:

- `summaryCards`: `CPF-contributing Employees`, `Number of 1.0 quota`, `Number of 0.5 quota`, `Near next quota tier`
- `lqsStaffChecklist`
- `lqsConditions`
- `findings`
- `assumptions`
- `sources`

Each `lqsStaffChecklist` row must include an explicit CPF contribution field or label showing whether that Employee is CPF contributing.

Do not use compliance framing, quick-win slang, or hard-code demo staff names in user-facing output. Decide opportunities from the current data.

## Report Rendering

Write or replace only this stage's HTML file:

```text
reports/<start>-to-<end>/03-lqs-quota-analysis.html
```

The HTML must be a standalone static report rendered from `lqs-quota-analysis.json` and current context metadata. The LQS heading must include a hover/focus tooltip explaining proxies, count logic, and assumptions. Do not display cost, SPLH, or MOM sections. If other block files exist in `artifacts/blocks/`, ignore them for this render. Do not create, overwrite, or delete `01-cost-and-splh.html` or `02-mom-compliance.html`.

## Validation

Before finishing:

1. Confirm `lqs-quota-analysis.json` exists.
2. Confirm `03-lqs-quota-analysis.html` contains `LQS Quota Analysis`, `LQS Staff Quota Detail`, and `LQS Requirement Notes`.
3. Confirm `03-lqs-quota-analysis.html` does not contain `Sales Per Labour Hour`, `Total estimated cost`, or `MOM Compliance Checklist`.
4. Confirm Owner and Manager access-level users are absent from LQS staff detail and quota opportunity tables, except for an optional excluded-scope note.
5. Confirm `03-lqs-quota-analysis.html` displays `CPF-contributing Employees`, not `Quota-eligible local workers`.
6. Confirm `LQS Staff Quota Detail` explicitly shows who is CPF contributing and who is not.
7. Confirm scheduled paid hours used for LQS projections exclude shift break hours.
8. Confirm final visible day schedule rows are present when final-day shifts or shift slots exist.
9. Confirm `schedule-context.json` records the visible `end` as the inclusive report end date, and if API fetch metadata is recorded, that the schedule API end is recorded separately as the exclusive `apiEnd`.
10. Scan `03-lqs-quota-analysis.html` for stale labels: `LQS Compliance`, `low-hanging`, `1.0 local count staff`, `0.5 local count staff`, and `Quota-eligible local workers`.
