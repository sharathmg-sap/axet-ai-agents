# basis_weekly_ewa_report — Agent Spec

## Goal
Summarize SAP EWA reports (DOC/DOCX input) into a fixed, **exactly 2-page/2-slide** BASIS-friendly PPTX (or PDF fallback) output:
1. **Performance Indicators** table (Value | Metric) — 9 fixed metrics.
2. **Early Watch Reports** table (# | Topic | Recommendation) — up to 6-8 prioritized findings.

Only deviate from this fixed 2-slide/table format if the user explicitly requests a different slide count or output style.

## Non-goals
- No external SAP knowledge injection beyond what is explicitly written in the input.
- No remediation steps beyond the report's stated recommendations.
- No additional slides/sections (Executive Summary, Security Findings, etc.) by default — these are out of scope unless explicitly requested.

## Output modes
1. **Fixed 2-page/2-table output (default)** — Performance Indicators table + Early Watch Reports table.
2. **Slides outline with custom count (only when explicitly requested)** — falls back to broader section groupings (Executive Summary, Key Risks, Recommended Actions, Key Metrics Snapshot) to fill the requested slide count.

## Key acceptance criteria
- Default output is always exactly 2 slides/pages, formatted as tables (not prose/bullets).
- Slide 1 contains exactly these 9 metric rows, in order: Active Users (>400 steps), Avg. Availability per Week, Avg. Response Time in Dialog Task, Max. Dialog Steps per Hour, Avg. Response Time at Peak Dialog Hour, Avg. Response Time in RFC Task, Avg. RFC Response Time at Peak Hour, DB Size, DB Growth Last Month.
- Slide 2 contains up to 6-8 findings rows sourced from (in priority order): Alert Overview, ABAP Dumps (ST22), Red-rated checks, Gateway/error logs.
- All findings and metric values traceable to input text.
- Missing info clearly marked as "Not stated in input" (rows are never dropped due to missing values).
- Slide mode respects user-provided slide count only when explicitly requested; otherwise the fixed 2-slide default applies.
