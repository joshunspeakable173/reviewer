---
name: paper-reviewer
description: Use this skill when the task is to review an academic paper PDF with multiple specialized auditors and compile a final report.
---

# Paper Reviewer Skill

This skill runs a multi-agent paper-review workflow.

## Use this skill when
- the input is an academic paper PDF
- the task is review, auditing, or verification
- the user wants a structured report
- the task involves literature claims, references, numeric checks, or internal cross-references

## Do not use this skill when
- the user wants only a summary
- the user wants only proofreading
- the user wants only a rewrite
- parsed artifacts already exist and the request is unrelated to the review pipeline

## Workflow
1. Ensure parsed artifacts exist under `work/<paper_id>/parsed/`.
2. If not, run `scripts/preprocess_pdf.py`.
3. Launch the five reviewers:
   - literature_auditor
   - numerical_auditor
   - reference_auditor
   - crossref_auditor
   - claim_evidence_auditor
4. Validate each JSON output.
5. Launch `editor` on the validated reviewer outputs.
6. Write the final report to `outputs/<paper_id>/report.md`.

## Critical rules
- Internal reviewers return JSON only.
- The editor is the only component that emits final markdown.
- Literature and reference verification require web search.
- Never guess missing evidence; use `cannot_verify`.
- Preserve exact source locations whenever possible.
