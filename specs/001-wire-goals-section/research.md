# Research: Wire the Goals Section

Date: 2026-11-05. Everything below was read from repository files. Nothing was executed. Items that need an external source are marked **Unverified** and become task T001.

## R-1: FHIR version

- **Decision**: target FHIR R4. Confirm 4.0.1 in T001.
- **Evidence**: `README.md:40-41`, `docs/architecture.md:7`, `docs/repo-analysis.md:13`, and `lifecycleStatus` in `Resource/_Goal.liquid:17`.
- **Uncertainty**: the repository never names 4.0.1.

## R-2: Goal status rule

- **Decision**: emit a Goal only when the source status is exactly a code in the R4 `GoalLifecycleStatus` set. Skip otherwise. No default, no mapping (FR-005, 005a, 005b).
- **Rationale**: `lifecycleStatus` is 1..1 in R4, per the requester and the cited page.
- **Unverified**: the code list (`proposed`, `planned`, `accepted`, `active`, `on-hold`, `completed`, `cancelled`, `entered-in-error`, `rejected`) is from the published R4 value set and is not stored in the repo. No `ValueSet/Goal...` key exists in `ValueSet.json`. T001 checks it at https://hl7.org/fhir/R4/valueset-goal-status.html.
- **Alternatives rejected**: pass-through (current, invalid output), default status (invented), cross-code mapping such as C-CDA to R4 equivalents (spec forbids until a developer decides, with a cited source).

## R-3: Where the guard lives

- **Decision**: in `Entry/Goal/_entry.liquid`, wrapping both includes.
- **Rationale**: `parse_fhir` merges the Resource entry and the Subject entry by resource type and id (`hl7.py:240-252`). Skipping only the Resource would leave a status-less stub from the Subject include.
- **Alternatives**: guard in `Resource/_Goal.liquid` (leaves the stub), a `ValueSet.json` lookup (unmapped codes return an empty string, which would hide the problem, contrary to `CLAUDE.md` section 3).

## R-4: Isolating a failing entry

- **Finding**: `batch_render` (`filters.py:338-340`) has no try/except and shares one buffer. A render exception fails the document (`TC-DT-05`, discovery probe). Liquid cannot catch it.
- **Decision**: data-level cases by template guard (no Python). Render-time cases need a new opt-in filter, which requires approval (D3). Do not modify `batch_render`.
- **Unverified**: whether per-entry buffers via `Context.get_buffer`/`extend` behave correctly in python-liquid 1.12.1. Prototype outside the repo (T010).
- **Limit**: cannot repair a fragment that renders without error but is invalid JSON.

## R-5: Date failure example

- `add_hyphens_date` (`filters.py:148-152`) returns an empty string for blank input but calls `hl7_to_fhir_dtm` otherwise, which raises on a malformed value. `Resource/_Goal.liquid:42` uses it. The raised message contains the bad text (TC-DT-05 cause text), so diagnostics must never use it.

## R-6: Description is also required

- R4 `Goal.description` is 1..1. The existing template writes an empty `description` when `code` is absent, which `parse_fhir` prunes, leaving an invalid Goal. **The spec does not cover this** (D4). Not changed in this plan.

## R-7: Placement in the CCD root

- Any position inside the `entry` array works. Other sections' output is independent of it. Plan: after `Section/Payer` and before the `DocumentReference` lines. Documents without goals are unaffected by position.

## R-8: Fixture location

- Existing pytest tests read `tests/data/...` and relative `data/templates/ccda`. Manual cases use `data/synthetic/`. Plan: one fixture at `data/synthetic/ccda/TD-CCDA-13_goals.ccda`, read by relative path in pytest, to avoid two copies.

## R-9: Existing coverage

- No test or fixture mentions these OIDs (discovery C.3). Existing CCD fixtures (`tests/data/ccda/CCD.ccda`, `History_and_Physical.ccda`) are expected to have no Goals Section. T002 re-checks by search.
