---
name: workshop-mom-compliance
description: Stage 2 of the Schedule Analysis workshop. Fetch Workspace API schedule data, write the MOM Compliance Checklist block, and render the isolated MOM HTML report.
---

# Workshop MOM Compliance

## Purpose

Generate schedule-only MOM checks as a standalone stage. This skill owns the MOM stage end to end: fetch the Workspace API data it needs, format the MOM block, and refresh the report.

Do not require `schedule-context.json` to have been created by another stage. Reuse it only as a cache when it matches the requested range and already contains the data needed by this stage.

This skill is isolated. Do not require, read, preserve, or render cost, SPLH, or LQS block files, even if those files already exist from a previous run. Do not overwrite another stage's HTML file.

## Employee Access-Level Scope

This workshop targets StaffAny users whose access level is `Employee`.

Exclude users whose access level is `Owner` or `Manager` from all MOM checks, findings, assumptions tables, and report calculations. Treat those users as workshop runners or participants, not target employees.

If a scheduled slot is assigned to an excluded access-level user, ignore that slot for this report and do not treat it as unassigned or as a MOM risk. If access-level evidence is missing for a scheduled user, do not infer employee status from role or name; exclude the row from MOM checks and add a finding that access-level evidence is unavailable for that user.

## Inputs

- `start`: visible report start date, inclusive, `YYYY-MM-DD`.
- `end`: visible report end date, inclusive, `YYYY-MM-DD`.
- Workspace API key in the workshop folder. It starts with `wks_` and may be in `.env`, notes, or another local text file.
- StaffAny API JSON in the workshop folder, such as `api.json` or `api-1.json`, for endpoint reference.
- `reportDir`: default `reports/<start>-to-<end>`.

## Workspace API Fetch

Fetch the data needed by this stage directly:

- `/workspace/v1/orgs/current` for current organisation and timezone.
- `/workspace/v1/sections`.
- `/workspace/v1/roles`.
- `/workspace/v1/staff`, including access level, statutory, and employment fields.
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
reports/<start>-to-<end>/02-mom-compliance.html
```

## Checks

Use visible report-range schedule rows assigned to Employee access-level users for the MOM checks, except `72-hour monthly overtime cap`, which uses month-to-date schedule rows assigned to Employee access-level users as described below.

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

- `Breaks for shifts longer than 6 hours`: check Part 4 salary-qualified slots for Employee access-level users using scheduled gross hours. Because this workshop assumes every scheduled Employee access-level user qualifies for Part 4, treat every assigned Employee access-level slot as Part 4 salary-qualified. Mark `Pass` when no Part 4 salary-qualified slot exceeds 6 scheduled gross hours without break evidence. Mark `Review` when any Part 4 salary-qualified slot exceeds 6 scheduled gross hours without break evidence. Use `N/A` only when slot start/end or break evidence is not exposed.
- `72-hour monthly overtime cap`: fetch month-to-date schedule data from Workspace API and compute estimated overtime from scheduled paid hours for Employee access-level users only. Estimate overtime per staff member as scheduled paid hours above 44 hours per week, summed across the month-to-date schedule rows. Mark `Pass` when every included staff member is at or below `72` estimated overtime hours. Mark `Review` when any included staff member exceeds `72` estimated overtime hours. Mark `N/A` only when month-to-date schedule data cannot be fetched or is not exposed.

## Assumptions

- For the workshop model, assume every scheduled Employee access-level user qualifies for Part 4.
- For the workshop model, assume every role is non-workman. Do not classify `Chef`, `Barista`, `Waiter`, or any other role as workman.
- Do not use projected monthly salary, salary thresholds, nationality, contract type, actual duties, legal manager/executive status, or role names to disqualify Employee access-level users from Part 4 in this workshop stage. StaffAny `Manager` access level is excluded before Part 4 classification.
- Do not treat unassigned slots or slots assigned to non-Employee access-level users as a compliance risk.
- For `Rest-day work pay`, use compensation snapshot working-day settings as the workshop mapping when available.
- Mark `Rest-day work pay` as `Review` when Part 4-assumed staff are scheduled outside their inferred working-day pattern. Phrase this as rest-day or off-day pay review, not as confirmed rest-day pay due.

List the Part 4 classification assumptions at the bottom of the report for included Employee access-level users only, including staff, role, Part 4 qualifies, workman classification, and relevant info. Do not list Owner or Manager access-level users in the Part 4 table except for an optional excluded-scope note.

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

Write or replace only this stage's HTML file:

```text
reports/<start>-to-<end>/02-mom-compliance.html
```

The HTML must be a standalone static report rendered from `mom-compliance-checklist.json` and current context metadata. Do not display cost, SPLH, or LQS sections. If other block files exist in `artifacts/blocks/`, ignore them for this render. Do not create, overwrite, or delete `01-cost-and-splh.html` or `03-lqs-quota-analysis.html`.

## Validation

Before finishing:

1. Confirm `mom-compliance-checklist.json` exists.
2. Confirm `02-mom-compliance.html` contains `MOM Compliance Checklist`.
3. Confirm `02-mom-compliance.html` does not contain `Sales Per Labour Hour`, `Total estimated cost`, or `LQS Quota Analysis`.
4. Confirm Owner and Manager access-level users are absent from MOM checks and Part 4 tables, except for an optional excluded-scope note.
