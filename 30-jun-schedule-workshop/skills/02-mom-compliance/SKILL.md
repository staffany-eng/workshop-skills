---
name: workshop-mom-compliance
description: Stage 2 of the Schedule Analysis workshop. Fetch Workspace API schedule data, add the MOM Compliance Checklist block, and refresh the partial HTML report while preserving earlier blocks.
---

# Workshop MOM Compliance

## Purpose

Add schedule-only MOM checks after cost and SPLH are visible. This skill owns the MOM stage end to end: fetch the Workspace API data it needs, format the MOM block, and refresh the report.

Do not require `schedule-context.json` to have been created by another stage. Reuse it only as a cache when it matches the requested range and already contains the data needed by this stage.

## Inputs

- `start`: visible report start date, inclusive, `YYYY-MM-DD`.
- `end`: visible report end date, inclusive, `YYYY-MM-DD`.
- Workspace API key in the workshop folder. It starts with `wks_` and may be in `.env`, notes, or another local text file.
- StaffAny API JSON in the workshop folder, such as `api.json` or `api-1.json`, for endpoint reference.
- `reportDir`: default `reports/<start>-to-<end>`.

Keep any existing cost or SPLH block files already created by earlier stages.

## Workspace API Fetch

Fetch the data needed by this stage directly:

- `/workspace/v1/orgs/current` for current organisation and timezone.
- `/workspace/v1/sections`.
- `/workspace/v1/roles`.
- `/workspace/v1/staff`, including statutory and employment fields.
- `/workspace/v2/shifts` for the visible range with unpublished rows included when available.
- `/workspace/v2/shift-slots` for the visible range with unpublished and unassigned rows included when available.
- For `72-hour monthly overtime cap`, also fetch month-to-date shifts and shift slots for each calendar month touched by the visible report range. Use the first local day of the month through the last visible local date in that month.
- `/workspace/v2/compensation-snapshots` for each local date in the visible range.
- Public holiday or special-date rows exposed by Workspace API when the local API JSON exposes such a read endpoint. If unavailable, record that public holiday evidence is unavailable.

Write or refresh `reports/<start>-to-<end>/artifacts/schedule-context.json` with the data used by this stage. Do not use StaffAny support sessions or customer-app mutation endpoints.

## Output

Write or update:

```text
reports/<start>-to-<end>/artifacts/blocks/mom-compliance-checklist.json
reports/<start>-to-<end>/schedule-cost-report.html
```

## Checks

Use visible report-range schedule rows for the MOM checks, except `72-hour monthly overtime cap`, which uses month-to-date schedule rows as described below.

Include:

- `Part 4 coverage`
- `Maximum 12 hours worked in a day`
- `44-hour weekly / shift-work average threshold`
- `Normal daily hours screen`
- `Breaks for shifts longer than 6 hours`
- `Weekly rest day`
- `Maximum interval between rest days`
- `72-hour monthly overtime cap`
- `Rest-day work pay`
- `Public holiday scheduling`

Use checklist statuses:

- `Pass`
- `Review`
- `N/A`

Mark checks as `N/A` when required evidence is not exposed by the Workspace API, such as designated rest days. Prefer `Review` for schedule patterns that need HR interpretation.

Break and overtime status rules:

- `Breaks for shifts longer than 6 hours`: check Part 4 salary-qualified slots using scheduled gross hours. Because this workshop assumes every scheduled staff member qualifies for Part 4, treat every assigned slot as Part 4 salary-qualified. Mark `Pass` when no Part 4 salary-qualified slot exceeds 6 scheduled gross hours without break evidence. Mark `Review` when any Part 4 salary-qualified slot exceeds 6 scheduled gross hours without break evidence. Use `N/A` only when slot start/end or break evidence is not exposed.
- `72-hour monthly overtime cap`: fetch month-to-date schedule data from Workspace API and compute estimated overtime from scheduled paid hours. Estimate overtime per staff member as scheduled paid hours above 44 hours per week, summed across the month-to-date schedule rows. Mark `Pass` when every staff member is at or below `72` estimated overtime hours. Mark `Review` when any staff member exceeds `72` estimated overtime hours. Mark `N/A` only when month-to-date schedule data cannot be fetched or is not exposed.

## Assumptions

- For the workshop model, assume every scheduled staff member qualifies for Part 4.
- For the workshop model, assume every role is non-workman. Do not classify `Chef`, `Barista`, `Waiter`, or any other role as workman.
- Do not use projected monthly salary, salary thresholds, nationality, contract type, actual duties, manager/executive status, or role names to disqualify staff from Part 4 in this workshop stage.
- Do not treat unassigned slots as a compliance risk.
- For `Rest-day work pay`, use compensation snapshot working-day settings as the workshop mapping when available.
- Mark `Rest-day work pay` as `Review` when Part 4-assumed staff are scheduled outside their inferred working-day pattern. Phrase this as rest-day or off-day pay review, not as confirmed rest-day pay due.

List the Part 4 classification assumptions at the bottom of the report, including staff, role, Part 4 qualifies, workman classification, and relevant info.

## MOM Block

Write `mom-compliance-checklist.json` with:

- `id`: `momComplianceChecklist`
- `title`: `MOM Compliance Checklist`
- `checklist`
- `findings`
- `assumptions`
- `part4Classification`
- `sources`

Each checklist item should include lowercase machine status and report-facing label:

```json
{
  "status": "pass | review | na",
  "label": "Pass | Review | N/A",
  "title": "Maximum 12 hours worked in a day",
  "detail": "Plain-language schedule finding."
}
```

## Report Rendering

Refresh the HTML while preserving existing cost and SPLH sections when their block files exist. Do not display LQS until the LQS stage creates `lqs-quota-analysis.json`.

## Validation

Before finishing:

1. Confirm `mom-compliance-checklist.json` exists.
2. Confirm the HTML contains `MOM Compliance Checklist`.
3. Confirm earlier cost/SPLH sections are still present when their block files exist.
4. Confirm the HTML does not contain `LQS Quota Analysis` before the LQS stage runs.
