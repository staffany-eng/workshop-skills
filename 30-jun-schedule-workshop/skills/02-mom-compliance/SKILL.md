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

This workshop targets StaffAny users whose access level is `Employee`; call them `included employees` in the rest of this skill.

Exclude users whose access level is `Owner`, `Manager`, or unknown from all MOM checks, findings, assumptions tables, and report calculations. Treat those users as workshop runners or participants, not target employees.

If a scheduled slot is assigned to an excluded access-level user, ignore that slot for this report and do not treat it as unassigned or as a MOM risk. If access-level evidence is missing for a scheduled user, do not infer employee status from role or name; exclude the row from MOM checks and add a finding that access-level evidence is unavailable for that user.

StaffAny `Employee` access level is only a workshop scope filter. It is not legal Part 4 classification.

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
- For `72-hour monthly overtime cap`, also fetch shifts and shift slots for the full calendar month touched by the visible report range. Use the first local day of the month through the last local day of the month. If the visible range touches multiple months, fetch each full month.
- `/workspace/v2/compensation-snapshots` for every local date needed by visible-range checks and full-month overtime checks. Use the snapshot effective for each slot's local date when reading `dailyHours`, `weeklyHours`, and any exposed scheduled overtime fields.
- Day-off rows from Workspace API when the local API JSON exposes a read endpoint such as `/workspace/v1/dayOffs`. Use these only as supporting evidence; rest-day grading is based on no-shift local dates from schedule rows.

Write or refresh `reports/<start>-to-<end>/artifacts/schedule-context.json` with the data used by this stage. Do not use StaffAny support sessions or customer-app mutation endpoints.

## Output

Write or update:

```text
reports/<start>-to-<end>/artifacts/blocks/mom-compliance-checklist.json
reports/<start>-to-<end>/02-mom-compliance.html
```

## Checks

Use visible report-range schedule rows for included employees for the visible-range MOM checks, except `72-hour monthly overtime cap`, which uses full-calendar-month schedule rows for included employees as described below.

Include:

- `Part 4 coverage`
- `Maximum 12 hours worked in a day`
- `44-hour weekly / shift-work average threshold`
- `Breaks for shifts longer than 6 hours`
- `Weekly rest day`
- `Maximum interval between rest days`
- `72-hour monthly overtime cap`

Use checklist statuses:

- `Pass`
- `Review`
- `N/A`

Mark checks as `N/A` only when neither primary evidence nor the stated fallback evidence is available. Prefer `Review` for schedule patterns that need HR interpretation.

## Counting Rules

All hour-based checks must use organisation-local dates. Use "scheduled paid hours" in report copy whenever the source is future schedule data. Do not describe scheduled future rows as confirmed worked hours.

Scheduled paid hours are:

```text
slot paid hours = max(0, slot timeEnd - slot timeStart - positive break evidence)
```

Positive break evidence can come from, in order:

1. slot break override minutes
2. slot break start/end
3. parent shift break minutes
4. parent shift break start/end

When no positive break evidence exists, use `0` break hours for paid-hour totals and let the break-specific check decide whether review is needed.

Every included employee drilldown row must contain one per-check result for each checklist item, including concise metrics, thresholds, statuses, details, and evidence rows. Do not render raw fetched JSON in HTML. Per-check `Drilldown` bullets below describe only the check-specific evidence to include.

### Part 4 coverage

- Source: `/workspace/v1/staff` access-level evidence and assigned schedule rows in the visible report range.
- Count: count scheduled staff by access-level scope: included `Employee`, excluded `Owner`/`Manager`, and excluded unknown access level.
- Threshold: no legal Part 4 threshold is applied in this workshop stage.
- Status: always `N/A`.
- Reason: this workshop scopes by StaffAny access level, not by legal Part 4 classification.
- Drilldown: show each scheduled staff member's access level, included/excluded decision, Part 4 status `N/A`, workman classification `not assessed`, and excluded reason where applicable.
- Detail: show included Employee count, excluded Owner/Manager count, missing-access-level count, and a caveat that Employee access level is not legal Part 4 coverage.

### Maximum 12 hours worked in a day

- Source: visible report-range assigned slots for included employees.
- Count: group scheduled paid hours by `userId` and organisation-local date.
- Threshold: `12` scheduled paid hours per employee per local date.
- Status: `Review` if any employee-date total is greater than `12`; otherwise `Pass`.
- Drilldown: show daily scheduled paid hours per included employee, source slots, break evidence, threshold, and employee-date result.
- Detail: say "scheduled paid hours" and identify each employee-date over the threshold.

### 44-hour weekly / shift-work average threshold

- Source: visible report-range assigned slots for included employees.
- Count: sum scheduled paid hours by employee across the visible report range.
- Threshold: `44` scheduled paid hours.
- Status: `Review` if any included employee is greater than `44`; otherwise `Pass`.
- Drilldown: show each included employee's visible-range scheduled paid hours, source slots, threshold, and result.
- Do not use the StaffAny daily or weekly OT-after settings for this check. This check is a fixed 44-hour schedule screen.

### Breaks for shifts longer than 6 hours

- Source: visible report-range assigned slots for included employees.
- Count: use scheduled gross hours for the greater-than-6-hour trigger, before subtracting breaks. Break evidence must be positive and come from the same evidence priority used for scheduled paid hours.
- Threshold: greater than `6` scheduled gross hours without positive break evidence.
- Status: `Review` if any included employee slot exceeds `6` scheduled gross hours without positive break evidence; otherwise `Pass`. Use `N/A` only when slot start/end or break fields are not exposed.
- Drilldown: show each included employee's slots longer than 6 scheduled gross hours, break evidence, threshold, and slot result.
- Detail: identify employee, date, slot start/end, scheduled gross hours, and break evidence used.

### Weekly rest day

- Source: visible report-range assigned schedule rows. Day-off rows may be fetched as supporting evidence only.
- Count: for each included employee, treat every organisation-local date in the visible report range with no assigned scheduled slot as a no-shift rest date.
- Threshold: at least one no-shift rest date per included employee in the visible report range.
- Status: `Review` if an included employee has no no-shift rest date in the visible report range; otherwise `Pass`. If an included employee has zero assigned slots in the visible report range, every visible local date is a no-shift rest date and this check is `Pass`, not `N/A`. Use `N/A` only when complete visible-range schedule evidence is unavailable.
- Drilldown: show each included employee's no-shift rest dates, threshold, and result. Include day-off rows only as optional supporting evidence; empty day-off rows must not cause `Review`.
- Detail: state that rest was inferred from no assigned shifts.

### Maximum interval between rest days

- Source: buffered assigned schedule rows. Day-off rows may be fetched as supporting evidence only.
- Fetch window: visible report range plus a 7-local-day buffer before `start` and after `end`.
- Count: for each included employee, treat buffered local dates with no assigned scheduled slot as no-shift rest dates, then compute the maximum number of consecutive local dates without a no-shift rest date.
- Threshold: no more than `7` consecutive local dates without a no-shift rest date.
- Status: `Review` if any included employee exceeds the threshold; otherwise `Pass`. If an included employee has zero assigned slots in the buffered range, every buffered local date is a no-shift rest date and this check is `Pass`, not `N/A`. Use `N/A` only when complete buffered schedule evidence is unavailable.
- Drilldown: show each included employee's buffered no-shift rest dates, maximum consecutive dates without rest, threshold, and result.

### 72-hour monthly overtime cap

- Source: full-calendar-month assigned schedule rows for included employees, plus compensation evidence for the same dates.
- Count: sum scheduled overtime hours per included employee per full calendar month touched by the visible report range.
- Prefer app-provided `scheduledHours.overtimeHours` when fetched data exposes it.
- If app-provided scheduled overtime is not exposed, estimate scheduled overtime using the same schedule OT semantics as StaffAny's webapp schedule tooltip.
- Source references for the intended logic:
  - `apps/gryphon/src/main/schedule/redux/selectors/hoursCosts.ts`
  - `apps/gryphon/src/common/domain/limitVariance.ts`
  - backend parity references: `countWorkHours.ts` and `breakdownHours.ts`
- Estimation algorithm when app-provided scheduled overtime is unavailable:
  1. Group full-month assigned Employee slots by `userId` and local schedule week.
  2. Sort each employee-week by local slot start time, then stable slot id.
  3. For each slot, calculate `workedDuration` as scheduled paid hours.
  4. Read `dailyLimit = compensation.dailyHours` and `weeklyLimit = compensation.weeklyHours` for the slot's local date. If both are missing for a slot, the monthly overtime check is `N/A` for that employee.
  5. Maintain `hoursWorkedInDay` and `hoursWorkedInWeek` counters using non-overtime worked hours only.
  6. Calculate daily overtime as hours above `dailyLimit` for the current local date when `dailyLimit` exists.
  7. Calculate weekly overtime as hours above `weeklyLimit` for the current local week when `weeklyLimit` exists.
  8. Count the larger of daily overtime and weekly overtime for that slot. If they are equal, use the daily overtime result.
  9. Add only the non-overtime part of the slot to `hoursWorkedInDay` and `hoursWorkedInWeek`.
  10. Sum slot overtime across all local weeks in each full calendar month touched by the visible report range.
- Threshold: `72` scheduled overtime hours per employee per calendar month.
- Status: `Review` if any included employee-month is greater than `72`; `Pass` if every included employee-month is at or below `72`; `N/A` only when schedule or compensation evidence is missing enough to estimate.
- Drilldown: include daily OT-after, weekly OT-after, total scheduled paid hours, estimated scheduled OT hours, diff to the 72-hour cap, source slots, and compensation evidence.

## Assumptions

- Do not assess workman/non-workman classification in this workshop stage.
- Do not use projected monthly salary, salary thresholds, nationality, contract type, actual duties, legal manager/executive status, or role names to determine Part 4 coverage.
- Do not treat unassigned slots as a compliance risk.
- For rest-day checks, use no-shift local dates from schedule rows. Day-off rows are supporting evidence only.
- This is a schedule-risk screen, not legal advice or a final payroll audit.

Render Part 4 scope in the employee drilldown table, not as a separate Part 4 table. For included employees, show Part 4 status `N/A`, workman classification `not assessed`, and a compact scope note. Excluded users may appear only in scope counts or compact excluded-scope notes.

## MOM Block

Write `mom-compliance-checklist.json` with:

- `id`: `momComplianceChecklist`
- `title`: `MOM Compliance Checklist`
- `checklist`
- `findings`
- `assumptions`
- `part4Classification`
- `employeeDrilldown`
- `sources`

Keep `findings` for compatibility, but leave it empty unless there is an exceptional execution issue. Do not use it for routine fetch counts or summary duplication.

Each checklist item should include lowercase machine status and report-facing label:

```json
{
  "status": "pass | review | na",
  "label": "Pass | Review | N/A",
  "title": "Maximum 12 hours worked in a day",
  "detail": "Plain-language schedule finding."
}
```

Each checklist item should also include a metric object where possible:

```json
{
  "metric": {
    "unit": "hours | slots | employees | dates",
    "value": 47,
    "threshold": 44,
    "source": "visible assigned Employee slots"
  }
}
```

Write `employeeDrilldown` as an array with one concise row per scheduled staff member considered by the stage:

```json
{
  "staffId": "user-id",
  "name": "Employee Name",
  "accessLevel": "Employee | Owner | Manager | unknown",
  "included": true,
  "excludedReason": null,
  "part4Status": "N/A",
  "workmanClassification": "not assessed",
  "scopeNote": "Included by StaffAny Employee access level only.",
  "metrics": {
    "maxDailyScheduledPaidHours": 8,
    "visibleScheduledPaidHours": 40,
    "breakRisk": "Pass",
    "noShiftRestDates": ["2026-07-10"],
    "maxNoRestIntervalDays": 5,
    "monthlyScheduledOtHours": 3.3
  },
  "checks": [
    {
      "checkTitle": "44-hour weekly / shift-work average threshold",
      "status": "pass | review | na",
      "label": "Pass | Review | N/A",
      "metric": {
        "unit": "hours",
        "value": 47,
        "threshold": 44,
        "source": "visible scheduled paid hours"
      },
      "detail": "Plain-language employee-level finding.",
      "evidenceRows": []
    }
  ]
}
```

Excluded Owner, Manager, and unknown-access rows may contain only scope evidence and excluded reason.

Keep `part4Classification` as a compatibility mirror of the Part 4 fields in `employeeDrilldown`; do not render it as a separate HTML section.

## Report Rendering

Write or replace only this stage's HTML file:

```text
reports/<start>-to-<end>/02-mom-compliance.html
```

The HTML must be a standalone static report rendered from `mom-compliance-checklist.json` and current context metadata. Do not display cost, SPLH, LQS, routine fetch-count findings, raw fetched data, or raw JSON evidence. If other block files exist in `artifacts/blocks/`, ignore them for this render. Do not create, overwrite, or delete `01-cost-and-splh.html` or `03-lqs-quota-analysis.html`.

Render an `Employee Drilldown` table below the summary checklist. It must combine Part 4 scope and per-employee check results in one simplified table with employee name, access level, Part 4 status, workman classification, max daily scheduled paid hours, visible scheduled paid hours, break risk, no-shift rest dates, max no-rest interval, monthly scheduled OT, and check statuses. Omit the `Findings` section when `findings` is empty. Do not render a separate `Part 4 Scope Assumptions` table.

## Validation

Before finishing:

1. Confirm `mom-compliance-checklist.json` exists.
2. Confirm `02-mom-compliance.html` contains `MOM Compliance Checklist`.
3. Confirm `02-mom-compliance.html` contains `Employee Drilldown`.
4. Confirm `02-mom-compliance.html` does not contain `Sales Per Labour Hour`, `Total estimated cost`, `LQS Quota Analysis`, `Findings`, `Fetched Data`, raw `<pre>` JSON drilldown blocks, or `Part 4 Scope Assumptions`.
5. Confirm Owner and Manager access-level users are absent from MOM calculations and Part 4 tables, except for excluded-scope notes.
6. Confirm `Part 4 coverage` is `N/A`.
7. Confirm the former standalone normal-daily-hours checklist item is absent.
8. Confirm every checklist item has source, count rule, threshold, status rule, and drilldown requirement.
9. Confirm the removed rest-day-pay and holiday-scheduling checks are absent from checklist items, counting rules, and drilldown requirements.

Use these fixture expectations when validating the implementation:

- Owner/Manager/unknown-access scheduled rows are excluded; Part 4 coverage remains `N/A`.
- Employee with `44.00` scheduled paid hours passes the 44-hour weekly threshold; `44.01` scheduled paid hours is `Review`.
- Employee with `72.00` monthly scheduled OT hours passes the 72-hour monthly cap; `72.01` monthly scheduled OT hours is `Review`.
- OT fixture mirrors the Gryphon schedule tooltip: daily OT after `7.2h`, weekly OT after `36h`, total scheduled hours `26h`, daily limit `28.8h` for 4 days, scheduled OT hours `3.3h`, and diff to OT after `3.3h`.
- Weekly rest day passes when an included employee has at least one no-shift local date in the visible range.
- Weekly rest day also passes when an included employee has zero assigned slots in the visible range, because every visible local date is a no-shift rest date.
- Maximum interval between rest days passes when no included employee exceeds 7 consecutive scheduled local dates in the buffered range.
- Maximum interval between rest days also passes when an included employee has zero assigned slots in the buffered range, because every buffered local date is a no-shift rest date.
- Drilldown shows concise metrics and per-check result for every included employee without raw fetched data.
