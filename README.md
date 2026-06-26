# StaffAny Workshop Skills

This repository is used to publicly share Codex skills prepared for StaffAny workshops.

Each workshop folder may contain one or more skills that can be copied into a local Codex skills directory and used as reference implementations or starting points for similar workflows.

## Current Workshops

- `30-jun-schedule-workshop/`

## 30 Jun Schedule Workshop

The workshop skills are intentionally isolated. Each stage discovers the Workspace API key, fetches the data it needs, writes its own block artifacts, and writes its own standalone HTML report.

Do not make one stage depend on a previous stage. Do not render block files from another stage just because they already exist in the report folder. Do not overwrite another stage's HTML file.

Each stage targets StaffAny users with access level `Employee` only. Exclude `Owner` and `Manager` access-level users from report calculations and report tables because those users are workshop runners or participants.

Run the stages independently:

1. `workshop-cost-and-splh`
   - Fetch schedule, staff, compensation, and sales data for the visible range.
   - Compute schedule cost and Sales Per Labour Hour.
   - Render only the cost and SPLH report sections to `reports/<start>-to-<end>/01-cost-and-splh.html`.

2. `workshop-mom-compliance`
   - Fetch schedule, staff, compensation, and public holiday or special-date data needed for MOM checks.
   - Write the MOM Compliance Checklist block.
   - Render only the MOM report section to `reports/<start>-to-<end>/02-mom-compliance.html`.

3. `workshop-lqs-quota-analysis`
   - Fetch staff, statutory, compensation, schedule, and timesheet data needed for LQS quota analysis.
   - Write the LQS Quota Analysis block.
   - Render only the LQS report section to `reports/<start>-to-<end>/03-lqs-quota-analysis.html`.

At the end of the workshop, the report folder should contain all three HTML files.

## Intended Use

The skills in this repository are intended for public dissemination and workshop enablement. They are shared to make workshop material easier to inspect, reuse, and adapt.

These skills are primarily designed around StaffAny Workspace API use cases, including schedule analysis, labour cost review, compliance checks, sales per labour hour analysis, and related reporting workflows. They may not fit every business, industry, jurisdiction, or StaffAny account configuration without modification.

## Disclaimers

- These skills are provided as-is.
- These skills may not be updated after publication.
- These skills may become outdated as StaffAny products, APIs, policies, or workshop materials change.
- These skills may be changed, moved, deprecated, or removed at any time at StaffAny's discretion.
- These skills are not a commitment to provide support, maintenance, compatibility, or continued availability.
- Users are responsible for reviewing outputs before relying on them, especially for compliance, payroll, labour cost, or operational decisions.

## Repository Layout

```text
<workshop-folder>/
  skills/
    <skill-name>/
      SKILL.md
      agents/
      scripts/
      references/
      assets/
```

Only `SKILL.md` is required for a skill. Supporting folders are included only where they are useful to the skill workflow.
