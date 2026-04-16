# Pipeline Status

This file records what has been built, what has been tested, and what remains for the automated reviewer workflow.

## Implemented

- Repo guidance exists in `AGENTS.md`.
- Project Codex defaults exist in `.codex/config.toml`.
- Custom agents exist under `.codex/agents/`.
- The enabled reviewer roster is configured in `config/reviewers.json`.
- The repo-scoped paper-reviewer skill exists under `.agents/skills/paper-reviewer/`.
- PDF preprocessing is implemented in `scripts/preprocess_pdf.py`.
- Reviewer JSON validation is implemented in `scripts/validate_review_json.py`.
- Reviewer validation now includes semantic contract checks for stable IDs, issue type, confidence, location precision, source objects, cannot-verify reasons, and numerical recomputation notes.
- Reusable reviewer/editor prompt templates live under `prompts/templates/`.
- Run-specific prompt rendering is implemented in `scripts/render_prompts.py`.
- Reviewer normalization/deduplication is implemented in `scripts/normalize_review_outputs.py`.
- Editor input assembly is implemented in `scripts/build_editor_input.py`.
- Final report smoke checking is implemented in `scripts/check_final_report.py`.
- Full pipeline orchestration is implemented in `scripts/review_paper.py`.

## Smoke-Tested So Far

- `inputs/paper1.pdf` can be preprocessed into `work/paper1/parsed/`.
- The parser has been tightened so Figure 3 is restored through raw-caption fallback.
- Cross-reference extraction now suppresses known false positives such as `Appendix p`, `Appendix s`, `table s`, and `figure u`.
- Page-label inference improved label coverage for `paper1` from empty labels to labels on most pages.
- The default configured reviewer agents have produced regenerated schema-valid JSON files for `paper1`.
- `scripts/normalize_review_outputs.py` built `work/paper1/editor/normalized_bundle.json` from the regenerated reviewer outputs.
- `scripts/build_editor_input.py` built `work/paper1/editor/editor_input.md` from the normalized bundle and original reviewer JSON files.
- The editor generated `outputs/paper1/report.md`.
- `scripts/check_final_report.py --input outputs/paper1/report.md` passed after the editor report included canonical/source finding traceability.
- `scripts/review_paper.py --pdf inputs/paper1.pdf --paper-id paper1 --keep-going` passed end to end with parallel reviewer execution.
- `scripts/review_paper.py` also passed end to end on `paper2`: all default configured reviewer JSON files validated, `work/paper2/editor/normalized_bundle.json` and `work/paper2/editor/editor_input.md` were produced, the editor generated `outputs/paper2/report.md`, and `scripts/check_final_report.py --input outputs/paper2/report.md` passed.

## Known Weaknesses

- Page 41 still has scrambled normalized sorted text, even though Figure 3 is recovered from raw text.
- Raw-caption fallback crops are conservative because exact caption/body coordinates are not available from raw text.
- Normalization uses deterministic heuristics and should be revisited as more papers are tested.
- The first successful-looking editor report omitted canonical/source finding IDs and failed the final report check. The editor prompt template now requires traceability lines.
- Generated artifacts under `work/` and `outputs/` are ignored by Git, so successful proof-run outputs must be regenerated or preserved outside Git if needed.

## Current Best Workflow

The automated fresh-run entry point is:

```powershell
python scripts\review_paper.py --pdf inputs\paper2.pdf
```

It preprocesses, renders prompts, runs the configured reviewers in parallel, validates reviewer JSON, normalizes, builds editor input, runs the editor, and smoke-checks the final report.

```powershell
python scripts\preprocess_pdf.py --pdf inputs\paper1.pdf

python scripts\render_prompts.py `
  --paper-id paper1 `
  --parsed-dir work\paper1\parsed `
  --reviews-dir work\paper1\reviews `
  --schema-path schemas\reviewer_output.schema.json `
  --output-dir work\paper1\prompts

# Run the configured reviewer prompts with codex exec and --output-last-message.
# Validate each reviewer JSON.

python scripts\normalize_review_outputs.py `
  --paper-id paper1 `
  --reviews-dir work\paper1\reviews `
  --output work\paper1\editor\normalized_bundle.json

python scripts\build_editor_input.py `
  --paper-id paper1 `
  --editor-prompt work\paper1\prompts\editor_report.txt `
  --bundle work\paper1\editor\normalized_bundle.json `
  --reviews-dir work\paper1\reviews `
  --output work\paper1\editor\editor_input.md

Get-Content work\paper1\editor\editor_input.md -Raw |
  codex exec --output-last-message outputs\paper1\report.md -

python scripts\check_final_report.py --input outputs\paper1\report.md
```

## Remaining Automation Work

- Add optional selective rerun/resume flags only after the fresh wrapper is reliable on more than one paper.
- Consider a sequential-reviewer fallback flag if parallel Codex execution is hard to debug on some runs.
- Continue improving parser quality, especially page 41 text ordering and conservative figure/table crop boundaries.

## Wrapper Readiness

The wrapper exists, preserves the proven workflow, and has passed on both `paper1` and `paper2`. Parser quality still needs incremental improvement, and selective rerun/resume support would make repeated runs cheaper, but the wrapper is now the default entry point for fresh runs.
