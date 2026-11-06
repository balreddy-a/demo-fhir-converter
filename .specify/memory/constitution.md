<!--
SYNC IMPACT REPORT (temporary review scratch, remove before commit)
Version change: (unversioned template) -> 1.0.0
Modified principles: none (all five principles are new; template placeholders replaced)
  I.   Template-First Mapping
  II.  Output Stability
  III. Synthetic Data and Privacy (NON-NEGOTIABLE)
  IV.  Verify Before Claiming
  V.   Minimal, Scoped Change
Added sections: Technical Constraints, Development Workflow, Governance rules
Removed sections: none
Source: derived from CLAUDE.md (sections 1 to 9) and docs/day2/00-discovery.md. No principles
  were supplied by the user for this run, so every rule below mirrors an existing CLAUDE.md rule.
Deferred items:
  - TODO(RATIFICATION_DATE): this text is a draft until the project developers approve it.
    Set the date on approval.
-->
# python-fhir-converter Constitution

## Core Principles

### I. Template-First Mapping
Mapping logic lives in Liquid templates, not in Python. A change to what the converter emits
MUST be made in `fhir_converter/templates/` unless a template cannot express it. Python code
changes are limited to the loaders, renderers, parsers, filters, tags and CLI. A new template in
a sub-folder MUST use the `_` file prefix (`Section/Problem` loads `Section/_Problem.liquid`).
A C-CDA section MUST be found by templateId OID and each entry rendered through `batch_render`.
Rationale: the project is a library of mappings. Keeping them in templates keeps behavior
reviewable and lets users override them with `--template-dir`.

### II. Output Stability
Converter output is consumed by other systems, so these are fixed unless a spec explicitly
approves a change:
- The inputs hashed by `Utils/_GenerateId` and `generate_uuid` MUST NOT change, because that
  changes every consumer's resource ids.
- The public API MUST stay stable: `BaseFhirRenderer.render_fhir_string`, `render_fhir`,
  `render_to_fhir`, `make_environment`, `render_to_dir`, `render_files_to_dir`.
- The date rule (a datetime without a time zone is reduced to a date) MUST be kept.
- Changes to output shape, ids, exit codes, public API or dependencies MUST be raised for
  approval before implementation, and MUST be recorded in `CHANGES.md`.
Rationale: silent output changes break downstream data without any failing test.

### III. Synthetic Data and Privacy (NON-NEGOTIABLE)
Tests, docs, examples, commits, specs and chat MUST use only synthetic data. Files under
`data/synthetic/` are the approved source. Files under `data/sample` have unknown provenance and
MUST NOT be copied into tests, docs, specs or commits. Converter output embeds the whole source
document, so anything under `data/out` is sensitive and MUST NOT be committed. Patient data MUST
NOT be placed in exception messages, logs or UI error text. No real identifiers (NPI, MRN, DOB)
or addresses may be committed.
Rationale: the converter handles healthcare documents, and a leak is not reversible.

### IV. Verify Before Claiming
Work MUST be backed by evidence. The full test suite MUST be run before and after a change and
the real results reported. Template changes MUST be checked by converting a synthetic fixture
before and after and diffing the JSON. Anything not run MUST be reported as not run. Output
MUST NOT be described as FHIR-valid or US Core conformant, because no validator runs in this
project. Compliance language (HIPAA, SOC 2, HITRUST, ISO) MUST NOT be written as fact. Code,
citations and results MUST NOT be invented.
Rationale: unverified claims about clinical data conversion are a safety and trust risk.

### V. Minimal, Scoped Change
Diffs MUST be small and limited to the task. With about 1,090 template files, only the templates
the task needs may change, and none may be mass-reformatted or renamed. Dependencies and pins in
`pyproject.toml` MUST NOT change without approval. Generated or ignored paths (`htmlcov/`,
`site/`, `_site/`, `docs/docstrings/`, `.venv`) MUST NOT be edited. Commits and pushes happen
only when requested. Application code MUST NOT be modified for documentation or analysis work.
Rationale: a large template tree makes broad edits hard to review and easy to get wrong.

## Technical Constraints

- Library modules MUST stay compatible with Python 3.8 (CI tests 3.8 to 3.12). 3.9+ only syntax
  is allowed in `__main__.py` only. Using it in any other module requires approval.
- `python-liquid` stays at 1.12.1. `tags.py` uses internal python-liquid modules and is treated
  as fragile. Liquid templates need commas between parameters and explicitly named snippet
  variables, so templates copied from the Microsoft C# project need edits.
- Code is fully type-annotated. `mypy.ini` enables `warn_redundant_casts` and `warn_unreachable`.
  flake8 uses max line length 120, ignores `E203,E721`, max complexity 10. Docstrings use
  `Args:`, `Returns:`, `Raises:`.
- Failures surface as `RenderingError(msg, cause)` with the original error kept as the cause.
  Exceptions MUST NOT be swallowed. No `logging` or new output channels without agreement.
- Code translations belong in `templates/ccda/ValueSet/ValueSet.json`. Unmapped codes currently
  return an empty string, and template output is pruned by `parse_fhir`. Pruning MUST NOT be
  relied on to hide a wrong mapping.
- Any user interface or server added later MUST bind to loopback only by default and MUST NOT
  build file paths from request input.

## Development Workflow

1. Read the module and its tests. For template work, trace the include chain from the root
   template (`CCD.liquid`, then `Section/`, `Entry/`, `Resource/`, `DataType/`).
2. Search for existing behavior and check `docs/test-plan.md` observations that touch it.
3. Run the full test suite for a baseline (`.\.venv\Scripts\python.exe -m pytest -q`).
4. State the plan and the files that will change. Ask when the change affects output shape,
   ids, exit codes, public API or dependencies.
5. Add or update tests in the matching `tests/test_<module>.py`. The CLI has no unit tests, so
   CLI changes are checked manually against `docs/test-cases.md` (TC-CLI-*).
6. Run the tests again and report results with the commands used and anything not run.
7. Update `CHANGES.md` for user-visible behavior changes and `docs/test-cases.md` only after
   the cases have actually been run.

## Governance

This constitution governs how specs, plans, tasks and implementations are written for this
repository. Where it conflicts with a convenience practice, this constitution wins. `CLAUDE.md`
holds the detailed runtime guidance and MUST be consistent with this document.

Amendments MUST be proposed in writing with the reason, reviewed and approved by the project
developers, and recorded here with a new version. Versioning follows semantic versioning:
MAJOR for removing or redefining a principle, MINOR for adding a principle or materially
expanding guidance, PATCH for wording and clarifications.

Every spec, plan and pull request MUST be checked against these principles. A deviation MUST be
justified in the plan and approved before implementation. Compliance review is a review step, not
an attestation of regulatory compliance.

**Version**: 1.0.0 | **Ratified**: TODO(RATIFICATION_DATE): pending developer approval | **Last Amended**: 2026-11-05
