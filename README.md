# EDA Activity Tracking

> SE weekly activity tracking methodology and Salesforce import tooling.

Documentation and sample files for the EDA (Engineering and Design Activity) method of tracking Sales Engineer weekly activities at Snowflake.

## What It Does

The EDA method captures SE activities (customer meetings, demos, POCs, workshops) in structured weekly CSV files organized by SE folder, then converts them to Salesforce-importable format via the companion [`eda-sfdc-import`](https://github.com/sfc-gh-michael-lemke/eda-sfdc-import) CoCo skill.

## Repository Contents

| File | Purpose |
|------|---------|
| `sfdc_import_sample.csv` | Example structure for new SFDC activity imports |
| `sfdc_update_sample.csv` | Example structure for updating existing SFDC activities |
| `eda_activity_overlap_report.md` | Analysis of activity overlap patterns across SEs |
| `eda_activity_overlap_report_prompt.md` | Prompt template for generating overlap analysis |

## Business Value

| Benefit | Description |
|---------|-------------|
| Consistency | Standardized weekly activity format across the SE team |
| Automation | Works with `eda-sfdc-import` skill for zero-manual-entry SFDC updates |
| Auditability | Full SE activity history in Salesforce for performance reviews |
| Coaching data | Activity patterns feed SE HP Analysis for data-driven coaching |
| Compliance | Ensures complete SFDC data for pipeline influence attribution |

## Workflow

1. SEs log activities weekly in EDA CSV format (organized by SE name folder)
2. Run the [`eda-sfdc-import`](https://github.com/sfc-gh-michael-lemke/eda-sfdc-import) CoCo skill to convert to SFDC format
3. Import generated CSVs to Salesforce via bulk importer
4. Activity data flows into Snowflake via Fivetran for reporting and HP analysis

## CSV Column Schema

| EDA Column | SFDC Import Column |
|-----------|-------------------|
| SF Activity ID | SF Activity ID |
| Date | Date of Meeting |
| Meeting Title | Subject |
| SF Account ID | Account |
| Opportunity ID | Opportunity |
| Use Case ID | Use Case |
| Meeting SE Name | Owner (resolved to SFDC User ID) |
| Summary + Next Steps + Attendees | Description (formatted) |

## Related Projects

- [`eda-sfdc-import`](https://github.com/sfc-gh-michael-lemke/eda-sfdc-import) — CoCo skill that automates the EDA→SFDC conversion
- [`se-hp-analysis`](https://github.com/sfc-gh-michael-lemke/se-hp-analysis) — HP analysis driven by this activity data

## Value Realization

Structured, automated activity tracking ensures 100% SFDC data completeness for SE activities — enabling accurate pipeline influence analysis, fair performance reviews, and data-driven SE coaching without manual Salesforce entry. This methodology eliminates the weekly data entry burden and gives leadership a reliable, consistent view of SE contribution across accounts.
