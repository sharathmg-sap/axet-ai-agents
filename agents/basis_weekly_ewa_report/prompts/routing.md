---
name: routing
description: Simple routing instructions for basis_weekly_ewa_report agent.
version: "1.0"
---

# Routing

## If user provides DOC/DOCX content
- Use `skills/ewa_summary` to generate:
  - Summary document (default), OR
  - Slides outline if user specifies slide count or explicitly requests slides.

## If user asks for a different format
- Ask a clarifying question only if required (e.g., slide count not specified but slides requested).
