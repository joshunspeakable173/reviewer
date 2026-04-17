Review this repo as a Codex workflow, not just as a codebase.

Context:
- This repo is a local Windows/PowerShell project for a multi-agent academic paper reviewer.
- `scripts/review_paper.py` is the automated wrapper around the proven review flow and has passed end-to-end smoke tests across multiple papers.
- The current default wrapper mode uses parser-quality preflight and dynamic reviewer selection. Mandatory reviewers always run; optional conceptual reviewers are selected based on paper type.
- Editor-only refresh has also been tested from existing reviewer JSON and `normalized_bundle.json`.
- The proven flow is:
  1. preprocess PDF
  2. render run-specific prompts
  3. run parser-quality preflight
  4. dynamically select optional reviewers, unless static mode is requested
  5. write `work/<paper_id>/selection/selected_reviewers.json`
  6. rerender prompts for the selected reviewer roster
  7. run mandatory and selected reviewer agents
  8. validate reviewer JSON
  9. normalize/deduplicate reviewer outputs
  10. build editor input with deterministic editor guidance
  11. run the editor
  12. write the final markdown report
  13. smoke-check the final report
- The repo uses:
  - `AGENTS.md` for repo-wide guidance
  - `.codex/config.toml` for project defaults
  - `.codex/agents/*.toml` for custom reviewer/editor definitions
  - `config/reviewers.json` for the enabled reviewer roster
  - `.agents/skills/paper-reviewer/SKILL.md` for the workflow skill
  - `prompts/templates/*.txt` for reusable prompt templates
  - `schemas/*.json` for structured-output contracts
  - `scripts/*.py` for deterministic pipeline steps

Your task:
1. Check whether the repo still matches the intended workflow design.
2. Identify anything important that is missing, inconsistent, redundant, or brittle.
3. Focus especially on:
   - readiness for broader `scripts/review_paper.py` use
   - whether reviewer additions can be made through `config/reviewers.json`
   - whether `scripts/review_paper.py` faithfully preserves the proven sequence
   - whether dynamic reviewer selection and selected reviewer config files are handled clearly
   - whether editor-only refresh is safe and documented
   - stale-output and partial-rerun risks
   - reviewer prompt quality
   - schema/output stability
   - preprocessing/parser weaknesses
   - normalization/deduplication behavior
   - editor traceability with `CANON-###` and source finding IDs
   - whether the final report keeps traceability in the appendix, not repeated body footers
   - whether the editor brief caps highest-priority synthesis and routes secondary findings cleanly
   - whether `scripts/check_final_report.py` is being treated as the smoke check it is, not as a full semantic traceability audit
   - whether README, pipeline status, prompts, and scripts remain aligned
4. Do not change files yet.
5. Return a short review with:
   - what looks solid
   - what should be improved before heavier wrapper use
   - what should be simplified
   - whether the project is ready to rely on the single-script wrapper as the default path

Be concrete, critical, and repo-specific.
