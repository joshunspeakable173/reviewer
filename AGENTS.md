# AGENTS.md

## Project purpose
This repository builds and runs a multi-agent reviewer for academic economics papers.

## Primary learning goal
This project is also a training environment for learning how to work effectively with Codex:
- project-scoped instructions
- reusable Skills
- custom agents
- non-interactive runs with `codex exec`
- structured outputs
- safe multi-agent orchestration

## Non-negotiable workflow
1. Never review a raw PDF directly if parsed artifacts are missing.
2. Always run `scripts/preprocess_pdf.py` first.
3. Reviewer agents output strict JSON only.
4. The final editor is the only component that outputs the final Markdown report.
5. Use web verification for literature and reference auditing.
6. If a claim cannot be verified exactly, return `cannot_verify` rather than guessing.
7. Preserve exact evidence locations whenever possible:
   - PDF page
   - section
   - table / figure / appendix item
   - reference-list line or entry
8. Do not rewrite the paper unless a separate repair mode is explicitly requested.
9. Do not soften findings. If a claim is too strong, flag it.
10. Do not invent citations, DOIs, URLs, or page numbers.

## Output contract
A successful run produces:
- parsed artifacts under `work/<paper_id>/parsed/`
- reviewer JSON files under `work/<paper_id>/reviews/`
- a final Markdown report at `outputs/<paper_id>/report.md`
- a manifest at `outputs/<paper_id>/manifest.json`

## Failure policy
- Invalid JSON output = failed reviewer
- Missing required fields = failed reviewer
- `cannot_verify` is allowed and does not count as failure
- The orchestrator may retry one failed reviewer once

## Done criteria
A run is done only when:
- all reviewer JSON files validate against their schemas
- the final report exists
- the manifest exists
- all reviewer commands exited successfully