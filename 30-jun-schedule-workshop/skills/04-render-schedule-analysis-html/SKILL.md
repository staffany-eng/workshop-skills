---
name: workshop-render-schedule-analysis-html
description: Shared renderer behavior for the step-by-step Schedule Analysis workshop. Render HTML from whichever checker blocks exist so the report grows after each workshop part.
---

# Workshop Render Schedule Analysis HTML

## Purpose

Render the Schedule Analysis HTML from available artifacts. The workshop runs in parts, so the renderer must support partial reports.

Each analysis stage can call this behavior after it writes its own block file.

## Inputs

Read when present:

```text
reports/<start>-to-<end>/artifacts/schedule-context.json
reports/<start>-to-<end>/report-params.json
```

Read any block files that exist:

```text
reports/<start>-to-<end>/artifacts/blocks/schedule-cost-estimate.json
reports/<start>-to-<end>/artifacts/blocks/sales-per-labour-hour.json
reports/<start>-to-<end>/artifacts/blocks/mom-compliance-checklist.json
reports/<start>-to-<end>/artifacts/blocks/lqs-quota-analysis.json
```

Use `schedule-context.json` for organisation, timezone, and range metadata when present. If it is missing, use `report-params.json` and block metadata where available.

## Output

Write:

```text
reports/<start>-to-<end>/schedule-cost-report.html
```

## Rendering Rules

- Always render the `Schedule Analysis` title, organisation when known, and visible report range.
- Render cost sections only when `schedule-cost-estimate.json` exists.
- Render Sales Per Labour Hour only when `sales-per-labour-hour.json` exists.
- Render `Daily Cost Curve` from the cost block inside `Sales Per Labour Hour`, not as its own top-level section.
- Render MOM Compliance Checklist and Part 4 assumptions only when `mom-compliance-checklist.json` exists.
- Render LQS Quota Analysis only when `lqs-quota-analysis.json` exists.
- Do not show empty placeholders for stages that have not run.
- Keep styling quiet and operational.
- Include the LQS proxies/count-logic/assumptions tooltip whenever the LQS section is rendered.

## Validation

After rendering:

- Scan for stale labels such as `LQS Compliance`, `low-hanging`, `1.0 local count staff`, and `0.5 local count staff`.
- Confirm missing stage blocks do not produce empty sections.
- Open the generated HTML when visual output changed.
