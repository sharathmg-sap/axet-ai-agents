---
name: replication-table-lookup
description: Use when the user gives a table/object name and asks where it is replicated or which .hdbreptask file(s) reference it.
---

# Replication table lookup (accurate, evidence-backed)

## When to use this skill
Use this workflow when the user provides a table/object name (e.g., `BSID`) and wants to know which `.hdbreptask` replication task(s) reference/replicate it.

## How to use this skill

### Objective
Return **accurate** `.hdbreptask` matches for the requested table(s) with **proof (evidence snippets)**, and an explicit **completeness** statement. Never stop at the first plausible hit.

### Inputs
- **tables**: 1+ table/object names (case-insensitive). Treat each independently.
- **projects**: user-specified; otherwise default to `HC_ERP` and `HC_S4H`.
- **branch**: user-specified; otherwise use each project’s default branch.

### Hard constraints (must/never)
- **MUST search all `.hdbreptask` files** in the requested scope before concluding; **never** stop at the first hit.
- **MUST distinguish match types**:
  - `exact_token`: whole-identifier match of the table name
  - `partial_text`: substring match (report separately)
- **MUST provide evidence** for every reported match:
  - include a short **verbatim snippet** (few lines) from the `.hdbreptask` showing the matched table/mapping.
- **MUST validate the returned file**:
  - If recommending a specific `.hdbreptask`, also list the **distinct source tables** present in that file (deduped) so the user can sanity-check scope.
- **NEVER conflate tables**:
  - If user mentions multiple tables (e.g., BSID and BSAD), report `found / not found` per table separately.
- **MUST state completeness**:
  - declare `Complete` only if all target projects/branches were searched successfully; otherwise `Partial` with reason(s).

### Matching rules (precision)
- Match case-insensitively as a **whole identifier**, not as substring.
  - Use token boundaries equivalent to: `(?i)(^|[^A-Z0-9_])<TABLE>([^A-Z0-9_]|$)`
- Prefer `exact_token` matches inside a mapping/source section over incidental mentions.

### Procedure
1. Resolve target projects via connector project search/list.
   - If ambiguous, ask user to pick the intended project(s) before searching.
2. For each project/branch:
   - Enumerate all files matching `**/*.hdbreptask` (repository tree).
   - Search within those files for the requested table(s) using the identifier-boundary rule.
3. For each match:
   - Open the file content and extract:
     - file path
     - the evidence snippet lines containing the table/mapping
     - (if present) extracted mapping tuple(s) like `SOURCE_SCHEMA.TABLE → TARGET_TABLE` and any view/virtual table name (e.g., `VT_BSID`)
     - (if present) replication mode indicators (`initial`, `real-time`, etc.)
4. If the user asks for “the replication task” (singular) and multiple exact matches exist:
   - Return **all exact matches**; optionally rank by strength:
     1) match in mapping tuple
     2) match in source table list
     3) other exact token match
   - Do not arbitrarily pick one without disclosing the others.
5. If presenting a specific `.hdbreptask` as the best match:
   - List the **distinct source tables** referenced in that `.hdbreptask` (deduped).
6. Report results and completeness.

### Output format (required)
Provide a table per requested table:

| table | project | branch | reptask_path | match_type | evidence_snippet | extracted_source→target | replication_mode |
|---|---|---|---|---|---|---|---|

Then:

**Completeness:** `Complete` / `Partial` — include which projects/branches searched and any failures.

**Notes:** Only include observations directly supported by extracted file content (no inference).
