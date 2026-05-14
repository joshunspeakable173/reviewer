# Parser Repair Agent Experiment

## Objective

Test whether adding another LLM agent after parser-quality preflight can address parser limitations and fix issues before parsed evidence is passed to substantive reviewers.

## Implementation Tested

The experiment adds an opt-in parser repair planner:

```powershell
.\.venv\Scripts\python.exe scripts\review_paper.py --pdf "inputs\my-paper.pdf" --parser-repair plan
```

When enabled, the wrapper:

1. preprocesses the PDF
2. renders initial prompts
3. runs `parser_quality_auditor`
4. validates and gates parser-quality findings
5. runs `scripts/run_parser_repair_agent.py`
6. writes `work/<paper_id>/repair/parser_repair_plan.json`
7. writes `work/<paper_id>/repair/parser_repair_notes.md`
8. passes the repair-notes path into reviewer selection and substantive reviewer prompts

The repair planner uses `schemas/parser_repair_plan.schema.json`. It is asked to cover each high- or medium-severity parser artifact, identify safe fallback artifacts, identify artifacts that should not be primary evidence, and distinguish mitigated issues from issues that still require deterministic preprocessing changes.

## Safety Constraint

A live repair-agent run on private prior-paper artifacts was not performed. The attempted live test on `work/paper9` was blocked because it would export private parsed-paper artifacts and parser-review outputs to an external service. That is the correct safety posture unless the user explicitly approves private data export for this experiment.

To test the LLM step without exporting private papers, the branch includes a synthetic fixture under:

```text
tests/fixtures/parser_repair/synthetic-parser/
```

The fixture contains no private paper text. It simulates two parser-quality defects:

- broken structured table extraction with usable raw-page fallback
- scrambled normalized page text with usable raw-page fallback

## Evidence

Live synthetic repair-agent command:

```powershell
.\.venv\Scripts\python.exe scripts\run_parser_repair_agent.py `
  --paper-id synthetic-parser `
  --parsed-dir tests\fixtures\parser_repair\synthetic-parser\parsed `
  --parser-quality-output tests\fixtures\parser_repair\synthetic-parser\reviews\parser_quality_auditor.json `
  --output-dir work\experiments\parser-repair-agent\synthetic-parser\repair `
  --notes-output work\experiments\parser-repair-agent\synthetic-parser\repair\parser_repair_notes.md `
  --log-dir work\experiments\parser-repair-agent\synthetic-parser\logs
```

Result:

- repair entries: 2
- target parser findings covered: 2 of 2
- generated plan: `work/experiments/parser-repair-agent/synthetic-parser/repair/parser_repair_plan.json`
- generated notes: `work/experiments/parser-repair-agent/synthetic-parser/repair/parser_repair_notes.md`

Scoring command:

```powershell
.\.venv\Scripts\python.exe scripts\evaluate_parser_repair.py `
  --papers synthetic-parser `
  --work-root tests\fixtures\parser_repair `
  --repair-root work\experiments\parser-repair-agent `
  --output-json work\evaluations\parser_repair_agent_synthetic.json `
  --output-md work\evaluations\parser_repair_agent_synthetic.md
```

Score:

| Metric | Score |
| --- | ---: |
| mean score | 100.0 |
| mean coverage score | 100.0 |
| mean mitigation score | 100.0 |
| mean guidance score | 100.0 |

Prompt handoff smoke test:

```powershell
.\.venv\Scripts\python.exe scripts\render_prompts.py `
  --paper-id paper9 `
  --parsed-dir work\paper9\parsed `
  --reviews-dir work\paper9\reviews `
  --schema-path schemas\reviewer_output.schema.json `
  --output-dir work\experiments\parser-repair-agent\prompt-render-smoke `
  --reviewers-config work\paper9\selection\selected_reviewers.json `
  --parser-repair-notes work\paper9\repair\parser_repair_notes.md
```

The repair overlay was appended to substantive reviewer prompts and not appended to `parser_quality_audit.txt`.

## Interpretation

The added agent is useful as a repair overlay and evidence-routing step. It can translate parser-quality findings into clear downstream instructions:

- which fallback artifacts to trust
- which structured artifacts should not be primary evidence
- which parser defects remain unresolved
- which deterministic fixes are still required

It should not be described as fully fixing parser issues. In the successful synthetic test, the agent did not reconstruct the broken table or rewrite page text; it mitigated risk by telling reviewers to prefer raw-page fallback evidence and avoid misleading structured artifacts.

## Recommendation

Keep the parser repair planner as an opt-in experimental step, not as the default reviewer workflow.

Use it when:

- parser-quality preflight reports medium-severity parser artifacts
- usable fallback artifacts exist
- the user explicitly approves sending the relevant parsed artifacts to the LLM service, or the run uses synthetic/redacted/local artifacts

Do not rely on it when:

- OCR, table reconstruction, crop regeneration, or page-order repair is required
- no trustworthy fallback artifact exists
- private parsed artifacts cannot be exported

Before promoting it to default, run an approved private-artifact experiment on known weak cases such as paper1, paper2, and paper9, then compare downstream reviewer outputs with and without the repair overlay.
