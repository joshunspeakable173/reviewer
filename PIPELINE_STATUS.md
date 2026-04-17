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
- Editor input assembly is implemented in `scripts/build_editor_input.py`, including a deterministic editor brief for concise reviewer disclosure, priority scoring with a five-item synthesis cap, additional-finding candidates, section routing, and traceability-map rows.
- Final report smoke checking is implemented in `scripts/check_final_report.py`.
- Final report smoke checking now requires the grammar appendix heading when normalized copyedit findings are present and the traceability appendix when a normalized bundle is supplied.
- Full pipeline orchestration is implemented in `scripts/review_paper.py`, including parser-quality preflight gating.
- Editor-only report refresh is supported manually by rerendering prompts, rebuilding editor input from an existing normalized bundle and reviewer JSON files, rerunning the editor, and smoke-checking the report.
- The wrapper now detects implausibly short editor outputs and recovers the complete report from the editor transcript when `codex exec --output-last-message` captures a short acknowledgement after delegated editor output.

## Smoke-Tested So Far

- `inputs/paper1.pdf` can be preprocessed into `work/paper1/parsed/`.
- The parser has been tightened so Figure 3 is restored through raw-caption fallback.
- Cross-reference extraction now suppresses known false positives such as `Appendix p`, `Appendix s`, `table s`, and `figure u`.
- Page-label inference improved label coverage for `paper1` from empty labels to labels on most pages.
- The default configured reviewer agents have produced regenerated schema-valid JSON files for `paper1`.
- `scripts/normalize_review_outputs.py` built `work/paper1/editor/normalized_bundle.json` from the regenerated reviewer outputs.
- `scripts/build_editor_input.py` built `work/paper1/editor/editor_input.md` from the normalized bundle and original reviewer JSON files.
- The editor generated `outputs/paper1/report.md`.
- `scripts/check_final_report.py --input outputs/paper1/report.md --bundle work/paper1/editor/normalized_bundle.json` passed after the editor report included canonical/source finding traceability.
- `scripts/review_paper.py --pdf inputs/paper1.pdf --paper-id paper1 --keep-going` passed end to end with parallel reviewer execution.
- `scripts/review_paper.py` also passed end to end on `paper2`: all default configured reviewer JSON files validated, `work/paper2/editor/normalized_bundle.json` and `work/paper2/editor/editor_input.md` were produced, the editor generated `outputs/paper2/report.md`, and `scripts/check_final_report.py --input outputs/paper2/report.md --bundle work/paper2/editor/normalized_bundle.json` passed.
- `scripts/review_paper.py --pdf inputs/paper4.pdf --paper-id paper4 --keep-going` passed end to end with parser-quality preflight, dynamic reviewer selection, selected optional reviewers, reviewer validation, normalization, editor synthesis, and final report checking.
- The `paper4` selector classified the paper as `empirical_causal`, selected the relevant empirical/causal optional reviewers, and skipped model-equation and data-availability reviewers.
- `paper4` exposed parser-quality caveats that are now reportable rather than blocking: missing structured table/figure inventories despite visible Table 1 and Fig. 1, sparse citation inventory, noisy section inventory, and reference-list contamination.
- Editor-only report refresh on `paper4` passed after the latest editor prompt changes: the final report has five highest-priority findings, an `Additional Findings` table, a cannot-verify table, a grammar appendix, a traceability appendix, and passes `scripts/check_final_report.py --input outputs/paper4/report.md --bundle work/paper4/editor/normalized_bundle.json`.
- `paper3` exposed an editor-output capture failure where `--output-last-message` wrote `Received.` while the full report was present in the editor transcript. The recovery hook rebuilt `outputs/paper3/report.md` from `work/paper3/logs/editor.stderr.log`, and `scripts/check_final_report.py --input outputs/paper3/report.md --bundle work/paper3/editor/normalized_bundle.json` passed.

## Known Weaknesses

- Page 41 still has scrambled normalized sorted text, even though Figure 3 is recovered from raw text.
- Raw-caption fallback crops are conservative because exact caption/body coordinates are not available from raw text.
- Normalization uses deterministic heuristics and should be revisited as more papers are tested.
- The first successful-looking editor report omitted canonical/source finding IDs and failed the final report check. The editor prompt template now requires a traceability appendix rather than repeated body footers.
- Generated artifacts under `work/` and `outputs/` are ignored by Git, so successful proof-run outputs must be regenerated or preserved outside Git if needed.

## Current Best Workflow

The automated fresh-run entry point is:

```powershell
python scripts\review_paper.py --pdf inputs\paper4.pdf
```

It preprocesses, renders prompts, runs the configured reviewers in parallel, validates reviewer JSON, normalizes, builds editor input, runs the editor, and smoke-checks the final report.
The parser-quality auditor runs first and blocks only on high-severity, high-confidence parser artifacts that would make downstream review unsafe.
Dynamic reviewer selection is the default. Use `--reviewer-selection static` to run all enabled review-stage reviewers.

```powershell
python scripts\preprocess_pdf.py --pdf inputs\paper1.pdf

python scripts\render_prompts.py `
  --paper-id paper1 `
  --parsed-dir work\paper1\parsed `
  --reviews-dir work\paper1\reviews `
  --schema-path schemas\reviewer_output.schema.json `
  --output-dir work\paper1\prompts

# In dynamic mode, rerender after reviewer selection with:
python scripts\render_prompts.py `
  --paper-id paper1 `
  --parsed-dir work\paper1\parsed `
  --reviews-dir work\paper1\reviews `
  --schema-path schemas\reviewer_output.schema.json `
  --output-dir work\paper1\prompts `
  --reviewers-config work\paper1\selection\selected_reviewers.json

# Run the configured reviewer prompts with codex exec and --output-last-message.
# Validate each reviewer JSON.

python scripts\normalize_review_outputs.py `
  --paper-id paper1 `
  --reviews-dir work\paper1\reviews `
  --output work\paper1\editor\normalized_bundle.json `
  --reviewers-config work\paper1\selection\selected_reviewers.json

python scripts\build_editor_input.py `
  --paper-id paper1 `
  --editor-prompt work\paper1\prompts\editor_report.txt `
  --bundle work\paper1\editor\normalized_bundle.json `
  --reviews-dir work\paper1\reviews `
  --output work\paper1\editor\editor_input.md `
  --reviewers-config work\paper1\selection\selected_reviewers.json

Get-Content work\paper1\editor\editor_input.md -Raw |
  codex exec --output-last-message outputs\paper1\report.md -

python scripts\check_final_report.py `
  --input outputs\paper1\report.md `
  --bundle work\paper1\editor\normalized_bundle.json
```

For editor-only refreshes, reuse existing `work/<paper_id>/reviews/`, `work/<paper_id>/editor/normalized_bundle.json`, and `work/<paper_id>/selection/selected_reviewers.json`; rerender prompts, rebuild editor input, rerun only the editor, then run the final report checker with `--bundle`.

## Remaining Automation Work

- Add optional selective rerun/resume flags now that editor-only refresh has a proven manual recipe.
- Consider a sequential-reviewer fallback flag if parallel Codex execution is hard to debug on some runs.
- Continue improving parser quality, especially page 41 text ordering, conservative figure/table crop boundaries, and caption forms without colons.

## Wrapper Readiness

The wrapper exists, preserves the proven workflow, and has passed on `paper1`, `paper2`, and `paper4`. Parser quality still needs incremental improvement, and selective rerun/resume support would make repeated runs cheaper, but the wrapper is now the default entry point for fresh runs.
