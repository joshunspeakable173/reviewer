# Pipeline Status

This file records what has been built, what has been tested, and what remains for the automated reviewer workflow.

## Implemented

- Repo guidance exists in `AGENTS.md`.
- Project Codex defaults exist in `.codex/config.toml`.
- Custom agents exist under `.codex/agents/`.
- The enabled reviewer roster is configured in `config/reviewers.json`.
- The parser-quality auditor runs as a preflight reviewer before substantive reviewers.
- The grammar auditor runs as a review-stage copyedit reviewer and is intended for an appendix table in the final report.
- The reviewer selector can classify a paper and choose from optional paper-type-specific reviewers while mandatory baseline reviewers always run.
- The repo-scoped paper-reviewer skill exists under `.agents/skills/paper-reviewer/`.
- PDF preprocessing is implemented in `scripts/preprocess_pdf.py`.
- Reviewer JSON validation is implemented in `scripts/validate_review_json.py`.
- Reviewer validation now includes semantic contract checks for stable IDs, issue type, confidence, location precision, source objects, cannot-verify reasons, and numerical recomputation notes.
- Reusable reviewer/editor prompt templates live under `prompts/templates/`.
- Run-specific prompt rendering is implemented in `scripts/render_prompts.py`.
- Dynamic reviewer selection writes `work/<paper_id>/selection/reviewer_selection.json` and `work/<paper_id>/selection/selected_reviewers.json`.
- Reviewer normalization/deduplication is implemented in `scripts/normalize_review_outputs.py`.
- Editor input assembly is implemented in `scripts/build_editor_input.py`, including a deterministic editor brief for concise reviewer disclosure, priority scoring with an eight-item maximum for synthesis candidates, additional-finding candidates, section routing, and traceability-map rows.
- Final report smoke checking is implemented in `scripts/check_final_report.py`.
- Final report smoke checking now requires the grammar appendix heading when normalized copyedit findings are present and the traceability appendix when a normalized bundle is supplied.
- Full pipeline orchestration is implemented in `scripts/review_paper.py`, including parser-quality preflight gating.
- Editor-only report refresh is supported manually by rerendering prompts, rebuilding editor input from an existing normalized bundle and reviewer JSON files, rerunning the editor, and smoke-checking the report.
- The wrapper now detects implausibly short editor outputs and recovers the complete report from the editor transcript when `codex exec --output-last-message` captures a short acknowledgement after delegated editor output.
- The editor prompt now asks for 3 to 8 high-confidence top-level synthesis issues when supported by the evidence, preserves concrete study names in literature critiques, and adds an external-sources appendix when external evidence is cited in the report body.
- The external-sources appendix is guarded against hallucinated metadata: it should use only details present in reviewer evidence and leave missing bibliographic details blank or marked as not provided.

## Smoke-Tested So Far

- Preprocessing has been exercised on multiple PDFs and produces page text, page images, inventories, and manifest artifacts under `work/<paper_id>/parsed/`.
- The parser has been tightened with raw-caption fallback, false-positive cross-reference suppression, and improved page-label inference.
- The configured reviewer agents have produced schema-valid JSON outputs across multiple paper types.
- `scripts/normalize_review_outputs.py` has built normalized editor bundles from regenerated reviewer outputs.
- `scripts/build_editor_input.py` has built editor input from normalized bundles and original reviewer JSON files.
- The editor has generated final markdown reports that pass `scripts/check_final_report.py --bundle work/<paper_id>/editor/normalized_bundle.json`.
- `scripts/review_paper.py` has passed end-to-end smoke runs with parallel reviewer execution, parser-quality preflight, dynamic reviewer selection, selected optional reviewers, reviewer validation, normalization, editor synthesis, and final report checking.
- The selector has been exercised on different paper types and can select relevant optional reviewers while skipping irrelevant ones.
- Parser-quality caveats are now reportable rather than automatically blocking unless they are high-severity, high-confidence parser artifacts.
- Editor-only report refresh has been smoke-tested from existing reviewer JSON and normalized bundles.
- The wrapper has recovered complete editor reports from the editor transcript when `codex exec --output-last-message` captured only a short acknowledgement.

## Known Weaknesses

- Some pages in complex PDFs can still have scrambled normalized sorted text, even when raw-caption fallback recovers nearby figures or tables.
- Raw-caption fallback crops are conservative because exact caption/body coordinates are not available from raw text.
- Normalization uses deterministic heuristics and should be revisited as more papers are tested.
- The first successful-looking editor report omitted canonical/source finding IDs and failed the final report check. The editor prompt template now requires a traceability appendix rather than repeated body footers.
- The final report checker verifies report shape and traceability identifiers, but it does not independently verify external references or registry links.
- Generated artifacts under `work/` and `outputs/` are ignored by Git, so successful proof-run outputs must be regenerated or preserved outside Git if needed.

## Current Best Workflow

The automated fresh-run entry point is:

```powershell
$paperId = "your-paper-id"
python scripts\review_paper.py --pdf "inputs\${paperId}.pdf"
```

It preprocesses, renders prompts, runs the configured reviewers in parallel, validates reviewer JSON, normalizes, builds editor input, runs the editor, and smoke-checks the final report.
The parser-quality auditor runs first and blocks only on high-severity, high-confidence parser artifacts that would make downstream review unsafe.
Dynamic reviewer selection is the default. Use `--reviewer-selection static` to run all enabled review-stage reviewers.

```powershell
$paperId = "your-paper-id"
python scripts\preprocess_pdf.py --pdf "inputs\${paperId}.pdf"

python scripts\render_prompts.py `
  --paper-id $paperId `
  --parsed-dir "work\${paperId}\parsed" `
  --reviews-dir "work\${paperId}\reviews" `
  --schema-path schemas\reviewer_output.schema.json `
  --output-dir "work\${paperId}\prompts"

# In dynamic mode, rerender after reviewer selection with:
python scripts\render_prompts.py `
  --paper-id $paperId `
  --parsed-dir "work\${paperId}\parsed" `
  --reviews-dir "work\${paperId}\reviews" `
  --schema-path schemas\reviewer_output.schema.json `
  --output-dir "work\${paperId}\prompts" `
  --reviewers-config "work\${paperId}\selection\selected_reviewers.json"

# Run the configured reviewer prompts with codex exec and --output-last-message.
# Validate each reviewer JSON.

python scripts\normalize_review_outputs.py `
  --paper-id $paperId `
  --reviews-dir "work\${paperId}\reviews" `
  --output "work\${paperId}\editor\normalized_bundle.json" `
  --reviewers-config "work\${paperId}\selection\selected_reviewers.json"

python scripts\build_editor_input.py `
  --paper-id $paperId `
  --editor-prompt "work\${paperId}\prompts\editor_report.txt" `
  --bundle "work\${paperId}\editor\normalized_bundle.json" `
  --reviews-dir "work\${paperId}\reviews" `
  --output "work\${paperId}\editor\editor_input.md" `
  --reviewers-config "work\${paperId}\selection\selected_reviewers.json"

Get-Content "work\${paperId}\editor\editor_input.md" -Raw |
  codex exec --output-last-message "outputs\${paperId}\report.md" -

python scripts\check_final_report.py `
  --input "outputs\${paperId}\report.md" `
  --bundle "work\${paperId}\editor\normalized_bundle.json"
```

For editor-only refreshes, reuse existing `work/<paper_id>/reviews/`, `work/<paper_id>/editor/normalized_bundle.json`, and `work/<paper_id>/selection/selected_reviewers.json`; rerender prompts, rebuild editor input, rerun only the editor, then run the final report checker with `--bundle`.

## Remaining Automation Work

- Add optional selective rerun/resume flags now that editor-only refresh has a proven manual recipe.
- Consider a sequential-reviewer fallback flag if parallel Codex execution is hard to debug on some runs.
- Continue improving parser quality, especially complex page text ordering, conservative figure/table crop boundaries, and caption forms without colons.

## Wrapper Readiness

The wrapper exists, preserves the proven workflow, and has passed smoke testing across multiple papers. Parser quality still needs incremental improvement, and selective rerun/resume support would make repeated runs cheaper, but the wrapper is now the default entry point for fresh runs.
