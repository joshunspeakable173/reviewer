# Project Documentation

This document explains the multi-agent academic paper reviewer as an external-facing project artifact. It describes what the system does, how a paper moves through the workflow, why the major design choices were made, and what prompt templates are used by the reviewer agents.

This document is not model guidance. Operational instructions for Codex live in the repository guidance, skills, prompt templates, and scripts.

## 1. Project Overview

The project builds a reproducible workflow for reviewing academic economics papers with a combination of deterministic preprocessing scripts and specialized LLM reviewer agents.

The system takes a source PDF, extracts structured artifacts, runs a configurable set of reviewer agents, validates their structured JSON outputs, normalizes overlapping findings, and asks an editor agent to synthesize a final Markdown report. The intended output is a readable review report that prioritizes substantive issues while preserving traceability back to the underlying reviewer findings.

The workflow is local and file-based. Source PDFs usually live under `inputs/`, intermediate artifacts under `work/<paper_id>/`, and final reports under `outputs/<paper_id>/`.

## 2. Design Goals

The project is designed around five goals:

1. **Reproducibility.** Each stage writes inspectable files so a run can be audited, rerun, or debugged.
2. **Separation of concerns.** Deterministic scripts handle file movement, parsing, validation, normalization, and orchestration. LLM agents handle judgment-heavy review tasks.
3. **Structured reviewer outputs.** Reviewer agents return JSON that must conform to a schema and semantic validation checks.
4. **Modular review.** Each reviewer owns one conceptual area, such as references, numerical consistency, identification, or grammar.
5. **Readable synthesis.** The final editor report is synthesis-first, with traceability identifiers collected in an appendix instead of cluttering the main prose.

## 3. End-to-End Workflow

The normal fresh-run workflow is:

1. Place the source PDF in `inputs/`.
2. Preprocess the PDF into structured artifacts under `work/<paper_id>/parsed/`.
3. Render run-specific prompts under `work/<paper_id>/prompts/`.
4. Run the parser-quality preflight auditor.
5. Validate preflight JSON and stop only on high-confidence parser defects that would make downstream review unsafe.
6. Classify the paper and select optional reviewers, unless static mode is requested.
7. Write the selected reviewer roster under `work/<paper_id>/selection/selected_reviewers.json`.
8. Rerender prompts for the active reviewer roster.
9. Run mandatory baseline reviewers and selected optional reviewers on the parsed artifacts.
10. Validate reviewer JSON outputs under `work/<paper_id>/reviews/`.
11. Normalize and deduplicate reviewer outputs into `work/<paper_id>/editor/normalized_bundle.json`.
12. Build editor input at `work/<paper_id>/editor/editor_input.md`.
13. Run the editor to write `outputs/<paper_id>/report.md`.
14. Smoke-check the final report.

The wrapper script `scripts/review_paper.py` orchestrates this sequence for fresh runs.

## 4. Why Preprocessing Comes First

PDF extraction quality strongly shapes all downstream review quality. Academic PDFs can contain multi-column text, appendix labels, figures, tables, footnotes, scanned pages, and embedded images. If these artifacts are missing or scrambled, reviewer agents may make unreliable judgments.

Preprocessing therefore creates page-level and object-level artifacts before any reviewer reads the paper. The parsed output includes normalized page text, raw page text, page images, word/block metadata, table inventories, figure inventories, references, citations, cross-reference candidates, and a manifest.

The parser-quality auditor runs before substantive review so the pipeline can distinguish between manuscript problems and extraction problems. High-confidence blocking parser defects stop the run. Less severe parser caveats can be carried into the final report.

## 5. Why Reviewer Agents Return JSON

Each reviewer returns one structured JSON object. The schema requires fields such as stable finding IDs, issue type, severity, confidence, location precision, claim text, assessment, source objects, and suggested fixes.

This design makes reviewer outputs:

- easier to validate automatically,
- easier to rerun independently,
- easier to deduplicate across agents,
- easier to trace in the editor report,
- less dependent on free-form prose formatting.

The validation layer checks both JSON schema conformance and semantic expectations, such as stable ID prefixes, exact-location consistency, source-object requirements for verifiable findings, cannot-verify reasons, and numeric-check fields for numerical findings.

## 6. Reviewer Selection

Mandatory reviewers always run when enabled. Optional reviewers are selected dynamically based on the paper type and available parsed evidence.

The selector classifies the paper as one of several broad types, such as empirical causal, empirical descriptive, theory, methods, literature review, mixed, or unknown. It then selects optional reviewers whose audits are likely to catch high-impact errors for that paper.

Dynamic selection helps keep the report focused. For example, a theory-heavy paper may need equation and model review, while an empirical causal paper may need identification, robustness, sample construction, and numerical review. Static mode is still available when all enabled review-stage agents should run.

## 7. Normalization and Deduplication

Multiple reviewers can identify overlapping problems. The normalizer groups similar findings into canonical findings with IDs such as `CANON-001`.

Normalization separates issue classes such as:

- manuscript issues,
- parser artifacts,
- reference integrity issues,
- bibliography maintenance,
- copyediting issues,
- cannot-verify items.

This step gives the editor a consolidated evidence bundle instead of a flat pile of reviewer outputs. It also lets the final report include a traceability appendix that maps canonical findings back to source reviewer IDs.

## 8. Editor Synthesis

The editor receives three inputs:

1. a deterministic editor brief,
2. the normalized editor bundle,
3. the original configured reviewer JSON outputs.

The editor brief is generated by `scripts/build_editor_input.py`. It gives the editor a deterministic map for prioritization, section routing, optional-reviewer disclosure, additional findings, cannot-verify items, grammar appendix items, and traceability rows.

The final report is intentionally not organized agent by agent. It starts with cross-agent synthesis, limits the highest-priority section to five findings, moves lower-priority items into additional findings or domain-specific sections, and keeps all canonical/source finding IDs in a traceability appendix.

## 9. Report Checking

The final report checker is a smoke check, not a full semantic review. It rejects outputs that are too short, look like run-status notes, omit required report headings, omit grammar or traceability appendices when needed, or fail to mention canonical/source finding identifiers from the normalized bundle.

This catches common output failures while keeping substantive judgment in the reviewer and editor stages.

## 10. Main Artifacts

| Location | Purpose |
| --- | --- |
| `inputs/` | Source PDFs. |
| `work/<paper_id>/parsed/` | Preprocessed page text, page images, extracted objects, inventories, and manifest. |
| `work/<paper_id>/prompts/` | Run-specific rendered prompts. |
| `work/<paper_id>/selection/` | Reviewer selector decision and selected reviewer roster. |
| `work/<paper_id>/reviews/` | Structured reviewer JSON outputs. |
| `work/<paper_id>/editor/normalized_bundle.json` | Normalized and deduplicated reviewer findings. |
| `work/<paper_id>/editor/editor_input.md` | Prompt plus structured evidence for the editor. |
| `outputs/<paper_id>/report.md` | Final human-readable review report. |

## 11. Agent Roster

| Agent | Stage | Selection | Role | Search | Output | Purpose |
| --- | --- | --- | --- | --- | --- | --- |
| `parser_quality_auditor` | preflight | mandatory | manuscript | no | `parser_quality_auditor.json` | Checks whether parsed artifacts are reliable enough for downstream review. |
| `crossref_auditor` | review | mandatory | crossref | no | `crossref_auditor.json` | Checks internal references, numbering, and appendix-label consistency. |
| `reference_auditor` | review | mandatory | reference | yes | `reference_auditor.json` | Checks citation/reference integrity and bibliography-maintenance issues. |
| `grammar_auditor` | review | mandatory | copyedit | no | `grammar_auditor.json` | Finds concise grammar and copyediting issues for a report appendix. |
| `numerical_auditor` | review | optional | manuscript | no | `numerical_auditor.json` | Checks numeric claims, signs, units, rounding, and table/text consistency. |
| `claim_evidence_auditor` | review | optional | manuscript | no | `claim_evidence_auditor.json` | Checks whether major textual claims are supported by displayed evidence. |
| `literature_auditor` | review | optional | manuscript | yes | `literature_auditor.json` | Checks novelty claims, prior-work characterization, and literature positioning. |
| `identification_auditor` | review | optional | manuscript | no | `identification_auditor.json` | Checks causal language, identifying assumptions, and threats to validity. |
| `robustness_auditor` | review | optional | manuscript | no | `robustness_auditor.json` | Checks robustness and sensitivity claims against supporting evidence. |
| `sample_construction_auditor` | review | optional | manuscript | no | `sample_construction_auditor.json` | Checks sample restrictions, attrition, denominators, and variable construction. |
| `abstract_conclusion_consistency_auditor` | review | optional | manuscript | no | `abstract_conclusion_consistency_auditor.json` | Checks abstract, introduction, and conclusion claims against the body. |
| `limitations_external_validity_auditor` | review | optional | manuscript | no | `limitations_external_validity_auditor.json` | Checks limitations, scope, policy claims, and external validity. |
| `model_equation_auditor` | review | optional | manuscript | no | `model_equation_auditor.json` | Checks model notation, equations, assumptions, and text-equation consistency. |
| `data_availability_replication_auditor` | review | optional | manuscript | yes | `data_availability_replication_auditor.json` | Checks preregistration, pre-analysis-plan documentation, planned-vs-reported analyses, and scoped reproducibility claims. |

## 12. Current Limitations

The system is designed to be inspectable, not infallible.

- PDF parsing can still fail on complex page layouts, scanned pages, unusual captions, or difficult table structures.
- Reviewer findings depend on the quality of parsed artifacts.
- Dynamic reviewer selection can choose only among configured optional reviewers.
- Normalization uses deterministic heuristics and may need adjustment as the reviewer roster evolves.
- The final report checker verifies structure and traceability coverage, but it does not prove that the editor synthesized every issue perfectly.
- Large editor inputs can increase latency and make editor output capture more fragile.

## Appendix A. Reviewer Output Contract

Reviewer outputs must conform to `schemas/reviewer_output.schema.json`. Each output includes:

- `reviewer`: the reviewer name,
- `paper_id`: the run identifier,
- `run_status`: `ok`, `partial`, `cannot_verify`, or `failed`,
- `summary`: a short reviewer summary,
- `findings`: a list of structured findings,
- `notes`: optional process-neutral notes.

Each finding includes:

- stable `id` using the configured prefix,
- `category`,
- `issue_type`,
- `severity`,
- `confidence`,
- precise or explicitly missing `location`,
- `claim_text`,
- `assessment`,
- `cannot_verify_reason` when needed,
- `evidence_summary`,
- `source_objects`,
- `claim_evidence_links`,
- `numeric_check` when relevant,
- `suggested_fix`.

## Appendix B. Final Report Structure

The editor is guided toward this report shape:

```markdown
# Multi-Agent Paper Review Report

## Executive Summary

## Review Configuration

## Highest-Priority Cross-Agent Findings

## Suggested Revision Priorities

## Additional Findings

## Literature Positioning and Novelty

## Reference Integrity and Bibliography Maintenance

## Parser and Preprocessing Caveats

## Items Marked Cannot Verify

## Appendix: Grammar and Copyediting Issues

## Appendix: Traceability Map
```

The traceability appendix maps each canonical finding to its source reviewer findings:

```markdown
| Report section | Finding | Canonical ID | Source finding IDs |
| --- | --- | --- | --- |
| Highest-Priority Cross-Agent Findings | Example issue summary | CANON-001 | claim_evidence_auditor:CEA-001 |
```

## Appendix C. Prompt Templates

This appendix reproduces the current prompt templates used by the workflow. The templates are rendered per run with paper-specific paths and schema locations.

### C.1 Preflight Prompt

#### `parser_quality_audit.txt`

````text
Use the custom agent named parser_quality_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit parser quality only:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "parser_quality_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted PARSER-001, PARSER-002, ...

Focus on parser defects that could poison downstream reviewers:
- missing page artifacts or likely scanned pages
- scrambled reading order in normalized page text
- suspicious OCR or text-extraction artifacts
- broken table extraction
- missing or detached table/figure captions
- figure/table extraction failures
- inventory contradictions, such as visible captions or embedded images missing from structured inventories

Use these severity rules:
- high: downstream reviewers cannot safely operate on a major evidence class or substantial page range, and no usable page-text/image fallback exists
- medium: reviewers can proceed, but one evidence class or important object may be incomplete or misleading; use this for missing table/figure inventories when page text or page images still provide usable fallback evidence
- low: localized parser defect or cleanup issue

For every finding:
- Set issue_type to parser_artifact or cannot_verify.
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Include source_objects for manifest, page text, raw text, word/block files, inventories, page images, table crops, figure crops, or embedded-image inventory entries used as evidence.
- Use cannot_verify only when the parsed artifacts do not allow a check, and then include cannot_verify_reason.
- Do not flag terminal display mojibake as an encoding defect. Only report encoding/OCR corruption when the file contents themselves contain replacement characters, repeated question-mark substitutions, or verified corrupted text. Normal Unicode such as en dashes, curly quotes, ligatures, accented names, or copyright symbols is not a parser defect by itself.
- Do not judge manuscript correctness, numerical correctness, reference correctness, or literature positioning.
- Do not include process notes about skill availability, tooling, or session state.
````

### C.2 Reviewer Selection Prompt

#### `reviewer_selection.txt`

````text
Use the custom agent named reviewer_selector.

Use the repo guidance and the paper-reviewer workflow.

Classify paper_id "{paper_id}" using parsed artifacts under:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{selection_schema_path}`

Select optional reviewers only from this catalog:

```json
{optional_reviewer_catalog}
```

Classification guidance:
- empirical_causal: empirical paper making causal, treatment-effect, identification, quasi-experimental, experimental, or policy-effect claims.
- empirical_descriptive: empirical paper mainly describing patterns, correlations, measurement, forecasting, or facts without causal identification as the main contribution.
- theory: theoretical, conceptual, or model-driven paper with little or no empirical analysis.
- methods: paper introducing an estimator, measurement method, dataset construction method, or computational procedure.
- literature_review: survey or literature review paper.
- mixed: multiple major paper types are central.
- unknown: parsed artifacts are too limited to classify confidently.

Selection rules:
- Select numerical_auditor for papers with important numeric claims, tables, figures, or appendix estimates.
- Select claim_evidence_auditor for papers with substantive empirical, policy, or interpretive claims tied to displayed evidence.
- Select literature_auditor for papers with novelty, positioning, or prior-work characterization claims.
- Select identification_auditor for empirical_causal papers or mixed papers with causal/identification claims.
- Select robustness_auditor for empirical papers with robustness checks, alternative specifications, subsamples, placebo tests, or appendix validations.
- Select sample_construction_auditor for empirical papers with nontrivial sample restrictions, attrition, variable construction, exclusions, or denominators.
- Select abstract_conclusion_consistency_auditor when abstract/introduction/conclusion claims appear important enough to check against the body.
- Select limitations_external_validity_auditor for papers making policy, welfare, generalizability, or scope claims.
- Select model_equation_auditor for theory, structural, model-heavy, equation-heavy, or methods papers.
- Select data_availability_replication_auditor for experiments, RCTs, preregistered or pre-planned empirical designs, explicit pre-analysis-plan claims, registry links or IDs, or concrete replication/reproducibility claims. Treat mentions of platforms such as AEA RCT Registry, OSF Registries, AsPredicted, and similar services as strong selection cues.

When unsure, prefer selecting a reviewer if its audit would catch high-impact errors and the parsed artifacts contain enough material to inspect.
````

### C.3 Reviewer Prompts

#### `crossref_audit.txt`

````text
Use the custom agent named crossref_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "crossref_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted CROSSREF-001, CROSSREF-002, ...

Focus on:
- whether internal references resolve
- numbering consistency
- appendix-label consistency

Classification rule:
- Set issue_type = "parser_artifact" for extraction/caption/OCR/false-positive issues.
- Set issue_type = "manuscript_issue" and category = "manuscript_crossref_issue" only for genuine broken or inconsistent paper references.
- Prioritize genuine manuscript_crossref_issue findings over parser_artifact findings.

For every finding:
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Include source_objects for referenced tables, figures, equations, sections, appendices, or parsed artifacts when available.
- Use cannot_verify only when the artifact does not allow a check, and then include cannot_verify_reason.

Do not include process notes about skill availability, tooling, or session state.
````

#### `literature_audit.txt`

````text
Use the custom agent named literature_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "literature_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted LIT-001, LIT-002, ...

Focus on:
- novelty claims
- literature positioning
- characterization of prior work
- conservative verification
- cannot_verify when verification is not possible

Ground critiques in concrete studies. A finding that says a novelty claim is overstated, that prior literature already covers an issue, or that the manuscript mischaracterizes a literature must cite specific studies or external sources in source_objects. If a broad novelty claim cannot be verified or contradicted with concrete studies, use cannot_verify rather than asserting lack of novelty.

For every finding:
- Set issue_type to manuscript_issue or cannot_verify.
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Include source_objects for the manuscript claim and any external or parsed sources used.
- Include claim_evidence_links when comparing a manuscript claim to prior work or external evidence.
- Use cannot_verify only when verification is not possible, and then include cannot_verify_reason.

Distinguish clearly between:
1. cannot_verify because proving novelty requires proving absence, and
2. overbroad positioning even without proving novelty.

Do not include process notes about skill availability, tooling, or session state.
````

#### `model_equation_audit.txt`

````text
Use the custom agent named model_equation_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit model, equation, notation, and text-equation consistency only:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "model_equation_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted MODEL-001, MODEL-002, ...

Focus on:
- notation definitions and reuse
- equation signs, variables, assumptions, constraints, and interpretation
- consistency between propositions, text, tables, figures, and equations
- model claims that do not follow from the displayed setup

For every finding:
- Set issue_type to manuscript_issue, parser_artifact, or cannot_verify.
- Set category to notation_inconsistency, equation_text_mismatch, assumption_mismatch, sign_or_definition_error, model_interpretation_overreach, or cannot_verify.
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Include source_objects for equations, text, definitions, propositions, or parsed artifacts.
- Include claim_evidence_links when comparing a textual model claim to an equation or definition.
- Use cannot_verify only when artifacts do not allow a check, and include cannot_verify_reason.

Do not include process notes about skill availability, tooling, or session state.
````

#### `numerical_audit.txt`

````text
Use the custom agent named numerical_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "numerical_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted NUM-001, NUM-002, ...

Focus on:
- numeric claims in text versus tables/figures/appendix
- sign, scale, units, rounding
- percentage versus percentage-point distinctions
- cannot_verify when parsed artifacts are insufficient

For every finding:
- Set issue_type to manuscript_issue, parser_artifact, or cannot_verify.
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Include source_objects for the text/table/figure/appendix material used as evidence.
- For verifiable numerical findings, include numeric_check with reported_value, expected_value, method, inputs, and recomputation_notes.
- Use cannot_verify only when the artifact does not allow a check, and then include cannot_verify_reason.

When a finding may overlap with claim_evidence_auditor, state the numeric defect cleanly so the editor can deduplicate it.

Do not include process notes about skill availability, tooling, or session state.
````

#### `robustness_audit.txt`

````text
Use the custom agent named robustness_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit robustness and sensitivity support only:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "robustness_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted ROB-001, ROB-002, ...

Focus on:
- robustness checks versus main claims
- alternative specifications, controls, subsamples, placebo tests, sensitivity analysis, and appendix validations
- cases where a paper says results are robust but the supporting evidence is mixed, missing, or narrower

For every finding:
- Set issue_type to manuscript_issue, parser_artifact, or cannot_verify.
- Set category to robustness_overstatement, missing_robustness_support, specification_sensitivity, appendix_mismatch, or cannot_verify.
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Include source_objects for main claims and robustness evidence.
- Include claim_evidence_links when comparing claims to robustness evidence.
- Use cannot_verify only when artifacts do not allow a check, and include cannot_verify_reason.

Do not include process notes about skill availability, tooling, or session state.
````

#### `sample_construction_audit.txt`

````text
Use the custom agent named sample_construction_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit sample construction, data-flow, and variable construction only:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "sample_construction_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted SAMPLE-001, SAMPLE-002, ...

Focus on:
- sample restrictions, exclusions, attrition, denominators, missingness, and observation counts
- variable definitions and transformations
- consistency between data section, tables, figures, appendices, and result interpretation

For every finding:
- Set issue_type to manuscript_issue, parser_artifact, or cannot_verify.
- Set category to sample_restriction_mismatch, attrition_unclear, denominator_mismatch, variable_construction_unclear, data_flow_inconsistency, or cannot_verify.
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Include source_objects for relevant data descriptions, tables, figures, appendix material, or parsed artifacts.
- Include claim_evidence_links when comparing a data claim to a table or appendix source.
- Use cannot_verify only when artifacts do not allow a check, and include cannot_verify_reason.

Do not include process notes about skill availability, tooling, or session state.
````

#### `abstract_conclusion_consistency_audit.txt`

````text
Use the custom agent named abstract_conclusion_consistency_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit consistency between summary sections and the manuscript body:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "abstract_conclusion_consistency_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted CONSIST-001, CONSIST-002, ...

Focus on:
- abstract, introduction, and conclusion claims that overstate or contradict the body
- stale summary language after edits
- missing caveats in summary sections
- conclusion or policy statements that exceed the reported evidence

For every finding:
- Set issue_type to manuscript_issue, parser_artifact, or cannot_verify.
- Set category to abstract_overstatement, conclusion_overstatement, stale_summary_claim, section_inconsistency, or cannot_verify.
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Include source_objects for both the summary claim and the body evidence where possible.
- Include claim_evidence_links when comparing summary language to body evidence.
- Use cannot_verify only when artifacts do not allow a check, and include cannot_verify_reason.

Do not include process notes about skill availability, tooling, or session state.
````

#### `claim_evidence_audit.txt`

````text
Use the custom agent named claim_evidence_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "claim_evidence_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted CEA-001, CEA-002, ...

Focus on:
- whether major textual claims are supported by displayed evidence
- overstatement
- weak support
- missing support
- cannot_verify when artifacts do not allow a check

For every finding:
- Set issue_type to manuscript_issue, parser_artifact, or cannot_verify.
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Include source_objects for the claim location and the tables, figures, or results used as evidence.
- Include claim_evidence_links connecting the claim to source_objects with relation supports, contradicts, partial, or cannot_verify.
- Use cannot_verify only when the artifact does not allow a check, and then include cannot_verify_reason.

If a finding overlaps with a numerical issue, focus on the interpretation/evidence problem and avoid repeating the full arithmetic diagnosis unless it adds new value.

Do not include process notes about skill availability, tooling, or session state.
````

#### `data_availability_replication_audit.txt`

````text
Use the custom agent named data_availability_replication_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit preregistration, pre-analysis-plan documentation, planned-vs-reported analyses, and scoped reproducibility claims only:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "data_availability_replication_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted REPL-001, REPL-002, ...

Focus on:
- whether experiments, RCTs, surveys, or other prospectively planned empirical designs identify preregistrations or pre-analysis plans
- registry records, pre-analysis plans, or preregistration materials linked or explicitly identified by the manuscript
- consistency between planned and reported samples, exclusions, treatment arms, outcomes, estimands, specifications, weighting, and robustness checks
- whether deviations from the plan are disclosed clearly enough for readers to distinguish planned analyses from exploratory or changed analyses
- data, code, or replication-package claims only when the manuscript makes a concrete availability/reproducibility claim, a directly relevant availability statement is at issue, or missing materials prevent checking a material planned-vs-reported issue

Treat AEA RCT Registry, OSF Registries, AsPredicted, and similar preregistration platforms as common non-exhaustive examples. If the manuscript provides registry URLs, DOIs, or explicit registry IDs, inspect those linked records when needed for the audit. Do not perform broad web searches for undisclosed plans.

Report only material, high-confidence issues. Do not manufacture checklist findings from minor omissions.

For every finding:
- Set issue_type to manuscript_issue, parser_artifact, or cannot_verify.
- Set category to data_availability_unclear, code_availability_unclear, replication_claim_mismatch, reproducibility_detail_missing, or cannot_verify.
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Include source_objects for data/code statements, empirical sections, appendix material, or parsed artifacts.
- Include claim_evidence_links when comparing reproducibility claims to manuscript evidence.
- Use cannot_verify only when artifacts do not allow a check, and include cannot_verify_reason.
- For registry lookup findings, cite both the manuscript claim and the linked external registry or pre-analysis-plan source. If a linked record is inaccessible, ambiguous, or insufficient, mark the relevant point cannot_verify rather than guessing.

Do not include process notes about skill availability, tooling, or session state.
````

#### `identification_audit.txt`

````text
Use the custom agent named identification_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit identification and causal interpretation only:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "identification_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted ID-001, ID-002, ...

Focus on:
- causal language and whether the design supports it
- stated and unstated identifying assumptions
- threats to validity, confounding, selection, spillovers, timing, and measurement
- whether limitations and interpretation match the identification strategy

For every finding:
- Set issue_type to manuscript_issue, parser_artifact, or cannot_verify.
- Set category to causal_claim_mismatch, identification_assumption, threat_to_validity, interpretation_overreach, or cannot_verify.
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Include source_objects for the claim, design description, table, figure, appendix, or parsed artifact used as evidence.
- Include claim_evidence_links when comparing a claim to design/evidence.
- Use cannot_verify only when artifacts do not allow a check, and include cannot_verify_reason.

Do not include process notes about skill availability, tooling, or session state.
````

#### `limitations_external_validity_audit.txt`

````text
Use the custom agent named limitations_external_validity_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit limitations, scope, and external validity only:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "limitations_external_validity_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted VALID-001, VALID-002, ...

Focus on:
- policy, welfare, scope, and generalizability claims
- limitations that are missing, understated, or inconsistent with the design/data
- external-validity language that exceeds the sampled population or setting

For every finding:
- Set issue_type to manuscript_issue, parser_artifact, or cannot_verify.
- Set category to external_validity_overreach, limitation_understated, scope_mismatch, policy_overstatement, or cannot_verify.
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Include source_objects for claims and limiting evidence.
- Include claim_evidence_links when comparing scope claims to evidence.
- Use cannot_verify only when artifacts do not allow a check, and include cannot_verify_reason.

Do not include process notes about skill availability, tooling, or session state.
````

#### `reference_audit.txt`

````text
Use the custom agent named reference_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "reference_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted REF-001, REF-002, ...

Prioritize citation/reference integrity issues:
- missing citation/reference pair
- wrong author/year/title identity
- duplicate entries
- wrong in-text key

Treat publication-status updates as secondary maintenance items unless they change the cited work's identity or materially affect the in-text year.

For every finding:
- Set issue_type to reference_integrity, bibliography_maintenance, parser_artifact, or cannot_verify.
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Include source_objects for in-text citations, reference-list entries, parsed reference artifacts, or external sources used.
- Use cannot_verify only when verification is not possible, and then include cannot_verify_reason.

Keep the main summary focused on integrity defects, not bibliography housekeeping.

Do not include process notes about skill availability, tooling, or session state.
````

#### `grammar_audit.txt`

````text
Use the custom agent named grammar_auditor.

Use the repo guidance and the paper-reviewer workflow.

Audit grammar and copyediting issues only:
`{parsed_dir}`

Return one JSON object that strictly conforms to:
`{schema_path}`

For this run:
- reviewer = "grammar_auditor"
- paper_id = "{paper_id}"
- finding IDs must be stable and formatted GRAM-001, GRAM-002, ...

Focus on high-signal reader-facing mistakes:
- grammar errors
- typos and duplicated words
- punctuation errors that affect readability
- word-choice mistakes
- sentence-level clarity problems that could confuse readers

Do not report:
- style preferences
- broad rewrites
- economics or evidence issues
- reference, citation, or literature-positioning issues
- parser artifacts unless the text is impossible to inspect, in which case use cannot_verify

Severity rules:
- high: the wording changes meaning or could materially confuse interpretation
- medium: a noticeable grammar or wording issue affects readability or polish
- low: an isolated typo, punctuation mistake, or minor copyedit

For every finding:
- Set issue_type to copyedit_issue or cannot_verify.
- Set category to grammar, typo, punctuation, word_choice, duplicated_word, or sentence_clarity.
- Set confidence to high, medium, or low.
- Set location.precision to exact, partial, or missing.
- Put the problematic text in location.text_quote and claim_text.
- Put the correction in suggested_fix.
- Include source_objects for the parsed page or full-text artifact used as evidence.
- Use cannot_verify only when the parsed artifacts do not allow a check, and then include cannot_verify_reason.

These findings are for an appendix markdown table, not the main substantive report. Keep each finding concise and actionable.

Do not include process notes about skill availability, tooling, or session state.
````

### C.4 Editor Prompt

#### `editor_report.txt`

````text
Use the custom agent named editor.

You will receive:
1. a deterministic editor brief,
2. a normalized editor bundle, and
3. the original configured reviewer JSON outputs.

Produce the final markdown report for paper_id "{paper_id}".

Hard rules:
- The output must be the final markdown report itself.
- Do not write a note saying where the report was saved.
- Do not include process/tool/session notes.
- Do not emit replacement-question sequences such as ?? or ???; if source text contains characters that may render unreliably, paraphrase or normalize the quote to readable ASCII.
- Do not invent findings.
- Preserve uncertainty, especially cannot_verify cases.
- Preserve exact locations when available.
- Use confidence, source_objects, claim_evidence_links, numeric_check, and cannot_verify_reason fields when they are present in the normalized bundle.
- Use the deterministic editor brief as the organizing map for prioritization, section routing, optional-reviewer disclosure, and traceability coverage.
- Treat the deterministic editor brief as internal planning guidance. Do not reproduce run summaries, scoring tables, reviewer-count tables, active-reviewer tables, section-routing tables, or other audit-log metadata in the final report.
- Merge duplicate findings across reviewers into one canonical issue.
- Prioritize high- and medium-severity substantive findings.
- Separate manuscript issues from parser/preprocessing issues.
- Separate reference-integrity problems from low-severity reference-maintenance updates.
- Put copyedit_issue findings only in the grammar appendix unless they materially affect the meaning of a substantive issue.
- Do not silently soften strong reviewer findings.
- Keep the main report readable. Do not place canonical IDs or source finding IDs in the body of the report except in the final traceability appendix.
- Use the canonical IDs and source finding IDs exactly as listed in the normalized editor bundle.
- Include a Review Configuration section in prose, not as a table. Briefly state which optional reviewer agents were used and why; mention mandatory baseline reviewers only briefly. Mention partial or failed reviewer statuses only when they materially limit confidence in the report.
- Start with cross-agent synthesis. Do not organize any report section by agent.
- Use the Highest-Priority Cross-Agent Findings section for the strongest high-confidence issues recommended by the deterministic editor brief. Aim for 3 to 8 top-level issues when the evidence supports that range; use fewer than 3 if fewer high-confidence issues exist, and never exceed 8. Place remaining issues in Additional Findings or the appropriate domain section.
- Use Additional Findings for lower-priority findings that were not elevated to cross-agent synthesis. Do not organize this section by agent.
- Include a final Appendix: Traceability Map table that maps every canonical finding in the normalized bundle to its source finding IDs.

Suggested report structure:
# Multi-Agent Paper Review Report

## Executive Summary

## Review Configuration

## Highest-Priority Cross-Agent Findings

## Suggested Revision Priorities

## Additional Findings

## Literature Positioning and Novelty

## Reference Integrity and Bibliography Maintenance

## Parser and Preprocessing Caveats

## Items Marked Cannot Verify

## Appendix: Grammar and Copyediting Issues

## Appendix: Traceability Map

In Additional Findings, prefer a markdown table with these columns:
| Area | Location | Issue | Suggested revision |

In Items Marked Cannot Verify, prefer a markdown table with these columns:
| Claim or item | Why it cannot be verified | Needed evidence |

When copyedit_issue findings exist, include a markdown table under the appendix heading with these columns:
| Location | Current text | Issue | Suggested correction |

Use one row per grammar or copyediting finding. If no copyedit_issue findings exist, omit the appendix.

The Traceability Map appendix must be the only place where canonical and source IDs are printed. Include every canonical finding in the normalized bundle, even if the finding is otherwise summarized briefly. Use this markdown table:
| Report section | Finding | Canonical ID | Source finding IDs |

Use the deterministic editor brief first for organization. Use the normalized editor bundle at `{editor_bundle_path}` as the authoritative source for canonical IDs and grouped evidence. Use the original reviewer files only to recover detail, quotations, and context.
````
