# Pipeline Status

This file records what has been built, what has been tested, and what remains before the reviewer workflow should be wrapped in one script.

## Implemented

- Repo guidance exists in `AGENTS.md`.
- Project Codex defaults exist in `.codex/config.toml`.
- Custom agents exist under `.codex/agents/`.
- The repo-scoped paper-reviewer skill exists under `.agents/skills/paper-reviewer/`.
- PDF preprocessing is implemented in `scripts/preprocess_pdf.py`.
- Reviewer JSON validation is implemented in `scripts/validate_review_json.py`.
- Reusable reviewer/editor prompt templates live under `prompts/templates/`.
- Run-specific prompt rendering is implemented in `scripts/render_prompts.py`.
- Reviewer normalization/deduplication is implemented in `scripts/normalize_review_outputs.py`.
- Editor input assembly is implemented in `scripts/build_editor_input.py`.
- Final report smoke checking is implemented in `scripts/check_final_report.py`.

## Smoke-Tested So Far

- `inputs/paper1.pdf` can be preprocessed into `work/paper1/parsed/`.
- The five reviewer agents have previously produced schema-valid JSON files for `paper1`.
- The parser has been tightened so Figure 3 is restored through raw-caption fallback.
- Cross-reference extraction now suppresses known false positives such as `Appendix p`, `Appendix s`, `table s`, and `figure u`.
- Page-label inference improved label coverage for `paper1` from empty labels to labels on most pages.

## Known Weaknesses

- Existing reviewer outputs under `work/paper1/reviews/` were produced before the latest parser changes and should be regenerated before trusting a new final report.
- Page 41 still has scrambled normalized sorted text, even though Figure 3 is recovered from raw text.
- Raw-caption fallback crops are conservative because exact caption/body coordinates are not available from raw text.
- Normalization uses deterministic heuristics; it should be inspected after the next reviewer rerun.
- The editor previously wrote a status note instead of a substantive report. `check_final_report.py` now exists to catch that failure mode.

## Current Best Manual Pipeline

```powershell
python scripts\preprocess_pdf.py --pdf inputs\paper1.pdf

python scripts\render_prompts.py `
  --paper-id paper1 `
  --parsed-dir work\paper1\parsed `
  --reviews-dir work\paper1\reviews `
  --schema-path schemas\reviewer_output.schema.json `
  --output-dir work\paper1\prompts

# Run the five reviewer prompts with codex exec and --output-last-message.
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

## Remaining Before Wrapper

- Rerun all five reviewers against the updated parsed artifacts.
- Validate all five regenerated reviewer JSON outputs.
- Inspect `normalized_bundle.json` to confirm grouping and issue-class assignment are useful.
- Rerun the editor from `editor_input.md`.
- Confirm `outputs/paper1/report.md` passes `check_final_report.py` and is substantively useful.
- Only then add `scripts/review_paper.py` to orchestrate the full pipeline.

## Wrapper Readiness

The repo is close, but not ready for a single-script wrapper yet. The deterministic middle layer now exists, but it still needs one full manual proof run after the parser and prompt-template changes.
