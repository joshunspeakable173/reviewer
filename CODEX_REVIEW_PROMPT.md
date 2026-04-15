Review this repo as a Codex workflow, not just as a codebase.

Context:
- This repo is a local Windows/PowerShell project for a multi-agent academic paper reviewer.
- The intended flow is:
  1. preprocess PDF
  2. run five reviewer agents
  3. validate reviewer JSON
  4. normalize/deduplicate reviewer outputs
  5. run the editor
  6. write the final markdown report
- The repo uses:
  - AGENTS.md for repo-wide guidance
  - .codex/config.toml for project defaults
  - .codex/agents/*.toml for custom reviewer agents
  - .agents/skills/paper-reviewer/SKILL.md for the workflow skill
  - prompts/*.txt for run-specific prompt files
  - schemas/*.json for structured-output contracts
  - scripts/*.py for deterministic pipeline steps

Your task:
1. Check whether the repo actually matches that intended design.
2. Identify anything important that is missing, inconsistent, redundant, or brittle.
3. Focus especially on:
   - reviewer prompt quality
   - schema/output stability
   - preprocessing/parser weaknesses
   - whether normalization/deduplication is implemented cleanly
   - whether the editor stage is using the right inputs and producing the real final report
   - whether the repo structure still matches the mental model in README.md
4. Do not change files yet.
5. Return a short review with:
   - what looks solid
   - what should be improved next
   - what should be simplified
   - whether the project is ready for a single-script wrapper

Be concrete, critical, and repo-specific.
