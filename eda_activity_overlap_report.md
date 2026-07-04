# EDA Activity Tracking — Overlap & Data Quality Report

**Period:** 2026-04-06 — 2026-06-18  
**Team members analyzed:** Jim Lebonitte, Patrick Sheehan, Michael Hamilton, Phani Alapaty, Steve Mitchener  
**Report generated:** 2026-06-25  
**Data sources:**
- Production: `SALES.SE_REPORTING.DIM_SE_ACTIVITY`
- New method: `TEMP.JLEBONITTE_EDA_ACTIVITY_TRACKING.SE_WEEKLY_ACTIVITIES`

---

## Executive Summary

The EDA team's new method is producing high-quality data that complements — not corrupts — the production system. Every SF Activity ID they provide is valid, and when they reference an existing record, they consistently add value (AI-generated meeting summaries and Use Case IDs) rather than overwriting correct data. The primary concern is not data quality but completeness: 82% of their CSV rows lack an SF Activity ID, meaning they are calendar-only records that were never matched to Setsail. Of those, roughly 60% are already in the production system (Setsail captured them automatically), while the remaining ~39% represent meetings that Setsail missed entirely and that could represent legitimate under-counting in the production system. There is exactly one data conflict requiring manual attention.

| Category | Count | % of Total (438) |
|---|---|---|
| Total CSV rows loaded | 438 | — |
| Rows **with** SF Activity ID (confirmed reference to existing record) | 77 | 17.6% |
| — SF Activity IDs that are **valid** (found in DIM) | 77 | 100% of above |
| — SF Activity IDs that are **invalid** (not found in DIM) | 0 | 0% |
| Rows **without** SF Activity ID (GCal-only, unmatched) | 361 | 82.4% |
| — Of which: probable match in DIM by date + title | 219 | 60.7% of above |
| — Of which: appear **truly new** (no DIM match on that date for that SE) | 142 | 39.3% of above |

---

## Section 1: Bulk Load Overlap

**Are they loading data that already exists in DIM?**

Yes — the majority of what the team loads is data that Setsail already captured in the production system. However, whether this counts as a "problem" depends on intent.

### Case A — Records with SF Activity ID (confirmed overlap)

All 77 rows where the team provided an SF Activity ID match exactly to records already in `DIM_SE_ACTIVITY`. The overlap rate is **100%** for this category, which is the expected and correct behavior: the SF Activity ID is *derived* from the Setsail record during the matching step in `snowhouse_week.sql`, so it should always point to an existing record.

### Case B — GCal rows without SF Activity ID (probable silent overlap)

Of the 361 rows with no SF Activity ID, **219 (60.7%)** have a meeting title that appears in `DIM_SE_ACTIVITY` for the same SE on the same date. These are meetings that Setsail auto-captured (it processes all Google Calendar events with external attendees), meaning the production system already has them. These rows are loaded by the new method in addition to the production system, not instead of it.

The remaining **142 rows (39.3%)** have no match of any kind in `DIM_SE_ACTIVITY` on that SE's date. These are meetings that Setsail did not log — likely because the meeting had no external attendees, exceeded the 25-participant threshold, or was an internal-only call that the team manually included.

### Per-SE breakdown

| Team Member | Total CSV Rows | Confirmed Overlap (ID match) | GCal Probable Duplicate | Likely Truly New |
|---|---|---|---|---|
| Jim Lebonitte | 231 | 33 | 129 | 69 |
| Phani Alapaty | 103 | 13 | 40 | 46 |
| Patrick Sheehan | 42 | 11 | 16 | 15 |
| Michael Hamilton | 33 | 14 | 13 | 6 |
| Steve Mitchener | 29 | 6 | 21 | 2 |
| **Total** | **438** | **77** | **219** | **142** |

**Assessment:** The bulk load overlap is structurally expected, not a process error. The new method pulls from Google Calendar first, then cross-references against Gong/Zoom to resolve SF Activity IDs. GCal rows that don't resolve to a Setsail match will always appear to "duplicate" what Setsail already captured. The meaningful question is whether the ~142 truly new records should be reviewed for entry into the production system.

---

## Section 2: What Is Being Changed

**For the 77 records where the team references an existing DIM record via SF Activity ID, what are they adding or modifying?**

### Summary counts

| Change Type | Count | % of 77 Matched Rows |
|---|---|---|
| Rows providing an AI-generated meeting summary | 70 | 90.9% |
| — Of which: DIM already had a description (enrichment) | 70 | 100% of above |
| — Of which: DIM had no description at all (net-new) | 0 | 0% |
| Rows adding a Use Case ID not present in DIM | 62 | 80.5% |
| Use Case ID conflicts (CSV and DIM disagree) | 0 | 0% |
| Rows adding an Opportunity ID not present in DIM | 1 | 1.3% |
| Opportunity ID conflicts (CSV and DIM disagree) | 1 | 1.3% |

### Per-SE breakdown

| Team Member | Matched Rows | Has AI Summary | Adds UC ID | UC Conflict | Adds Opp ID | Opp Conflict |
|---|---|---|---|---|---|---|
| Jim Lebonitte | 33 | 33 | 24 | 0 | 0 | 0 |
| Michael Hamilton | 14 | 13 | 13 | 0 | 1 | 1 |
| Phani Alapaty | 13 | 9 | 10 | 0 | 0 | 0 |
| Patrick Sheehan | 11 | 11 | 9 | 0 | 0 | 0 |
| Steve Mitchener | 6 | 4 | 6 | 0 | 0 | 0 |

### Character of description changes

The production system stores descriptions in the format `[Sent] Meeting Title` or `[Received] Meeting Title` — essentially the email subject line from the calendar invite. The new method replaces this with a 2–4 sentence AI-generated narrative that includes attendee names, topics discussed, and outcomes. These are substantively different in value. Example:

| | Content |
|---|---|
| **DIM description** | `[Sent] Snowflake / Inovalon - Data Teams of the Future` |
| **CSV AI summary** | *"Subhash, Michael, Jim and Jessica from Snowflake, along with Raghavendra, Rakesh and Alfred, discussed Snowflake's vision for 'Data Teams of the Future,' focusing on automating platform workflows and enabling self-service through agents and codified governance..."* |

The CSV version carries meaningful signal. This is enrichment, not noise.

### Use Case ID additions

62 of 77 matched rows provide a Use Case ID that is absent from the corresponding DIM record. None of these conflict with an existing UC ID in DIM — every case is an addition, not a correction. This means the new method is successfully linking meetings to open use cases that Setsail did not tag.

**Assessment:** The team's changes are entirely additive and safe. They are enriching records with AI summaries and adding Use Case linkages that the automated Setsail system missed. There are no cases where correct existing data is being overwritten with wrong values — with one exception.

---

## Section 3: SF Activity ID Health

### Part A — ID validity when provided

| Metric | Value |
|---|---|
| Total rows with SF Activity ID | 77 |
| Valid IDs found in DIM | **77 (100%)** |
| Invalid IDs not found in DIM | **0** |

No orphaned or fabricated IDs. Every SF Activity ID the team provided resolves to a real, existing Setsail activity record. This is the expected output of the `snowhouse_week.sql` matching logic.

### Part B — Source vs. blank SF Activity ID

| Source | Row Count | Blank SF ID | Has SF ID | % with SF ID |
|---|---|---|---|---|
| Google Calendar | 311 | 311 | 0 | 0.0% |
| Gong | 56 | 4 | 52 | 92.9% |
| Zoom | 55 | 32 | 23 | 41.8% |
| Calendar | 16 | 14 | 2 | 12.5% |

**Google Calendar rows:** All 311 are blank. This is correct and expected — GCal events that did not match a Gong or Zoom call recording have no basis for an SF Activity ID assignment. No issue here.

**Gong rows:** 52/56 (92.9%) have SF Activity IDs. The 4 blank Gong rows represent calls that were recorded in Gong but could not be matched to a Setsail activity. This is a minor data gap.

**Zoom rows:** Only 23/55 (41.8%) have SF Activity IDs. The **32 blank Zoom rows are a significant data gap** — these are Zoom meetings that the system could not match to a Setsail record. This is the largest source of unlinked data in the pipeline. Examples include "Hootsuite - Internal Sync," "Unified Lakehouse Architecture | Demo Review," and "Unacast <> Snowflake: working sessions." These Zoom calls likely represent real customer-facing meetings that Setsail did not capture.

**Calendar rows (mhamilton):** 14/16 (87.5%) blank. This is the same pattern as Google Calendar — meetings pulled from a calendar invite with no recording match.

**Overall health rating: Good with one area to watch.** SF Activity IDs are never fabricated and always valid when present. The main gap is the 32 Zoom rows without IDs — these calls occurred but were not linked back to the production system.

---

## Section 4: Sample Evidence

### Confirmed bulk overlap (SF ID match, no AI summary added)

These records are already in DIM and the CSV adds no new description — pure structural overlap with no enrichment value in this load.

| SE | Date | Meeting Title | SF Activity ID | DIM Description |
|---|---|---|---|---|
| Phani Alapaty | 2026-06-10 | Iceberg, Delta Direct Interoperability Labcorp | aPoVI000000DTUA0A4 | [Sent] Iceberg, Delta Direct Interoperability Labcorp |
| Steve Mitchener | 2026-05-01 | Snowflake \| Berklee: Architecture Review | aPoVI00000069fu0AA | [Sent] Snowflake \| Berklee: Architecture Review |
| Michael Hamilton | 2026-06-12 | Case 01373427: Internal error on ALTER STAGE… | aPoVI000000DiyC0AS | [Sent] Case 01373427: Internal error on ALTER STAGE SET DIRECTORY… |
| Phani Alapaty | 2026-06-10 | Iceberg, Delta Direct Interoperability Labcorp | aPoVI000000DSRD0A4 | [Sent] Iceberg, Delta Direct Interoperability Labcorp |
| Phani Alapaty | 2026-05-06 | Cigna Interop Architecture review with Snowflake | aPoVI0000005g7F0AQ | [Sent] Cigna Interop Architecture review with Snowflake(Phani A) |

### Good enrichment (SF ID match, CSV adds UC ID + AI summary)

These records demonstrate the core value of the new method — linking meetings to use cases and adding substantive narrative.

| SE | Date | Meeting Title | SF Activity ID | DIM UC ID | CSV UC ID Added | AI Summary Preview |
|---|---|---|---|---|---|---|
| Phani Alapaty | 2026-06-10 | Snowflake<>SoFi RDT (Q3 Roadmap review) | aPoVI000000DXUd0AO | *(none)* | aPaVI00000iDZgx0AG | Bernice, Phani and Arnold from Snowflake, along with Lillian, Svetlana, Maya… |
| Jim Lebonitte | 2026-05-05 | PepsiCo / Snowflake - Technical Deep Dive | aPoVI0000007hD70AI | *(none)* | aPaVI00000iGOcs0AG | Jeff, Jim, Vincent and Matthew from Snowflake, along with Joshua from PepsiCo… |
| Michael Hamilton | 2026-06-03 | NiCE \| Snowflake - Cortex Search weekly | aPoVI0000006GRi0AM | *(none)* | aPaVI00000iFB4m0AG | Ratheesh, Kapil, Morgan and Jeffrey from Snowflake discussed ongoing testing delays… |
| Jim Lebonitte | 2026-04-20 | Align on AI Pilot Strategy Ahead of Session w/ Girish | aPoVI0000005guL0AQ | *(none)* | aPaVI00000iFCCv0AO | David, Jim, Fru and Vijay from Snowflake, along with Shuvro and Venit from McKesson… |
| Jim Lebonitte | 2026-04-10 | Medtronic Dual Platform Governance Strategy Sync | aPoVI0000007hU10AI | *(none)* | aPaVI00000iFA4J0AW | Rob, Olga and Todd from Snowflake, along with Ravi, Brad and Babu from Medtronic… |

### GCal rows likely already in DIM (silent duplicates)

These calendar-only rows have no SF Activity ID but Setsail already captured the same meeting on the same date.

| SE | Date | CSV Title | DIM Description | DIM Activity ID |
|---|---|---|---|---|
| Jim Lebonitte | 2026-04-07 | Bloomberg: Architecture Review | [Sent] Bloomberg: Architecture Review | aPoVI0000009Xgb0AE |
| Jim Lebonitte | 2026-06-09 | Unified Lakehouse Architecture | [Received] [Hold] Unified Lakehouse Architecture Review Follow-Up #1 | aPoVI000000DRjP0AW |
| Phani Alapaty | 2026-06-04 | Iceberg, Delta Direct Interoperability Labcorp | [Sent] Iceberg, Delta Direct Interoperability Labcorp | aPoVI000000DSRD0A4 |
| Patrick Sheehan | 2026-05-18 | Snowflake/Raventrack: Project Reporting (GCP compete) | [Sent] Snowflake/Raventrack: Project Reporting (GCP compete) | aPoVI0000008h6Q0AQ |
| Jim Lebonitte | 2026-06-08 | Re: Horizon API, Network Flow, and SAS Token Behavior Discussion | [Received] Re: Horizon API, Network Flow, and SAS Token Behavior Discu… | aPoVI000000DPBe0AO |

### Truly new records (no DIM match on that date for that SE)

These 142 meetings were not captured by Setsail and exist only in the new method's data.

| SE | Date | Meeting Title | Customer | Source |
|---|---|---|---|---|
| Phani Alapaty | 2026-05-29 | Weekly meeting with Snowflake and AWS to work through tech arch | AbbVie Inc. | Google Calendar |
| Phani Alapaty | 2026-04-29 | Solution Engineering Welcome Call: William Summerhill / Brandon Adams | Fanatics Holdings Inc. | Zoom |
| Phani Alapaty | 2026-04-27 | Weekly Touchpoint Highspot / Snowflake | Highspot | Zoom |
| Phani Alapaty | 2026-05-07 | Internal - Country Financial Session Check-in and Dry Run | GoFundMe | Google Calendar |
| Jim Lebonitte | 2026-05-18 | CapOne Weekly Team Call | Capital One Services, LLC | Google Calendar |

### Conflicts (CSV and DIM disagree on field values)

There is exactly **one conflict** in the entire dataset.

| SE | Date | Meeting Title | SF Activity ID | CSV Opp ID | DIM Opp ID | CSV UC ID | DIM UC ID |
|---|---|---|---|---|---|---|---|
| Michael Hamilton | 2026-06-17 | NiCE \| Snowflake - Cortex Search weekly | aPoVI000000DIwy0AG | 006VI00000ZMTf7YAH | 006VI00000zVPeyYAG | aPaVI00000iFLnc0AG | *(none)* |

The CSV has a different Opportunity ID from DIM for this record. The UC ID addition is clean (DIM had none). This single record warrants a manual check to confirm which Opportunity ID is correct.

---

## Section 5: Recommendations

### 1. Bulk load behavior — continue as-is, but understand what you're getting

The overlap between the new method and production is by design. The new method is not trying to replace Setsail — it is adding a layer of narrative and context on top of it. Treat the 361 GCal-only rows as a separate reporting layer, not as a submission to the production system. If the intent ever changes to feeding data back into DIM, the ~219 probable duplicates should be deduplicated before any write operation.

### 2. Enrichment value — this data should flow back into production

The AI summaries and Use Case ID additions represent genuine intelligence that the production system lacks. 62 of 77 matched meetings now have a Use Case ID that Setsail never assigned. Consider an upsert process:
- For matched records (SF Activity ID present and valid): update `ACTIVITY_DESCRIPTION` in DIM with the AI summary, and update `USE_CASE_ID` / `OPP_ID` where DIM is null and CSV has a value.
- Do not overwrite non-null DIM values unless there is a manual review step.

### 3. Zoom rows without SF Activity ID — investigate the matching gap

32 Zoom rows (58% of all Zoom-sourced records) have no SF Activity ID, meaning the `snowhouse_week.sql` matching logic could not link them to a Setsail activity. These are not GCal-only meetings — they are Zoom-recorded calls that the system failed to join. The most likely cause is that the Setsail description does not contain enough of the meeting title to satisfy the `CONTAINS(..., SPLIT_PART(..., ' - ', 1))` match condition. Review the matching logic in `snowhouse_week.sql` for these specific titles.

### 4. Truly new records (~142) — review for Setsail capture eligibility

142 CSV rows represent meetings where Setsail has no record at all on that SE's date. Some may be internal-only calls that are legitimately outside Setsail's scope (the "Internal - Country Financial" and "CapOne Weekly Team Call" examples above fall into this category). Others may be external meetings that Setsail missed due to the external attendee detection rule. The team should review this list quarterly and flag any that should have been captured automatically.

### 5. The one conflict — requires manual resolution

The NiCE | Snowflake - Cortex Search weekly meeting (2026-06-17, Michael Hamilton, `aPoVI000000DIwy0AG`) has conflicting Opportunity IDs: the CSV supplies `006VI00000ZMTf7YAH` while DIM has `006VI00000zVPeyYAG`. Both IDs are present in Salesforce — one is the renewal opportunity and the other is a separate opportunity on the same account. Confirm with Michael Hamilton which opportunity this meeting should be credited to before any write-back occurs.
