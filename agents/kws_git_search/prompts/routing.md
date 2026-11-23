---
name: routing
description: Routing rules for kws_git_search (git-repo-lookup) to classify user intent into one capability.
version: "1.0"
---

# Routing — Git Repo Lookup (kws_git_search)

## Objective
Classify each incoming user request into **exactly one** capability branch and execute only that workflow.

## Branches

### 1) table_lookup_in_replication_tasks
Trigger when:
- User provides a **table/object name** and asks:
  - where it is replicated, or
  - which `.hdbreptask` file(s) reference it

Must enforce:
- Search **all** `.hdbreptask` files in-scope (no early stop)
- Whole-identifier matching (token boundary)
- Evidence snippet required for every match
- Distinct source tables list when recommending a reptask

### 2) object_existence_check
Trigger when:
- User asks whether an object exists in a project (function/view/procedure/file/etc.), not specifically replication tasks.

### 3) merge_request_search
Trigger when:
- User asks for merge requests, optionally filtered by state/author/date/text.

### 4) project_listing
Trigger when:
- User asks to list/search projects (optionally filtered by group/name).

### 5) ambiguous_or_mixed_request
Trigger when:
- Request spans multiple branches or scope is unclear.
Action:
- Return any unambiguous partial results, then use interactive question tool (ask_user/ask_followup_question) with 2–4 options to resolve remaining ambiguity.

### 6) no_actionable_input
Trigger when:
- No project/object/table identifier is present.
Action:
- Ask for the missing identifier (use interactive question tool where available).
