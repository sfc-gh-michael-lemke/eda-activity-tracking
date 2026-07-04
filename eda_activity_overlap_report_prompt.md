# EDA Activity Tracking — Overlap & Data Quality Report Prompt

**Purpose:** This prompt instructs an AI agent to generate a detailed report comparing the EDA
team's new activity tracking method (CSV files → Snowflake staging table) against the
established production activity system. Paste this entire prompt into Cortex Code (agent mode)
to generate the report.

---

## PROMPT — PASTE BELOW THIS LINE

You are a Snowflake data analyst. Generate a detailed data quality and overlap report comparing
the EDA team's new activity tracking method against the established production system.
Use the Snowflake SQL execution tool for all data retrieval. Produce a structured markdown
report at the end.

---

### CONTEXT

The team tracks SE (Sales Engineer) customer meetings using two parallel systems:

**1. Production system — `SALES.SE_REPORTING.DIM_SE_ACTIVITY`**
- Source: Setsail auto-captures meetings from Google Calendar where there is at least one
  external email domain attendee and fewer than 25 total participants. Also includes Vivun
  and Salesforce-logged activities.
- Primary key: `ACTIVITY_ID` — a Salesforce-assigned 18-character ID (e.g., `aPoVI000000DRXB0A4`)
- Relevant fields: ACTIVITY_ID, ACTIVITY_SE_NAME, ACTIVITY_DATE, ACTIVITY_DESCRIPTION,
  ACCOUNT_NAME, ACCOUNT_ID, OPP_ID, USE_CASE_ID, ACTIVITY_TYPE, SOURCE

**2. New EDA method — `TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES`**
- Source: CSV files exported by team members from a tool that combines Google Calendar,
  Gong call recordings, and Zoom meeting data. Loaded into Snowflake via `aggregate_activities.py`
  (GitHub: sfc-gh-psheehan/time-tracking). The table is TRUNCATED and fully reloaded on each run.
- Join key to production: `SF_ACTIVITY_ID` maps to `ACTIVITY_ID` in the production system
- Relevant fields: SF_ACTIVITY_ID, MEETING_SE_NAME, MEETING_DATE, MEETING_TITLE, CUSTOMER,
  SOURCE, SF_ACCOUNT_ID, OPPORTUNITY_ID, USE_CASE_ID, SUMMARY, NEXT_STEPS, ATTENDEES, SOURCE_FILE

**How SF Activity IDs get populated in the CSV files:**
When a meeting is matched to a Setsail-logged activity (via Gong recording or Zoom record),
the Salesforce Activity ID is written into `SF Activity ID` in the CSV. This only happens for
Gong- and Zoom-sourced rows. Google Calendar-only rows have no match and therefore have a blank
`SF Activity ID`. Blank SF Activity IDs are expected and acceptable for calendar-only records.

**The five team members producing CSV data:**
- Jim Lebonitte   (MEETING_SE_NAME = 'Jim Lebonitte')
- Patrick Sheehan (MEETING_SE_NAME = 'Patrick Sheehan')
- Michael Hamilton (MEETING_SE_NAME = 'Michael Hamilton')
- Phani Alapaty   (MEETING_SE_NAME = 'Phani Alapaty')
- Steve Mitchener (MEETING_SE_NAME = 'Steve Mitchener')

---

### REPORT SECTIONS — RUN THESE QUERIES IN ORDER

---

#### SECTION 0 — EXECUTIVE SUMMARY

```sql
-- Overall volume
SELECT
    COUNT(*)                                                                        AS total_csv_rows,
    MIN(MEETING_DATE)                                                               AS period_start,
    MAX(MEETING_DATE)                                                               AS period_end,
    COUNT(DISTINCT MEETING_SE_NAME)                                                 AS team_members,
    SUM(CASE WHEN SF_ACTIVITY_ID IS NOT NULL AND SF_ACTIVITY_ID != '' THEN 1 ELSE 0 END) AS rows_with_sf_id,
    SUM(CASE WHEN SF_ACTIVITY_ID IS NULL     OR SF_ACTIVITY_ID  = '' THEN 1 ELSE 0 END) AS rows_without_sf_id
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES;
```

```sql
-- Volume per team member
SELECT
    MEETING_SE_NAME,
    COUNT(*)                                                                             AS total_csv_records,
    COUNT(DISTINCT MEETING_DATE)                                                          AS distinct_meeting_days,
    SUM(CASE WHEN SF_ACTIVITY_ID IS NOT NULL AND SF_ACTIVITY_ID != '' THEN 1 ELSE 0 END) AS has_sf_activity_id,
    SUM(CASE WHEN SF_ACTIVITY_ID IS NULL     OR SF_ACTIVITY_ID  = '' THEN 1 ELSE 0 END) AS no_sf_activity_id
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES
GROUP BY MEETING_SE_NAME
ORDER BY total_csv_records DESC;
```

---

#### SECTION 1 — BULK LOAD OVERLAP ANALYSIS

**Question: How often is the team loading records that already exist in DIM_SE_ACTIVITY?**

This has two sub-cases:
- **Case A:** Rows with SF Activity ID — direct, definitive join to DIM.
- **Case B:** Rows without SF Activity ID (Google Calendar-only) — soft match by SE name,
  date, and meeting title substring.

**Case A — Direct ID match:**
```sql
SELECT
    COUNT(*)                                                                              AS total_csv_rows_with_sf_id,
    SUM(CASE WHEN d.ACTIVITY_ID IS NOT NULL THEN 1 ELSE 0 END)                           AS already_in_dim,
    SUM(CASE WHEN d.ACTIVITY_ID IS NULL     THEN 1 ELSE 0 END)                           AS not_in_dim,
    ROUND(SUM(CASE WHEN d.ACTIVITY_ID IS NOT NULL THEN 1.0 ELSE 0 END) / COUNT(*) * 100, 1) AS pct_already_in_dim
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES c
LEFT JOIN SALES.SE_REPORTING.DIM_SE_ACTIVITY d ON c.SF_ACTIVITY_ID = d.ACTIVITY_ID
WHERE c.SF_ACTIVITY_ID IS NOT NULL AND c.SF_ACTIVITY_ID != '';
```

**Case B — GCal rows: soft title match against DIM:**
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
      AND SOURCE        = 'Setsail'
      AND ACTIVITY_TYPE = 'Meeting'
      AND ACTIVITY_DATE BETWEEN '2026-04-06' AND '2026-06-21'
)
SELECT
    (SELECT COUNT(*) FROM csv_no_id)                                                          AS total_gcal_no_id_rows,
    COUNT(DISTINCT c.MEETING_SE_NAME || '|' || c.MEETING_DATE || '|' || c.MEETING_TITLE)     AS gcal_rows_with_title_match_in_dim,
    (SELECT COUNT(*) FROM csv_no_id) -
    COUNT(DISTINCT c.MEETING_SE_NAME || '|' || c.MEETING_DATE || '|' || c.MEETING_TITLE)     AS gcal_rows_potentially_new
FROM csv_no_id c
JOIN dim_setsail d
    ON  d.ACTIVITY_SE_NAME = c.MEETING_SE_NAME
    AND d.ACTIVITY_DATE    = c.MEETING_DATE
    AND TRIM(LOWER(d.ACTIVITY_DESCRIPTION)) LIKE '%' || TRIM(LOWER(c.MEETING_TITLE)) || '%';
```

After running both queries, report:
1. Rows with SF Activity ID already in DIM (confirmed overlap — count and %)
2. GCal rows likely already in DIM by title match (probable overlap — count and %)
3. GCal rows with no DIM match (likely truly new meetings — count and %)
4. Combined overlap rate across all 438 rows

---

#### SECTION 2 — MODIFICATION ANALYSIS

**Question: For records that already exist in DIM, what is the team changing or adding?**

This applies to rows with a confirmed SF Activity ID match.

```sql
-- Summary counts of what is changing
SELECT
    -- Description
    SUM(CASE WHEN c.SUMMARY IS NOT NULL AND c.SUMMARY != '' THEN 1 ELSE 0 END)
        AS csv_rows_with_ai_summary,
    SUM(CASE WHEN c.SUMMARY IS NOT NULL AND c.SUMMARY != ''
             AND d.ACTIVITY_DESCRIPTION IS NOT NULL AND d.ACTIVITY_DESCRIPTION != '' THEN 1 ELSE 0 END)
        AS dim_already_had_description,
    SUM(CASE WHEN c.SUMMARY IS NOT NULL AND c.SUMMARY != ''
             AND (d.ACTIVITY_DESCRIPTION IS NULL OR d.ACTIVITY_DESCRIPTION = '') THEN 1 ELSE 0 END)
        AS csv_adds_net_new_description,

    -- Use Case ID
    SUM(CASE WHEN (c.USE_CASE_ID IS NOT NULL AND c.USE_CASE_ID != '')
             AND  (d.USE_CASE_ID IS NULL      OR  d.USE_CASE_ID  = '') THEN 1 ELSE 0 END)
        AS csv_adds_use_case_id_not_in_dim,
    SUM(CASE WHEN (c.USE_CASE_ID IS NOT NULL AND c.USE_CASE_ID != '')
             AND  (d.USE_CASE_ID IS NOT NULL  AND d.USE_CASE_ID  != '')
             AND   c.USE_CASE_ID != d.USE_CASE_ID THEN 1 ELSE 0 END)
        AS use_case_id_conflict,

    -- Opportunity ID
    SUM(CASE WHEN (c.OPPORTUNITY_ID IS NOT NULL AND c.OPPORTUNITY_ID != '')
             AND  (d.OPP_ID IS NULL             OR  d.OPP_ID  = '') THEN 1 ELSE 0 END)
        AS csv_adds_opp_id_not_in_dim,
    SUM(CASE WHEN (c.OPPORTUNITY_ID IS NOT NULL AND c.OPPORTUNITY_ID != '')
             AND  (d.OPP_ID IS NOT NULL          AND d.OPP_ID != '')
             AND   c.OPPORTUNITY_ID != d.OPP_ID THEN 1 ELSE 0 END)
        AS opp_id_conflict,

    COUNT(*) AS total_matched_rows
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES c
JOIN SALES.SE_REPORTING.DIM_SE_ACTIVITY d ON c.SF_ACTIVITY_ID = d.ACTIVITY_ID
WHERE c.SF_ACTIVITY_ID IS NOT NULL AND c.SF_ACTIVITY_ID != '';
```

```sql
-- Row-level detail of what is changing (first 25 rows)
SELECT
    c.MEETING_SE_NAME,
    c.MEETING_DATE,
    c.MEETING_TITLE,
    c.SF_ACTIVITY_ID,

    CASE
        WHEN c.SUMMARY IS NOT NULL AND c.SUMMARY != ''
             AND d.ACTIVITY_DESCRIPTION IS NOT NULL AND d.ACTIVITY_DESCRIPTION != ''
        THEN 'Enriched — CSV adds AI summary alongside existing DIM description'
        WHEN c.SUMMARY IS NOT NULL AND c.SUMMARY != ''
             AND (d.ACTIVITY_DESCRIPTION IS NULL OR d.ACTIVITY_DESCRIPTION = '')
        THEN 'Net new description — DIM had none'
        ELSE 'No summary added'
    END AS description_change_type,

    CASE
        WHEN (c.USE_CASE_ID IS NOT NULL AND c.USE_CASE_ID != '')
             AND (d.USE_CASE_ID IS NULL OR d.USE_CASE_ID = '')
        THEN 'CSV adds UC ID — DIM had none ✓'
        WHEN (c.USE_CASE_ID IS NOT NULL AND c.USE_CASE_ID != '')
             AND (d.USE_CASE_ID IS NOT NULL AND d.USE_CASE_ID != '')
             AND c.USE_CASE_ID = d.USE_CASE_ID
        THEN 'UC ID matches ✓'
        WHEN (c.USE_CASE_ID IS NOT NULL AND c.USE_CASE_ID != '')
             AND (d.USE_CASE_ID IS NOT NULL AND d.USE_CASE_ID != '')
             AND c.USE_CASE_ID != d.USE_CASE_ID
        THEN 'UC ID CONFLICT ✗'
        ELSE 'No UC ID in CSV'
    END AS use_case_change_type,

    CASE
        WHEN (c.OPPORTUNITY_ID IS NOT NULL AND c.OPPORTUNITY_ID != '')
             AND (d.OPP_ID IS NULL OR d.OPP_ID = '')
        THEN 'CSV adds Opp ID — DIM had none ✓'
        WHEN (c.OPPORTUNITY_ID IS NOT NULL AND c.OPPORTUNITY_ID != '')
             AND (d.OPP_ID IS NOT NULL AND d.OPP_ID != '')
             AND c.OPPORTUNITY_ID = d.OPP_ID
        THEN 'Opp ID matches ✓'
        WHEN (c.OPPORTUNITY_ID IS NOT NULL AND c.OPPORTUNITY_ID != '')
             AND (d.OPP_ID IS NOT NULL AND d.OPP_ID != '')
             AND c.OPPORTUNITY_ID != d.OPP_ID
        THEN 'Opp ID CONFLICT ✗'
        ELSE 'No Opp ID in CSV'
    END AS opp_change_type,

    LEFT(d.ACTIVITY_DESCRIPTION, 80) AS dim_existing_description,
    LEFT(c.SUMMARY, 120)             AS csv_ai_summary_preview

FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES c
JOIN SALES.SE_REPORTING.DIM_SE_ACTIVITY d ON c.SF_ACTIVITY_ID = d.ACTIVITY_ID
WHERE c.SF_ACTIVITY_ID IS NOT NULL AND c.SF_ACTIVITY_ID != ''
ORDER BY c.MEETING_SE_NAME, c.MEETING_DATE
LIMIT 25;
```

For this section, answer:
- Are the team's changes **additive** (adding new data to existing records) or **contradictory**
  (overwriting correct existing values with wrong ones)?
- What is the character of the description change — AI summaries are substantively different
  from the brief `[Sent]/[Received] {meeting title}` format in DIM. Is that enrichment?
- Are Use Case ID additions valid? Do any conflict?

---

#### SECTION 3 — SF ACTIVITY ID HEALTH CHECK

**Question: When referencing existing records, is the correct SF Activity ID provided?
When logging a new record, is SF Activity ID appropriately blank?**

**Part A — ID validity when provided:**
```sql
SELECT
    COUNT(*)                                                                               AS total_rows_with_sf_id,
    SUM(CASE WHEN d.ACTIVITY_ID IS NOT NULL THEN 1 ELSE 0 END)                            AS valid_ids_found_in_dim,
    SUM(CASE WHEN d.ACTIVITY_ID IS NULL     THEN 1 ELSE 0 END)                            AS invalid_ids_not_in_dim,
    ROUND(SUM(CASE WHEN d.ACTIVITY_ID IS NOT NULL THEN 1.0 ELSE 0 END) / COUNT(*) * 100, 1) AS pct_valid
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES c
LEFT JOIN SALES.SE_REPORTING.DIM_SE_ACTIVITY d ON c.SF_ACTIVITY_ID = d.ACTIVITY_ID
WHERE c.SF_ACTIVITY_ID IS NOT NULL AND c.SF_ACTIVITY_ID != '';
```

If any IDs are invalid, list them explicitly:
```sql
SELECT c.MEETING_SE_NAME, c.MEETING_DATE, c.MEETING_TITLE, c.SF_ACTIVITY_ID, c.SOURCE_FILE
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES c
LEFT JOIN SALES.SE_REPORTING.DIM_SE_ACTIVITY d ON c.SF_ACTIVITY_ID = d.ACTIVITY_ID
WHERE c.SF_ACTIVITY_ID IS NOT NULL AND c.SF_ACTIVITY_ID != ''
  AND d.ACTIVITY_ID IS NULL;
```

**Part B — Source vs. blank SF Activity ID (expected behavior check):**
```sql
SELECT
    SOURCE,
    COUNT(*)                                                                              AS row_count,
    SUM(CASE WHEN SF_ACTIVITY_ID IS NULL OR SF_ACTIVITY_ID = '' THEN 1 ELSE 0 END)       AS blank_sf_id,
    SUM(CASE WHEN SF_ACTIVITY_ID IS NOT NULL AND SF_ACTIVITY_ID != '' THEN 1 ELSE 0 END) AS has_sf_id,
    ROUND(SUM(CASE WHEN SF_ACTIVITY_ID IS NOT NULL AND SF_ACTIVITY_ID != '' THEN 1.0 ELSE 0 END)
          / COUNT(*) * 100, 1)                                                            AS pct_with_sf_id
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES
GROUP BY SOURCE
ORDER BY row_count DESC;
```

Evaluate:
- Google Calendar rows should mostly have blank SF Activity IDs. That is **expected and correct**.
- Gong/Zoom rows should have populated SF Activity IDs. If they are blank, that is a data gap —
  those calls were not matched to a Setsail record.
- Any row with a populated SF Activity ID that does not match DIM is a **data quality error**.

---

#### SECTION 4 — SAMPLE EVIDENCE

Run these to gather representative examples for the report.

```sql
-- 5 examples of confirmed bulk overlap (SF ID matches, minimal enrichment)
SELECT
    c.MEETING_SE_NAME, c.MEETING_DATE, c.MEETING_TITLE, c.SF_ACTIVITY_ID,
    LEFT(d.ACTIVITY_DESCRIPTION, 80)  AS dim_description,
    LEFT(c.SUMMARY, 80)               AS csv_summary
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES c
JOIN SALES.SE_REPORTING.DIM_SE_ACTIVITY d ON c.SF_ACTIVITY_ID = d.ACTIVITY_ID
WHERE c.SF_ACTIVITY_ID IS NOT NULL AND c.SF_ACTIVITY_ID != ''
  AND (c.SUMMARY IS NULL OR c.SUMMARY = '')
  AND (c.USE_CASE_ID IS NULL OR c.USE_CASE_ID = '' OR c.USE_CASE_ID = d.USE_CASE_ID)
LIMIT 5;
```

```sql
-- 5 examples of good enrichment (SF ID matches, CSV adds UC ID and AI summary)
SELECT
    c.MEETING_SE_NAME, c.MEETING_DATE, c.MEETING_TITLE, c.SF_ACTIVITY_ID,
    LEFT(d.ACTIVITY_DESCRIPTION, 80) AS dim_description,
    LEFT(c.SUMMARY, 120)             AS csv_ai_summary,
    c.USE_CASE_ID                    AS csv_use_case_id,
    d.USE_CASE_ID                    AS dim_use_case_id
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES c
JOIN SALES.SE_REPORTING.DIM_SE_ACTIVITY d ON c.SF_ACTIVITY_ID = d.ACTIVITY_ID
WHERE c.SF_ACTIVITY_ID IS NOT NULL AND c.SF_ACTIVITY_ID != ''
  AND c.SUMMARY IS NOT NULL AND c.SUMMARY != ''
  AND c.USE_CASE_ID IS NOT NULL AND c.USE_CASE_ID != ''
  AND (d.USE_CASE_ID IS NULL OR d.USE_CASE_ID = '')
LIMIT 5;
```

```sql
-- 5 examples of GCal-only rows that appear to be already in DIM (potential silent duplicates)
SELECT
    c.MEETING_SE_NAME, c.MEETING_DATE, c.MEETING_TITLE  AS csv_title,
    c.SOURCE                                             AS csv_source,
    d.ACTIVITY_DESCRIPTION                               AS dim_description,
    d.ACTIVITY_ID                                        AS potential_dim_match
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES c
JOIN SALES.SE_REPORTING.DIM_SE_ACTIVITY d
    ON  d.ACTIVITY_SE_NAME = c.MEETING_SE_NAME
    AND d.ACTIVITY_DATE    = c.MEETING_DATE
    AND TRIM(LOWER(d.ACTIVITY_DESCRIPTION)) LIKE '%' || TRIM(LOWER(c.MEETING_TITLE)) || '%'
    AND d.SOURCE = 'Setsail'
    AND d.ACTIVITY_TYPE = 'Meeting'
WHERE (c.SF_ACTIVITY_ID IS NULL OR c.SF_ACTIVITY_ID = '')
LIMIT 5;
```

```sql
-- 5 examples of truly new records (GCal-only, no match in DIM at all on that date)
SELECT c.MEETING_SE_NAME, c.MEETING_DATE, c.MEETING_TITLE, c.CUSTOMER, c.SOURCE
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES c
WHERE (c.SF_ACTIVITY_ID IS NULL OR c.SF_ACTIVITY_ID = '')
  AND NOT EXISTS (
    SELECT 1 FROM SALES.SE_REPORTING.DIM_SE_ACTIVITY d
    WHERE d.ACTIVITY_SE_NAME = c.MEETING_SE_NAME
      AND d.ACTIVITY_DATE    = c.MEETING_DATE
      AND d.SOURCE           = 'Setsail'
      AND d.ACTIVITY_TYPE    = 'Meeting'
  )
LIMIT 5;
```

```sql
-- Any conflicts (SF ID present but UC ID or Opp ID values differ from DIM)
SELECT
    c.MEETING_SE_NAME, c.MEETING_DATE, c.MEETING_TITLE, c.SF_ACTIVITY_ID,
    c.USE_CASE_ID AS csv_use_case_id, d.USE_CASE_ID AS dim_use_case_id,
    c.OPPORTUNITY_ID AS csv_opp_id,   d.OPP_ID      AS dim_opp_id
FROM TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES c
JOIN SALES.SE_REPORTING.DIM_SE_ACTIVITY d ON c.SF_ACTIVITY_ID = d.ACTIVITY_ID
WHERE c.SF_ACTIVITY_ID IS NOT NULL AND c.SF_ACTIVITY_ID != ''
  AND (
       (c.USE_CASE_ID IS NOT NULL AND c.USE_CASE_ID != '' AND d.USE_CASE_ID IS NOT NULL AND d.USE_CASE_ID != '' AND c.USE_CASE_ID != d.USE_CASE_ID)
    OR (c.OPPORTUNITY_ID IS NOT NULL AND c.OPPORTUNITY_ID != '' AND d.OPP_ID IS NOT NULL AND d.OPP_ID != '' AND c.OPPORTUNITY_ID != d.OPP_ID)
  );
```

---

### REPORT OUTPUT FORMAT

Write the final report in this exact structure. Fill in all bracketed placeholders with
real numbers from the queries above.

```
# EDA Activity Tracking — Overlap & Data Quality Report

**Period:** [period_start] — [period_end]
**Team members analyzed:** Jim Lebonitte, Patrick Sheehan, Michael Hamilton, Phani Alapaty, Steve Mitchener
**Report generated:** [today's date]
**Data sources:**
- Production: SALES.SE_REPORTING.DIM_SE_ACTIVITY
- New method: TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES

---

## Executive Summary

[2–3 sentence plain-language summary of the most important findings.]

| Category | Count | % of Total |
|---|---|---|
| Total CSV rows loaded | [N] | — |
| Rows with SF Activity ID (confirmed references to existing records) | [N] | [N]% |
| — SF Activity IDs that are valid (found in DIM) | [N] | [N]% of above |
| — SF Activity IDs that are invalid (not found in DIM) | [N] | [N]% of above |
| Rows without SF Activity ID (GCal-only, unmatched) | [N] | [N]% |
| — Of which: probable match in DIM by date + title | [N] | [N]% of above |
| — Of which: appear truly new (no DIM match) | [N] | [N]% of above |

---

## Section 1: Bulk Load Overlap

**Are they loading data that already exists in DIM?**

[Narrative answer based on query results. Distinguish between:
- Confirmed overlap via SF Activity ID
- Probable overlap via title match for GCal rows
- Truly new records with no DIM counterpart]

[Include the numeric breakdown table]

**Assessment:** [Is this a problem? Should the team be doing anything differently?]

---

## Section 2: What Is Being Changed

**For records that already exist in DIM, what is the team adding or modifying?**

| Change Type | Count | % of Matched Rows |
|---|---|---|
| Rows with AI-generated summary added | [N] | [N]% |
| — Of which DIM already had a description | [N] | [N]% |
| — Of which DIM had no description (net-new) | [N] | [N]% |
| Rows adding a Use Case ID not in DIM | [N] | [N]% |
| Use Case ID conflicts (CSV and DIM disagree) | [N] | [N]% |
| Rows adding an Opportunity ID not in DIM | [N] | [N]% |
| Opportunity ID conflicts | [N] | [N]% |

**Character of description changes:** [Explain the AI summary vs. DIM email-subject format.
Is this enrichment or noise?]

**Assessment:** [Are changes additive and safe, or do any present data integrity risk?]

---

## Section 3: SF Activity ID Health

**Part A — ID validity when provided:**
[Total rows with SF ID, how many are valid, how many are invalid]
[If any invalid: list them by SE name, date, title, and the bad ID]

**Part B — Source vs. blank SF Activity ID:**
[Table from the source breakdown query]
[Assessment: are GCal rows appropriately blank? Are Gong/Zoom rows appropriately populated?
Flag any Gong/Zoom rows with blank IDs as data gaps.]

**Overall health rating:** [Good / Needs attention / Critical — with brief justification]

---

## Section 4: Sample Evidence

**Confirmed bulk overlap (SF ID matches, little enrichment):**
[5-row table from query]

**Good enrichment (SF ID matches, CSV adds UC ID + AI summary):**
[5-row table from query]

**GCal rows likely already in DIM (silent duplicates):**
[5-row table from query]

**Truly new records (no DIM match):**
[5-row table from query]

**Conflicts (CSV and DIM disagree on field values):**
[Table from query, or "No conflicts found"]

---

## Section 5: Recommendations

Based on the data above, provide recommendations on:

1. **Bulk load behavior** — Should the team continue loading all GCal meetings even when
   Setsail already captured them? Is there value in the duplicated records?

2. **Enrichment value** — The AI summaries and Use Case IDs being added are substantive.
   Should this data flow back into the production system? If so, what is the merge/upsert
   mechanism?

3. **SF Activity ID completeness** — Gong/Zoom rows without an SF Activity ID represent
   calls that weren't matched to a Setsail record. What should the team do with these?

4. **New records (no DIM match)** — The ~[N] GCal rows that Setsail did not capture represent
   potential under-counting in the production system. Should these be reviewed for manual entry?

5. **Data conflicts** — If any UC ID or Opp ID conflicts were found, list specific records that
   need manual review and correction.
```

---

## END OF PROMPT

---

## Reference — Pre-run Findings (April–June 2026 snapshot)

These numbers were verified interactively before the prompt was written. Use them to sanity-check
the output when running the prompt.

| Metric | Value |
|---|---|
| Total staging rows | 438 |
| Date range | 2026-04-06 – 2026-06-18 |
| Rows with SF Activity ID | 77 (17.6%) |
| SF Activity IDs found in DIM | 77 / 77 (100% — all valid) |
| Rows without SF Activity ID | 361 (82.4%) |
| GCal rows with title match in DIM | ~219 / 361 (60.7%) |
| GCal rows with no DIM match (truly new) | ~142 / 361 (39.3%) |
| Matched rows where description differs (AI summary vs. brief email subject) | 70 / 77 (90.9%) |
| Matched rows where CSV adds Use Case ID not in DIM | 62 / 77 (80.5%) |
| Use Case ID conflicts | 0 |
| Opportunity ID conflicts | 1 |
