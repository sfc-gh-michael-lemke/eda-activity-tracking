---
name: "eda activity tracking report prompt"
created: "2026-06-25T12:48:15.893Z"
status: pending
---

# Plan: EDA Activity Tracking — Report Generation Prompt

## Context

### What I explored

**Data sources:**

- `SALES.SE_REPORTING.DIM_SE_ACTIVITY` — the production table (149k+ rows, SOURCE = Setsail/Vivun/Salesforce). This is the "established" system. The key field is `ACTIVITY_ID`, which is a Salesforce-assigned 18-character ID (e.g., `aPoVI000000DRXB0A4`).
- `TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES` — the staging table populated by `aggregate_activities.py` in the GitHub repo. This loads CSV files from a shared Google Drive. The join key is `SF_ACTIVITY_ID`, which maps to `ACTIVITY_ID` in DIM.

**The five team members in the data:** jlebonitte (Jim Lebonitte), psheehan (Patrick Sheehan), mhamilton (Michael Hamilton), palapaty (Phani Alapaty), smitchener (Steve Mitchener).

**How SF Activity IDs flow:** The `snowhouse_week.sql` query in the GitHub repo pulls Zoom, Gong, and Setsail records and joins them on date/title proximity. Gong and Zoom rows that match a Setsail activity inherit its `ACTIVITY_ID`. Those IDs end up in the CSV files as `SF Activity ID`, then get loaded into the staging table as `SF_ACTIVITY_ID`.

**Source types in the CSV files:**

- `Google Calendar` / `Calendar` — meetings pulled from GCal with no Setsail match yet → `SF_ACTIVITY_ID` is empty
- `Gong` / `Zoom` — calls recorded and matched to Setsail → `SF_ACTIVITY_ID` is populated

### Key findings from exploratory queries

| Metric                                                       | Value                   |
| ------------------------------------------------------------ | ----------------------- |
| Total rows in staging table                                  | 438                     |
| Date range                                                   | 2026-04-06 → 2026-06-18 |
| Rows WITH SF Activity ID                                     | 77 (17.6%)              |
| Rows WITHOUT SF Activity ID                                  | 361 (82.4%)             |
| SF Activity IDs that match DIM                               | **77 / 77 (100%)**      |
| SF Activity IDs that do NOT match DIM                        | 0                       |
| Matched rows where description differs (enriched AI summary) | 70 / 77 (90.9%)         |
| Matched rows where CSV adds a Use Case ID not in DIM         | 62 / 77 (80.5%)         |
| Matched rows where Use Case IDs conflict                     | 0                       |
| GCal rows (no SF ID) with title match in DIM on same date/SE | \~219 / 361 (60.7%)     |
| GCal rows that appear truly new (no DIM match)               | \~142 / 361 (39.3%)     |

---

## The Report-Generation Prompt

Below is the complete prompt. When run in Cortex Code (agent mode), it will query both tables, run the comparisons, and produce a structured report.

---

````
You are a Snowflake data analyst. Your task is to generate a detailed data quality and overlap 
report comparing the EDA team's new activity tracking method against the established 
production system. Use the Snowflake SQL execution tool for all data retrieval. 
Produce a structured markdown report at the end.

---
## CONTEXT

The team tracks SE (Sales Engineer) customer meetings using two parallel systems:

1. **Production system** — `SALES.SE_REPORTING.DIM_SE_ACTIVITY`
   - Source: Setsail (auto-captures meetings from Google Calendar where there is at least one 
     external attendee and fewer than 25 total participants), plus Vivun and Salesforce.
   - Primary key: `ACTIVITY_ID` — a Salesforce-assigned 18-character ID (e.g., `aPoVI000000DRXB0A4`)
   - Key fields: ACTIVITY_ID, ACTIVITY_SE_NAME, ACTIVITY_DATE, ACTIVITY_DESCRIPTION, 
     ACCOUNT_NAME, ACCOUNT_ID, OPP_ID, USE_CASE_ID, ACTIVITY_TYPE, SOURCE

2. **New EDA method** — `TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES`
   - Source: CSV files exported by team members from a tool that combines Google Calendar, 
     Gong call recordings, and Zoom data. The files are loaded into this staging table via 
     `aggregate_activities.py` (GitHub: sfc-gh-psheehan/time-tracking).
   - Join key: `SF_ACTIVITY_ID` → maps to `ACTIVITY_ID` in the production system
   - Key fields: SF_ACTIVITY_ID, MEETING_SE_NAME, MEETING_DATE, MEETING_TITLE, CUSTOMER, 
     SOURCE, SF_ACCOUNT_ID, OPPORTUNITY_ID, USE_CASE_ID, SUMMARY, NEXT_STEPS, ATTENDEES, 
     SOURCE_FILE

**The SF Activity ID is the critical link.** When a meeting is matched to a Setsail record in 
the CSV pipeline (via Gong or Zoom), the Salesforce Activity ID is written into `SF_ACTIVITY_ID`. 
When a meeting is purely from Google Calendar with no Setsail match, `SF_ACTIVITY_ID` is NULL.

The five team members producing CSV data:
- Jim Lebonitte (MEETING_SE_NAME = 'Jim Lebonitte')
- Patrick Sheehan (MEETING_SE_NAME = 'Patrick Sheehan')
- Michael Hamilton (MEETING_SE_NAME = 'Michael Hamilton')
- Phani Alapaty (MEETING_SE_NAME = 'Phani Alapaty')
- Steve Mitchener (MEETING_SE_NAME = 'Steve Mitchener')

---
## REPORT SECTIONS TO GENERATE

Run the SQL queries below (or equivalent) to gather data for each section.

---
### SECTION 0 — EXECUTIVE SUMMARY

Run these queries:

```sql
-- Total volume overview
SELECT 
    COUNT(*) as total_csv_rows,
    MIN(MEETING_DATE) as period_start,
    MAX(MEETING_DATE) as period_end,
    COUNT(DISTINCT MEETING_SE_NAME) as team_members,
    SUM(CASE WHEN SF_ACTIVITY_ID IS NOT NULL AND SF_ACTIVITY_ID != '' THEN 1 ELSE 0 END) as rows_with_sf_id,
    SUM(CASE WHEN SF_ACTIVITY_ID IS NULL OR SF_ACTIVITY_ID = '' THEN 1 ELSE 0 END) as rows_without_sf_id
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES;
````

```sql
-- Volume per team member
SELECT 
    MEETING_SE_NAME,
    COUNT(*) as total_csv_records,
    COUNT(DISTINCT MEETING_DATE) as distinct_meeting_days,
    SUM(CASE WHEN SF_ACTIVITY_ID IS NOT NULL AND SF_ACTIVITY_ID != '' THEN 1 ELSE 0 END) as has_sf_activity_id,
    SUM(CASE WHEN SF_ACTIVITY_ID IS NULL OR SF_ACTIVITY_ID = '' THEN 1 ELSE 0 END) as no_sf_activity_id
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES
GROUP BY MEETING_SE_NAME
ORDER BY total_csv_records DESC;
```

---

### SECTION 1 — BULK LOAD OVERLAP ANALYSIS

Question: How often is the team loading records that already exist in DIM\_SE\_ACTIVITY?

**Approach A — Records with SF Activity ID (direct join possible):**

```sql
WITH csv_with_id AS (
    SELECT SF_ACTIVITY_ID, MEETING_SE_NAME, MEETING_DATE, MEETING_TITLE, CUSTOMER, SUMMARY,
           OPPORTUNITY_ID, USE_CASE_ID, SF_ACCOUNT_ID, NEXT_STEPS, SOURCE_FILE
    FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES
    WHERE SF_ACTIVITY_ID IS NOT NULL AND SF_ACTIVITY_ID != ''
)
SELECT
    COUNT(*) as total_csv_rows_with_sf_id,
    SUM(CASE WHEN d.ACTIVITY_ID IS NOT NULL THEN 1 ELSE 0 END) as already_in_dim,
    SUM(CASE WHEN d.ACTIVITY_ID IS NULL THEN 1 ELSE 0 END) as not_in_dim,
    ROUND(SUM(CASE WHEN d.ACTIVITY_ID IS NOT NULL THEN 1.0 ELSE 0 END) / COUNT(*) * 100, 1) as pct_already_in_dim
FROM csv_with_id c
LEFT JOIN SALES.SE_REPORTING.DIM_SE_ACTIVITY d ON c.SF_ACTIVITY_ID = d.ACTIVITY_ID;
```

**Approach B — Records WITHOUT SF Activity ID (potential silent duplicates):** These are Google Calendar-sourced rows. Check whether Setsail already captured the same meeting based on SE name, date, and meeting title substring match.

```sql
WITH csv_no_id AS (
    SELECT MEETING_SE_NAME, MEETING_DATE, MEETING_TITLE, CUSTOMER, SOURCE
    FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES
    WHERE (SF_ACTIVITY_ID IS NULL OR SF_ACTIVITY_ID = '')
),
dim_setsail AS (
    SELECT ACTIVITY_SE_NAME, ACTIVITY_DATE, ACTIVITY_DESCRIPTION, ACTIVITY_ID
    FROM SALES.SE_REPORTING.DIM_SE_ACTIVITY
    WHERE ACTIVITY_SE_NAME IN 
        ('Jim Lebonitte','Michael Hamilton','Patrick Sheehan','Phani Alapaty','Steve Mitchener')
    AND SOURCE = 'Setsail'
    AND ACTIVITY_TYPE = 'Meeting'
    AND ACTIVITY_DATE BETWEEN '2026-04-06' AND '2026-06-21'
)
SELECT
    (SELECT COUNT(*) FROM csv_no_id) as total_gcal_no_id_rows,
    COUNT(DISTINCT c.MEETING_SE_NAME || '|' || c.MEETING_DATE || '|' || c.MEETING_TITLE) as gcal_rows_with_title_match_in_dim,
    (SELECT COUNT(*) FROM csv_no_id) - 
    COUNT(DISTINCT c.MEETING_SE_NAME || '|' || c.MEETING_DATE || '|' || c.MEETING_TITLE) as gcal_rows_potentially_new
FROM csv_no_id c
JOIN dim_setsail d 
    ON d.ACTIVITY_SE_NAME = c.MEETING_SE_NAME
    AND d.ACTIVITY_DATE = c.MEETING_DATE
    AND TRIM(LOWER(d.ACTIVITY_DESCRIPTION)) LIKE '%' || TRIM(LOWER(c.MEETING_TITLE)) || '%';
```

Report the count and percentage of all three categories:

1. CSV rows with SF Activity ID that are already in DIM (confirmed overlap)
2. CSV rows without SF Activity ID that have a probable DIM match (potential silent overlap)
3. CSV rows without SF Activity ID with no DIM match (likely truly new meetings)

---

### SECTION 2 — MODIFICATION ANALYSIS

Question: For records that already exist in DIM, what is the team changing?

This applies only to the 77 rows with SF Activity IDs (the ones confirmed to exist in DIM).

```sql
SELECT
    c.MEETING_SE_NAME,
    c.SF_ACTIVITY_ID,
    c.MEETING_DATE,
    c.MEETING_TITLE,

    -- Description enrichment
    d.ACTIVITY_DESCRIPTION as dim_existing_description,
    c.SUMMARY as csv_ai_summary,
    CASE 
        WHEN c.SUMMARY IS NOT NULL AND c.SUMMARY != '' 
             AND d.ACTIVITY_DESCRIPTION IS NOT NULL AND d.ACTIVITY_DESCRIPTION != '' 
        THEN 'Enriched (both have description, CSV adds AI summary)'
        WHEN c.SUMMARY IS NOT NULL AND c.SUMMARY != '' 
             AND (d.ACTIVITY_DESCRIPTION IS NULL OR d.ACTIVITY_DESCRIPTION = '')
        THEN 'Net new description (DIM had none)'
        ELSE 'No enrichment'
    END as description_change_type,

    -- Use Case ID enrichment
    d.USE_CASE_ID as dim_use_case_id,
    c.USE_CASE_ID as csv_use_case_id,
    CASE 
        WHEN c.USE_CASE_ID IS NOT NULL AND c.USE_CASE_ID != '' 
             AND (d.USE_CASE_ID IS NULL OR d.USE_CASE_ID = '')
        THEN 'CSV adds UC ID (DIM had none)'
        WHEN c.USE_CASE_ID IS NOT NULL AND d.USE_CASE_ID IS NOT NULL 
             AND c.USE_CASE_ID = d.USE_CASE_ID
        THEN 'UC ID matches'
        WHEN c.USE_CASE_ID IS NOT NULL AND d.USE_CASE_ID IS NOT NULL 
             AND c.USE_CASE_ID != d.USE_CASE_ID
        THEN 'UC ID CONFLICT — values differ'
        ELSE 'No UC ID in CSV'
    END as use_case_change_type,

    -- Opportunity ID enrichment
    d.OPP_ID as dim_opp_id,
    c.OPPORTUNITY_ID as csv_opp_id,
    CASE 
        WHEN c.OPPORTUNITY_ID IS NOT NULL AND c.OPPORTUNITY_ID != '' 
             AND (d.OPP_ID IS NULL OR d.OPP_ID = '')
        THEN 'CSV adds Opp ID (DIM had none)'
        WHEN c.OPPORTUNITY_ID IS NOT NULL AND d.OPP_ID IS NOT NULL 
             AND c.OPPORTUNITY_ID = d.OPP_ID
        THEN 'Opp ID matches'
        WHEN c.OPPORTUNITY_ID IS NOT NULL AND d.OPP_ID IS NOT NULL 
             AND c.OPPORTUNITY_ID != d.OPP_ID
        THEN 'Opp ID CONFLICT — values differ'
        ELSE 'No Opp ID in CSV'
    END as opp_change_type

FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES c
JOIN SALES.SE_REPORTING.DIM_SE_ACTIVITY d ON c.SF_ACTIVITY_ID = d.ACTIVITY_ID
WHERE c.SF_ACTIVITY_ID IS NOT NULL AND c.SF_ACTIVITY_ID != ''
ORDER BY c.MEETING_SE_NAME, c.MEETING_DATE;
```

Summarize the results with counts per change type:

- How many add net-new AI summaries
- How many have summaries that differ from the existing DIM description (AI-generated vs. email subject line format)
- How many add a Use Case ID
- How many Use Case IDs conflict
- How many add an Opportunity ID
- How many Opportunity IDs conflict

---

### SECTION 3 — SF ACTIVITY ID HEALTH CHECK

Question: When they are referencing an existing record, do they provide the correct SF Activity ID? Question: When they are logging a new record, is the SF Activity ID appropriately blank?

**Part A — ID validity for existing record references:**

```sql
SELECT
    COUNT(*) as total_rows_with_sf_id,
    SUM(CASE WHEN d.ACTIVITY_ID IS NOT NULL THEN 1 ELSE 0 END) as valid_sf_ids,
    SUM(CASE WHEN d.ACTIVITY_ID IS NULL THEN 1 ELSE 0 END) as invalid_sf_ids,
    ROUND(SUM(CASE WHEN d.ACTIVITY_ID IS NOT NULL THEN 1.0 ELSE 0 END) / COUNT(*) * 100, 1) as pct_valid
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES c
LEFT JOIN SALES.SE_REPORTING.DIM_SE_ACTIVITY d ON c.SF_ACTIVITY_ID = d.ACTIVITY_ID
WHERE c.SF_ACTIVITY_ID IS NOT NULL AND c.SF_ACTIVITY_ID != '';
```

If any IDs are invalid (not found in DIM), list them:

```sql
SELECT c.MEETING_SE_NAME, c.MEETING_DATE, c.MEETING_TITLE, c.SF_ACTIVITY_ID, c.SOURCE_FILE
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES c
LEFT JOIN SALES.SE_REPORTING.DIM_SE_ACTIVITY d ON c.SF_ACTIVITY_ID = d.ACTIVITY_ID
WHERE c.SF_ACTIVITY_ID IS NOT NULL AND c.SF_ACTIVITY_ID != ''
AND d.ACTIVITY_ID IS NULL;
```

**Part B — Blank SF Activity ID for GCal-only records (expected behavior):**

```sql
SELECT 
    SOURCE,
    COUNT(*) as row_count,
    SUM(CASE WHEN SF_ACTIVITY_ID IS NULL OR SF_ACTIVITY_ID = '' THEN 1 ELSE 0 END) as blank_sf_id,
    SUM(CASE WHEN SF_ACTIVITY_ID IS NOT NULL AND SF_ACTIVITY_ID != '' THEN 1 ELSE 0 END) as has_sf_id
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES
GROUP BY SOURCE
ORDER BY row_count DESC;
```

Evaluate: GCal-sourced rows should mostly have blank SF Activity IDs (they haven't been matched to Setsail yet). Gong/Zoom-sourced rows should mostly have populated SF Activity IDs.

---

### SECTION 4 — SAMPLE EVIDENCE

Provide 5 representative examples for each pattern:

1. **Exact bulk duplicate** — Row where CSV matches DIM on all key fields (customer, date, title)
2. **Enrichment (good)** — Row where CSV adds AI summary + Use Case ID to an existing DIM record
3. **GCal row with DIM match** — A calendar-only row where Setsail captured the same meeting
4. **Truly new record** — A GCal row that has no match in DIM (missed by Setsail)
5. **Any conflict** — Row where CSV and DIM disagree on a field value

---

### REPORT OUTPUT FORMAT

Write the report in this structure:

```
# EDA Activity Tracking — Overlap & Data Quality Report
**Period:** [start date] — [end date]
**Team members analyzed:** [list]
**Report generated:** [current date]
**Data sources:**
  - Production: SALES.SE_REPORTING.DIM_SE_ACTIVITY
  - New method: TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES

---
## Executive Summary
[2–3 sentence summary of overall findings]

| Category | Count | % of Total |
|----------|-------|------------|
| Total CSV rows loaded | N | — |
| Rows with SF Activity ID (confirmed existing records) | N | N% |
| Rows without SF Activity ID (GCal-only, unmatched) | N | N% |
| — Of which: probable match in DIM by date+title | N | N% |
| — Of which: appear truly new (no DIM match) | N | N% |

---
## Section 1: Bulk Load Overlap
[Findings from Section 1 queries]
[Commentary: is this good or bad? What should change?]

---
## Section 2: What Is Being Changed
[Findings from Section 2 queries]
[Table showing change types and counts]
[Assessment: are changes additive (enrichment) or contradictory (conflict)?]

---
## Section 3: SF Activity ID Health
[Findings from Section 3 queries]
[Part A: Are IDs valid when provided? List any invalid IDs]
[Part B: Are blank IDs appropriate? What % of GCal rows are blank vs Gong/Zoom rows]

---
## Section 4: Sample Evidence
[5 examples per pattern, formatted as a readable table]

---
## Section 5: Recommendations
Based on the data, provide recommendations on:
1. Whether the team should continue the new method, modify it, or align with the production system
2. Specific data quality issues to fix (e.g., missing Use Case IDs in DIM, blank SF Activity IDs 
   that could be resolved by running the Setsail match)
3. Any records that need manual attention
```

````

---

## Critical Files

- `TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES` — Staging table with all loaded CSV data (438 rows, Apr–Jun 2026)
- `SALES.SE_REPORTING.DIM_SE_ACTIVITY` — Production activity table (join target via ACTIVITY_ID)
- `/Users/mlemke/Documents/projects/Activty_Tracking_EDA_Method/From-EDA/eda-activity-tracking/` — Local CSV files (5 team members, Apr–Jun 2026)
- `https://github.com/sfc-gh-psheehan/time-tracking/blob/main/aggregate_activities.py` — Python loader (maps CSV → staging table, validates SF IDs, truncates+reloads each run)
- `https://github.com/sfc-gh-psheehan/time-tracking/blob/main/snowhouse_week.sql` — SQL that assigns SF Activity IDs by joining Zoom/Gong to Setsail

---

## Verification

The prompt is self-contained: every SQL query in it references tables that exist and have the schema described. The queries have been validated interactively above. Before using the prompt, confirm the staging table is current by running:
```sql
SELECT COUNT(*), MAX(LOADED_AT) FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES;
````

If `LOADED_AT` is stale, re-run `bash run.sh aggregate` from the GitHub repo to reload from Google Drive.
