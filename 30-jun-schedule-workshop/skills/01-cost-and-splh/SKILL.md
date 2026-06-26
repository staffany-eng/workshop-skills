---
name: workshop-cost-and-splh
description: Stage 1 of the Schedule Analysis workshop. Fetch Workspace API data, compute schedule cost and Sales Per Labour Hour, write block artifacts, and render the isolated cost/SPLH HTML report.
---

# Workshop Cost And SPLH

## Purpose

Generate the first analysis layer for the workshop: schedule cost and Sales Per Labour Hour. This skill owns the full Stage 1 flow from Workspace API fetch to block JSON and isolated HTML output.

Do not defer cost or SPLH logic to another skill. Use the rules in this file to produce useful Stage 1 output end to end.

This skill is isolated. Do not require or render any block file from MOM or LQS stages, even if those files already exist from a previous run. Do not overwrite another stage's HTML file.

## Employee Access-Level Scope

This workshop targets StaffAny users whose access level is `Employee`.

Exclude users whose access level is `Owner` or `Manager` from all cost, paid-hour, SPLH, findings, and report-table calculations. Treat those users as workshop runners or participants, not target employees.

If a scheduled slot is assigned to an excluded access-level user, ignore that slot for this report and do not treat it as unassigned, missing compensation, or a labour-cost issue. If access-level evidence is missing for a scheduled user, do not infer employee status from role or name; exclude the row from primary calculations and add a finding that access-level evidence is unavailable for that user.

## Inputs

- `start`: visible report start date, inclusive, `YYYY-MM-DD`.
- `end`: visible report end date, inclusive, `YYYY-MM-DD`.
- Workspace API key in the workshop folder. It starts with `wks_` and may be in `.env`, notes, or another local text file.
- StaffAny API JSON in the workshop folder, such as `api.json` or `api-1.json`, for endpoint reference.
- `reportDir`: default `reports/<start>-to-<end>`.

Do not ask for the organisation name. Resolve the organisation from the Workspace API key.

## Workspace API Fetch

Fetch the data needed by this stage directly. Do not require a previous data-fetch stage.

Use Workspace API only:

- `/workspace/v1/orgs/current` for current organisation and timezone.
- `/workspace/v1/sections`.
- `/workspace/v1/roles`.
- `/workspace/v1/staff`, including access level, statutory, and compensation-relevant fields.
- `/workspace/v2/shifts` for the visible range with unpublished rows included when available.
- `/workspace/v2/shift-slots` for the visible range with unpublished and unassigned rows included when available.
- `/workspace/v2/compensation-snapshots` for each local date in the visible range.
- `/workspace/v2/sales` with `granularity=day` for section-level target sales when available.

Write or refresh:

```text
reports/<start>-to-<end>/artifacts/schedule-context.json
reports/<start>-to-<end>/report-params.json
```

The context should include `org`, `range`, `sections`, `roles`, `staff`, `shifts`, `shiftSlots`, `compensationSnapshotsByDate`, `sales`, and `generatedAt`. Keep enough staff access-level evidence to explain which users were included or excluded.

## Outputs

Write or update:

```text
reports/<start>-to-<end>/artifacts/blocks/schedule-cost-estimate.json
reports/<start>-to-<end>/artifacts/blocks/sales-per-labour-hour.json
reports/<start>-to-<end>/01-cost-and-splh.html
```

Do not write MOM or LQS block files in this stage.

## Schedule Cost Rules

Use rows from `shiftSlots`. Join each slot to:

- `shifts` by `shiftId` for parent break time and shift metadata.
- `staff` by `userId`; continue only when the joined staff user's access level is `Employee`.
- `roles` by `roleId`.
- `sections` by `sectionId`.
- `compensationSnapshotsByDate[localShiftDate].items` by `staffId`.

Calculate:

- Gross slot hours from `timeStart` to `timeEnd` in the organisation timezone.
- Break minutes from shift data: use `slot.breakOverride` when present; otherwise use the parent shift `breakTime`.
- Paid hours as gross slot hours minus break hours from shift data; cap paid hours at zero. Use paid hours, not gross hours, for cost totals, summary cards, section allocation, role breakdown, and SPLH.

### Compensation Type Handling

- Treat a row as salaried only when `compensation.type` is explicitly salaried, for example `SALARIED`.
- Do not use `compensation.cycle` to decide whether a row is salaried.
- `type: HOURLY` with `cycle: MONTHLY` is still hourly. Use `compensation.rate` directly as the hourly rate.
- For hourly rows and rows that are not explicitly salaried, calculate staff cost as `paidHours * compensation.rate`.
- Apply `(compensation.rate * 12) / (52 * 44)` only when `compensation.type` is explicitly salaried.
- Do not infer salaried status from role, access level, pay amount, missing compensation type, or `compensation.cycle`. Use `44` as the fixed weekly-hour divisor for explicitly salaried compensation in this workshop model.

Examples:

- `{ type: "HOURLY", cycle: "MONTHLY", rate: 24.5 }` means `$24.50/hour`.
- `{ type: "SALARIED", cycle: "MONTHLY", rate: 4200 }` means monthly salary, converted to hourly as `(4200 * 12) / (52 * 44)`.

Exclude unassigned slots and slots assigned to non-Employee access-level users from primary staff cost totals. Do not add a finding solely because unassigned slots or Owner/Manager slots exist, and do not treat them as a compliance risk.

If compensation is missing or cannot produce a usable hourly equivalent, surface a finding. Do not invent a rate.

## Schedule Cost Block

Write `schedule-cost-estimate.json` with:

- `id`: `scheduleCostEstimate`
- `summaryCards`: `Total estimated cost`, `Shift slots`, `Paid hours`
- `dailyCostCurve`
- `sectionAllocation`
- `compensationMix`
- `roleBreakdown`
- `findings`
- `assumptions`

Chart/table arrays should include `name`, `slots`, `hours`, and `cost`. Base primary cost totals, charts, and tables on assigned Employee access-level staff rows with usable compensation.

## SPLH Rules

Use Workspace API target sales rows, the computed schedule-cost block, and scheduled paid hours. SPLH is a section-level concept in this workshop, so calculate and present it by section, not only as a whole-report or whole-day metric.

- Use British spelling in user-facing output: `labour`.
- The default labour-cost target is `30%` of sales.
- Use target sales only.
- Calculate section scheduled paid hours from assigned Employee access-level shift slots grouped by `sectionId`.
- Calculate section labour cost from assigned Employee access-level shift slots grouped by `sectionId`.
- Calculate `Target SPLH = sectionTargetSales / sectionScheduledPaidHours`.
- Calculate `Required SPLH for 30% labour cost = sectionAverageLabourCostPerHour / 0.30`.
- Calculate `Labour cost % (projected) = sectionLabourCost / sectionTargetSales`.
- A section needs schedule-cost review only when projected `Labour cost % > 30%`.
- If target sales data is missing for a section, keep the section row, mark SPLH values unavailable, and surface the missing target sales in `findings`.

Do not add a `Days above 30%` summary card. The section table already shows which sections exceed target.

## SPLH Block

Write `sales-per-labour-hour.json` with:

- `id`: `salesPerLabourHour`
- `title`: `Sales Per Labour Hour`
- `targetLabourCostPercent`: `30`
- `summaryCards`
- `sections`
- `sectionDaily`
- `findings`
- `assumptions`

Each `sections` row should include:

- `sectionId`
- `section`
- `targetSales`
- `paidHours`
- `labourCost`
- `targetSplh`
- `splhFloorAtTarget`
- `labourCostPercent`
- `recommendation`

Each `sectionDaily` row should include:

- `date`
- `sectionId`
- `section`
- `targetSales`
- `paidHours`
- `labourCost`
- `targetSplh`
- `splhFloorAtTarget`
- `labourCostPercent`
- `recommendation`

Do not include `name` in SPLH rows. Use `date` only for section daily detail rows.

## Report Rendering

After writing both block files, write or replace only this stage's HTML file:

```text
reports/<start>-to-<end>/01-cost-and-splh.html
```

The HTML must be a standalone static report rendered from the current stage data. It must include only:

- `Schedule Analysis` header with organisation, timezone, and visible report range.
- Cost summary cards.
- `Section Allocation`.
- `Compensation Mix`.
- `Role Breakdown`.
- `Sales Per Labour Hour`, with section-level summary cards, `Daily Cost Curve` collapsed into this section, and a section-by-section SPLH table.
- Stage 1 findings and assumptions.

Render `Daily Cost Curve` from `schedule-cost-estimate.dailyCostCurve` as a subsection under `Sales Per Labour Hour`, not as a separate top-level report section.

Do not display MOM or LQS sections, and do not read or preserve their block files during rendering. Do not show empty placeholders for later stages. Do not create, overwrite, or delete `02-mom-compliance.html` or `03-lqs-quota-analysis.html`.

## Validation

Before finishing:

1. Confirm `schedule-context.json` and both Stage 1 block files exist.
2. Confirm `01-cost-and-splh.html` contains `Total estimated cost` and `Sales Per Labour Hour`.
3. Confirm `01-cost-and-splh.html` does not contain `MOM Compliance Checklist`, `LQS Quota Analysis`, or empty later-stage placeholders.
4. Confirm Owner and Manager access-level users are absent from cost and SPLH tables, except for an optional excluded-scope note.
5. Confirm paid hours exclude shift break hours.
6. Confirm salaried conversion is used only for rows whose compensation type is explicitly salaried.
7. Confirm `type: HOURLY` rows are not converted with the salaried formula, even when `cycle: MONTHLY`.
8. Summarize total estimated cost, shift slots, paid hours, and projected labour cost percentage.
9. Confirm the SPLH table is section-level and contains no actual-sales labels or values.
10. Confirm `Daily Cost Curve` appears only inside `Sales Per Labour Hour`, not as its own top-level report section.
