# EDA Activity Tracking

> SE weekly activity tracking methodology and Salesforce import tooling.

Documentation and sample files for the EDA (Engineering and Design Activity) method of tracking Sales Engineer weekly activities at Snowflake.

## What It Does

The EDA method captures SE activities (customer meetings, demos, POCs, workshops) in structured weekly CSV files organized by SE, then converts them to Salesforce-importable format via the companion [`eda-sfdc-import`](https://github.com/sfc-gh-michael-lemke/eda-sfdc-import) CoCo skill.

## Repository Contents

| File/Directory | Purpose |
|----------------|---------|
| `sfdc_import_sample.csv` | Example import CSV structure (new activities) |
| `sfdc_update_sample.csv` | Example update CSV structure (existing activities) |
| `eda_activity_overlap_report.md` | Analysis of activity overlap patterns |
| `eda_activity_overlap_report_prompt.md` | Prompt template for overlap analysis |

## Business Value

| Benefit | Description |
|---------|-------------|
| Consistency | Standardized activity tracking format across SE team |
| Automation | Works with `eda-sfdc-import` skill for zero-manual-entry SFDC updates |
| Auditability | Full SE activity history in Salesforce for performance reviews |
| Coaching | Activity data feeds SE HP Analysis for coaching insights |

## Workflow

1. SEs log activities weekly in EDA CSV format (organized by SE folder)
2. Run the [`eda-sfdc-import`](https://github.com/sfc-gh-michael-lemke/eda-sfdc-import) CoCo skill to convert to SFDC format
3. Import generated CSVs to Salesforce via the bulk importer
4. Activity data flows into Snowflake via Fivetran for reporting

## Related Projects

- [`eda-sfdc-import`](https://github.com/sfc-gh-michael-lemke/eda-sfdc-import) — CoCo skill that automates the EDA→SFDC conversion
- [`se-hp-analysis`](https://github.com/sfc-gh-michael-lemke/se-hp-analysis) — HP analysis that consumes this activity data

## Value Realization

Structured, automated activity tracking ensures 100% SFDC data completeness for SE activities — enabling accurate pipeline influence analysis, fair performance reviews, and data-driven SE coaching without manual Salesforce entry.
