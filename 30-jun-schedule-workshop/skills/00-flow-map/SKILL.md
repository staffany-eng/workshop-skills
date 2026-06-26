---
name: workshop-schedule-analysis-flow-map
description: Map the step-by-step Schedule Analysis workshop stages. Each analysis stage fetches the Workspace API data it needs and updates the report incrementally.
---

# Workshop Schedule Analysis Flow Map

## Purpose

Show the workshop sequence. This is not an orchestrator; run each stage skill only when that part of the workshop starts.

There is no standalone fetch stage. Each analysis stage must be able to discover the Workspace API key, fetch the data it needs, format its block artifact, and refresh the report.

## Stage Order

1. `workshop-cost-and-splh`
   - Fetch Workspace API schedule, staff, compensation, and sales data for the visible range.
   - Compute schedule cost and Sales Per Labour Hour.
   - Write cost and SPLH block JSON.
   - Render the report with completed cost/SPLH sections.

2. `workshop-mom-compliance`
   - Fetch Workspace API schedule, staff, compensation, and public holiday or special-date data needed for MOM checks.
   - Add the MOM Compliance Checklist block.
   - Refresh the report while preserving earlier sections.

3. `workshop-lqs-quota-analysis`
   - Fetch Workspace API staff, statutory, compensation, schedule, and timesheet data needed for LQS quota analysis.
   - Add LQS Quota Analysis.
   - Refresh the final report with the LQS tooltip and quota detail.

4. `workshop-render-schedule-analysis-html`
   - Shared renderer behavior used by each analysis stage.
   - Render only sections whose block files exist.

## Artifact Rule

Every stage writes into the same report folder:

```text
reports/<start>-to-<end>/
  artifacts/
    schedule-context.json
    blocks/
      schedule-cost-estimate.json
      sales-per-labour-hour.json
      mom-compliance-checklist.json
      lqs-quota-analysis.json
  schedule-cost-report.html
```

`schedule-context.json` is a supporting cache. It may be created or refreshed by any stage, but no stage should require a previous stage to have fetched it already.

Do not write generated workshop artifacts into the repository root.
