Review this repo as a Codex workflow, not just as a codebase.

Context:
- This repo is a local Windows/PowerShell project for a multi-agent academic paper reviewer.
- The manual pipeline has been proven for `inputs/paper1.pdf`.
- `scripts/review_paper.py` is the automated wrapper around that proven flow and has passed end-to-end smoke tests on both `paper1` and `paper2`.
- The proven flow is:
  1. preprocess PDF
  2. render run-specific prompts
  3. run configured reviewer agents
  4. validate reviewer JSON
  5. normalize/deduplicate reviewer outputs
  6. build editor input
  7. run the editor
  8. write the final markdown report
  9. smoke-check the final report
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
   - whether `scripts/review_paper.py` faithfully preserves the proven manual sequence
   - stale-output and partial-rerun risks
   - reviewer prompt quality
   - schema/output stability
   - preprocessing/parser weaknesses
   - normalization/deduplication behavior
   - editor traceability with `CANON-###` and source finding IDs
   - whether README, pipeline status, prompts, and scripts remain aligned
4. Do not change files yet.
5. Return a short review with:
   - what looks solid
   - what should be improved before heavier wrapper use
   - what should be simplified
   - whether the project is ready to rely on the single-script wrapper as the default path

Be concrete, critical, and repo-specific.
