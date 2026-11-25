---
name: ewa_summary
description: Use when extracting SAP EarlyWatch Alert (EWA) report text (DOC/DOCX) into the fixed 2-page/2-table output — a Performance Indicators table and an Early Watch Reports (findings) table.
---

# Skill: EWA Summary Builder (DOC/DOCX)

## Purpose
Convert SAP EarlyWatch Alert (EWA) report text into a fixed, **exactly 2-page/2-table** output:
1. **Performance Indicators** table (Value | Metric)
2. **Early Watch Reports** table (# | Topic | Recommendation)

## Inputs
- Extracted text from DOC/DOCX (as provided by user)
- Optional: slide_count (integer) — only honored if the user explicitly requests a different count than the default 2

## Rules
- Use only input content. Do not guess missing values — write "Not stated in input".
- Default output is always exactly 2 slides/pages/tables (see Output below), regardless of how much content is in the source document.
- Only deviate from the 2-table default if slide_count is explicitly provided and differs from 2.

## Extraction Rules

### Table 1 — Performance Indicators (Value | Metric)
Pull only these 9 metrics, in this exact order, from the input's performance indicators table (e.g., "Performance Indicators for <SID>", Area/Indicators/Value columns):
1. Active Users (>400 steps)
2. Avg. Availability per Week
3. Avg. Response Time in Dialog Task
4. Max. Dialog Steps per Hour
5. Avg. Response Time at Peak Dialog Hour
6. Avg. Response Time in RFC Task
7. Avg. RFC Response Time at Peak Hour
8. DB Size
9. DB Growth Last Month

If a value isn't in the input, use "Not stated in input" for that row (never drop the row). Ignore all other KPI rows (Fiori Users, CPU Utilization, DB Request Times, etc.).

### Table 2 — Early Watch Reports (# | Topic | Recommendation)
Scan the input in this priority order for distinct findings:
1. Alert Overview bullets
2. Program Errors (ABAP Dumps) — dump types/counts if stated
3. Red-rated checks / red alert counts, if stated
4. Gateway logs / error logs — named log categories

Cap at 6–8 rows (prioritize Red/critical). Recommendation column = stated text if present, else "Not stated in input".

## Output
- Default: Markdown with exactly 2 tables, formatted per the Extraction Rules above, ready to hand to the `pptx` (or `pdf` fallback) skill as 2 slides/pages.
- Only if slide_count is explicitly provided and != 2: produce that many slides instead, using the broader section groupings (Executive Summary, Key Risks, Recommended Actions, Key Metrics Snapshot) as a fallback outline.
