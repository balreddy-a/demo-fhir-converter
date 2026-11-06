# Implementation Plan: Convert C-CDA Goals Section to FHIR Goal

**Branch**: none (spec directory `specs/001-wire-goals-section`) | **Date**: 2026-11-05 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-wire-goals-section/spec.md`

**Status**: PLAN ONLY. Per FR-012 nothing is implemented, and nothing may be until a developer explicitly approves the output-shape change and the decisions in "Approvals required". No application code, template or test was modified while writing this plan. No tests, conversions or validators were run.

**Sources read**: the spec, `.specify/memory/constitution.md` (v1.0.0, still a draft awaiting ratification), `CLAUDE.md`, `docs/day2/00-discovery.md`, and the repository files cited below.

## Summary

The Goals chain already exists (`Section/_Goal` to `Entry/Goal/_entry` to `Resource/_Goal` and `Reference/Goal/_Subject`) but it is never included by a root template and its entry template contains a `{$ ... $}` typo that makes conversion fail. The plan is:

1. Fix the typo and add a status guard in the entry template, so a goal is emitted only when it has an R4 lifecycle status (FR-002, FR-005, FR-005a, FR-005b).
2. Include `Section/Goal` in `CCD.liquid` only (FR-001, scope).
3. Add a synthetic fixture, tests and a before/after regression diff (FR-006, FR-007, FR-010).
4. Update `CHANGES.md` and correct `docs/repo-analysis.md:111` (FR-011).
5. Optionally, with separate approval, add one new opt-in Python filter so a goal that fails at render time is skipped without failing the document (FR-013). This is the only Python change and it is not assumed.

## Technical Context

**Language/Version**: Liquid templates (python-liquid 1.12.1, pinned). Python only if the optional isolation filter is approved. Library modules must stay Python 3.8 compatible.

**Primary Dependencies**: unchanged. No dependency or pin changes.

**Storage**: N/A.

**Testing**: pytest (`.\.venv\Scripts\python.exe -m pytest -q`, baseline expected 475 passed, not re-run here). Synthetic fixtures under `data/synthetic/ccda/`.

**Target Platform**: library and CLI, CI on Ubuntu, Windows and macOS, Python 3.8 to 3.12.

**Project Type**: library plus CLI.

**Performance Goals**: none specified. The guard adds a handful of Liquid comparisons per goal entry.

**Constraints**: preserve ids, public API, CLI options, exit codes and the output of documents with no Goals Section (FR-006, FR-007, FR-009). No new logging or output channel without agreement (`CLAUDE.md` section 5).

**Scale/Scope**: one root template, one entry template, one fixture, one test module section.

**FHIR target**: FHIR R4. The task asks for R4 4.0.1, and the repository says only "R4" (see spec Clarifications). Fields below are taken from the published R4 Goal definition (https://hl7.org/fhir/R4/goal.html) as I know it. I did not fetch the page in this session, so task T001 re-checks each statement against it before implementation.

### R4 Goal elements that matter here

| Element | R4 cardinality | Current template | Plan |
|---|---|---|---|
| `lifecycleStatus` | 1..1, required binding to `GoalLifecycleStatus` | Copies `statusCode.code` raw, empty when absent | Guard: emit the Goal only when the code is in the allowed set |
| `description` | 1..1 | From `code` | **Spec gap, see Decision D4** |
| `subject` | 1..1 | From the patient reference | Unchanged. Must stay attached to the same Goal id |
| `identifier`, `priority`, `target.measure`, `target.detailQuantity`, `target.dueDate` | 0..* or 0..1 | Present | Unchanged (FR-004, FR-008) |
| `achievementStatus`, `startDate`, `addresses`, `expressedBy` | 0..1 or 0..* | Not mapped | Out of scope, no new mappings |

**Allowed `GoalLifecycleStatus` codes** (published R4 value set, to be confirmed in T001 against https://hl7.org/fhir/R4/valueset-goal-status.html): `proposed`, `planned`, `accepted`, `active`, `on-hold`, `completed`, `cancelled`, `entered-in-error`, `rejected`. The guard accepts a source code only when it is exactly one of these. No other source code is translated into one (FR-005b). The C-CDA status codes the guard will meet are not stored in the repository and are not enumerated here.

## Constitution Check

*GATE: checked before Phase 0 and again after Phase 1 design. The constitution is v1.0.0 and unratified, so this is a check against a draft.*

| Principle | Assessment | Result |
|---|---|---|
| I. Template-First Mapping | Mapping and guard live in templates. New files use the `_` prefix rule (none are new). The section is found by OID and rendered through `batch_render`. Python is limited to one optional, approval-gated filter. | Pass, with the Python item flagged |
| II. Output Stability | Output shape changes for CCD bundles that have a Goals Section. That needs explicit approval (FR-012) and a `CHANGES.md` entry. Ids, API, date rule are untouched. Documents without goals must diff clean. | Pass only after approval |
| III. Synthetic Data and Privacy | Fixture invented, no `data/sample` content, diagnostics carry no source data (FR-014). | Pass |
| IV. Verify Before Claiming | Baseline suite, before/after JSON diff and new tests are planned. No FHIR-valid or conformance claim is made. The R4 facts above are unverified until T001. | Pass |
| V. Minimal, Scoped Change | Two template edits, one fixture, tests, two doc edits. No mass edit. The Instructions typo is left alone. | Pass |
| Technical Constraints | Python 3.8, type annotations, flake8 limits apply if the filter is approved. | Pass if approved |

No unjustified violations. Complexity Tracking is empty.

## Existing Liquid relationships (verified by reading the files)

```text
CCD.liquid  (Goal NOT included today)
  -> {% include 'Section/Goal' %}                      [NEW line, inside the Bundle "entry" array]
       Section/_Goal.liquid
         msg | get_first_ccda_sections_by_template_id: '2.16.840.1.113883.10.20.22.2.60'
         .2_16_840_1_113883_10_20_22_2_60.entry | to_array | batch_render: 'Entry/Goal/entry', 'entry'
           Entry/Goal/_entry.liquid
             assign observationId = entry.observation | to_json_string | generate_uuid       [id hash, unchanged]
             include 'Resource/Goal'            goalEntry: entry.observation, ID: observationId
             {$ include 'Reference/Goal/Subject' ... $}   <-- DEFECT, must be {% ... %}
               Resource/_Goal.liquid     -> Goal with fullUrl, identifier, description, lifecycleStatus, priority, target, request(PUT)
               Reference/Goal/_Subject   -> second Goal entry (id + subject) with no fullUrl
```

Variables the chain needs from the root: `msg` and `fullPatientId` (both already defined in `CCD.liquid`).

**Important merge behavior** (`hl7.py:240-252`, `get_fhir_entry_key`): the Resource and the Subject reference each emit a bundle entry with the same resource type and id, and `parse_fhir` merges them. Consequence: if the guard skips only the Resource include, the Subject include still emits a stub `Goal` with only an id and a subject and no status, which is an invalid R4 Goal. **The guard MUST wrap both includes**, so a skipped goal leaves nothing behind. A test must assert there is no orphan Goal stub.

### Files

| File | Change | Why |
|---|---|---|
| `fhir_converter/templates/ccda/Entry/Goal/_entry.liquid` | Fix `{$ ... $}` to `{% ... %}` and wrap both includes in the status guard | FR-002, FR-005, FR-005a |
| `fhir_converter/templates/ccda/CCD.liquid` | Add `{% include 'Section/Goal' -%}` | FR-001 |
| `fhir_converter/templates/ccda/Resource/_Goal.liquid` | Likely none. Possible small edit only if T001 or D4 shows a required element is written empty | Keep the diff minimal |
| `data/synthetic/ccda/TD-CCDA-13_goals.ccda` | New synthetic fixture | FR-010 |
| `tests/test_renderers.py` (matching module) | New tests | FR-010 |
| `CHANGES.md` | Entry for the output-shape change | FR-011 |
| `docs/repo-analysis.md` line 111 | Correct the "present in other root templates" statement | FR-011 |
| `docs/test-cases.md` | New cases, statuses set only after they are actually run | `CLAUDE.md` section 6 |
| `fhir_converter/filters.py`, `tests/test_filters.py` | **Only if D3 option B is approved** | FR-013 |

Not changed: `Section/_Goal.liquid`, `Reference/Goal/_Subject.liquid`, `Utils/_GenerateId`, `ValueSet.json`, all other root templates, `Entry/Instruction/_entry.liquid`, `pyproject.toml`, `__main__.py`, `renderers.py`.

## Design

### Status guard (template only)

In `Entry/Goal/_entry.liquid`, read `entry.observation.statusCode.code`, test it for exact membership in the allowed set, and render both includes only on a match. Membership can be written in Liquid (split a fixed string and use `contains`, or a `case` block), so no Python and no `ValueSet.json` key is needed. The allowed set stays in that one template.

Properties this gives:
- A missing, empty or unrecognised status renders nothing for that entry (FR-005a). No default is substituted and no source value is mapped (FR-005b).
- An entry with no `observation` (for example a non-observation entry in the section) also has no status, so it is skipped by the same guard.
- `observationId` is computed with the same expression as today, so Goal ids stay what the existing templates would have produced (FR-007).
- Output from a skipped entry is empty text, which `batch_render` tolerates: it appends a comma only when the buffer tail is non-empty and does not end in a comma (`filters.py:341-343`).

### How malformed entries are isolated without breaking `batch_render`

Two kinds of malformed entry behave differently.

**Data-level (missing or unrecognised status, missing observation).** Handled by the guard above. No change to `batch_render`. Covers FR-005a and the data cases in FR-013.

**Render-time failure (an exception while the entry renders).** `batch_render` (`filters.py:338-340`) renders every entry into one shared buffer with no try/except, and the discovery probe and `TC-DT-05` show that one failing entry fails the whole document. Liquid has no catch construct, so a template cannot trap it. A concrete Goal example: `Resource/_Goal.liquid:42` calls `add_hyphens_date`, which raises `Malformed HL7 datetime` for a bad `effectiveTime`, and the exception message includes the date text (the TC-DT-05 cause text shows this). A second failure mode is a rendered fragment that is not valid JSON, which the final JSON5 parse rejects for the entire document.

Isolation therefore needs Python. The least invasive design is a **new, opt-in filter**, for example `batch_render_isolated`, added beside `batch_render`:
- It renders each entry into its own buffer, and on success appends that buffer to the output with the same trailing-comma rule.
- On an exception it discards that entry's partial output and moves on, so the other entries still convert.
- It does not use `str(exception)` or any source value in anything it records (FR-014).
- `Section/_Goal.liquid` would call it instead of `batch_render`. Every other section keeps `batch_render` untouched, so no other conversion changes.

Not recommended: changing `batch_render` itself, because that alters failure behavior for all 30 sections and every root template.

Limits to state plainly: the filter catches exceptions raised while a single entry renders. It does not repair a fragment that renders without error but is invalid JSON, because that only fails at the later whole-document parse. Whether per-entry buffers work with python-liquid 1.12.1's `Context.get_buffer`/`extend` as `batch_render` uses them is **not verified**. Task T010 prototypes it outside the repository first. `tags.py` and these internals are fragile (`CLAUDE.md` section 4).

### Is Python necessary?

| Requirement | Needs Python? |
|---|---|
| FR-001, FR-002, FR-005, FR-005a, FR-005b, FR-008 | No. Templates only |
| FR-013 for data-level malformed entries | No |
| FR-013 for entries that fail at render time (as the spec text literally says, "cannot be rendered") | **Yes**, one new filter in `filters.py` plus tests |
| FR-014 | Satisfied by design in both options |

**This needs developer approval** (D3). Without it, a goal with a malformed date still fails the whole document, exactly as every other section does today.

### Diagnostics (FR-014)

By default a skipped goal is silent, because the project has no logging and `CLAUDE.md` forbids new output channels without agreement. The consequence is that source goal data can disappear with no signal. If the developers want a signal, the only safe content is the section template id and the entry position, and the channel must be agreed first (D5). Under no option does the plan record a status value, date, id or display text, and the filter must never surface `str(exception)`.

## Approvals required before any implementation (FR-012)

| ID | Decision | Recommendation |
|---|---|---|
| D1 | Approve adding Goal resources to CCD bundles that contain a Goals Section (output-shape change) | Required gate. Nothing starts without it |
| D2 | Confirm CCD only. Other root templates and the Instructions typo stay out | As the spec states |
| D3 | Approve one new opt-in Python filter for render-time isolation (option B), or accept template-only isolation (option A) | Option A first as its own deliverable. Approve B only if FR-013 must cover render-time failures such as a malformed date |
| D4 | Spec gap: R4 `Goal.description` is 1..1. A goal with no description code would be an invalid Goal. Amend the spec to skip such goals, as FR-005a does for status | Amend the spec (needs your decision, because the spec does not say this today). I have not changed the spec |
| D5 | Is a diagnostic channel wanted for skipped goals, and if so which | None by default. Raise only if silent loss is unacceptable |

## Regression and test strategy

**Baseline first** (constitution IV): run `.\.venv\Scripts\python.exe -m pytest -q` and record the count. Not run for this plan. flake8, mypy and Poetry were not installed locally when last checked.

**Before/after output diff**: before any edit, convert and save to a temporary folder (never `data/out`, not committed) the outputs of `TD-CCDA-01` to `TD-CCDA-12` (excluding `TD-CCDA-08`, which fails by design, see `TC-DT-05`), `tests/data/ccda/CCD.ccda` and `tests/data/ccda/History_and_Physical.ccda`. Repeat after the change and diff. Expected: zero differences (SC-003, FR-006). None of these contains a Goals Section (discovery C.3), which T002 re-confirms by search.

**New synthetic fixture** `data/synthetic/ccda/TD-CCDA-13_goals.ccda`, built from the existing synthetic header and one Goals Section, invented values only:

| Goal | Purpose | Expected |
|---|---|---|
| A: status is an R4 code, with description, due date, priority | Happy path (US1 scenarios 1, 2, 5) | Goal emitted |
| B: valid status, quantity target with unit | Target mapping (US1 scenario 3) | Goal with `detailQuantity` |
| C: no `statusCode` | US3 scenario 2 | No Goal, no stub |
| D: status code is an invented value that is not in the R4 set | US3 scenario 3 | No Goal, raw value appears nowhere in output (except inside the embedded source copy, see risk R1) |
| E: valid status, no priority, no target, no due date | FR-008 | Goal without empty structures |

SC-008 is covered by C, D plus two valid goals (A, B or E). The fixture avoids real C-CDA status codes beyond the plain R4-matching ones, because the C-CDA code list is not verified.

**Automated tests** (pytest, in the matching `tests/test_renderers.py` section, reading the fixture by relative path as existing tests do):
1. Each emitted Goal has a `lifecycleStatus` in the allowed set (SC-005) and a `subject` equal to the Patient reference (SC-002).
2. Exactly the expected number of Goals for the fixture, and no Goal entry that lacks `lifecycleStatus` (catches orphan stubs).
3. No Goal contains the invented unrecognised value or any default status (SC-008, SC-009).
4. Two conversions give identical Goal ids (SC-006).
5. A document with no Goals Section, and one with an empty Goals Section, produce no Goal and match the pre-change output (SC-003).
6. The conversion completes (no exception) for the full fixture.
7. If option B is approved: a fixture goal with a malformed `effectiveTime` is skipped while the other goals convert, and no exception text or date appears in anything the filter records. Unit tests in `tests/test_filters.py` for the new filter.

**Manual**: add `TC-MAP-*` cases to `docs/test-cases.md` for the fixture, and mark them only after running them.

**Not covered**: FHIR validity. The project runs no validator, so tests check the specific R4 rules above, not conformance.

## Risks

| # | Risk | Mitigation |
|---|---|---|
| R1 | **Privacy**: converter output embeds the whole source document (`_originalData`, `DocumentReference`). An "unrecognised value appears nowhere" test must exclude that embedded copy, and emitted Goals still contain real clinical text for real documents | Synthetic fixture only. Output goes to a temp folder. Do not describe the converter as privacy-safe |
| R2 | **Privacy**: exception text can carry source data (TC-DT-05 cause text includes the date; JSON5 errors carry partial output per the discovery) | The isolation filter must discard exceptions without reading their message. Existing leakage in the CLI is a separate, pre-existing item (discovery question 10) |
| R3 | **Silent data loss**: skipped goals leave no trace by default | Decide D5. Document it in `CHANGES.md` |
| R4 | **Output shape change**: CCD consumers gain `Goal` entries | D1 approval, `CHANGES.md` entry, before/after diff proves no change for other documents |
| R5 | **Orphan stub**: guarding only the Resource include leaves a status-less Goal from the Subject include | Guard wraps both. Test 2 |
| R6 | **Allowed set drift**: the R4 code list is written into one template from memory of the spec | T001 verifies it against the official page. Single location, easy to correct |
| R7 | **R4 minor version**: repo says only "R4" | State R4, confirm 4.0.1 in T001 |
| R8 | **Python filter fragility** (option B): relies on python-liquid 1.12.1 internals like `tags.py` | Prototype outside the repo, keep `batch_render` untouched, 3.8-safe typing |
| R9 | **Id stability**: edits to the entry template could change the hashed text | Keep the `observationId` line byte-identical. Test 4 and the diff |
| R10 | **Constitution unratified** | Treat as draft guidance, no compliance claim made |
| R11 | **Other entry types in the section** (non-observation entries) are skipped silently by the guard | Stated limit. Out of scope |
| R12 | **No FHIR validation** | No conformance claim anywhere. Tests check named R4 rules only |

## Project Structure

### Documentation (this feature)

```text
specs/001-wire-goals-section/
├── plan.md              # This file
├── research.md          # Phase 0
├── data-model.md        # Phase 1
├── quickstart.md        # Phase 1
├── contracts/
│   └── goal-output-contract.md   # Phase 1
└── tasks.md             # Phase 2 (/speckit-tasks, not created here)
```

### Source code (repository root)

```text
fhir_converter/templates/ccda/
├── CCD.liquid                     # +1 include line
├── Entry/Goal/_entry.liquid       # typo fix + status guard
├── Section/_Goal.liquid           # unchanged (or calls the new filter if D3-B)
├── Resource/_Goal.liquid          # unchanged unless T001 or D4 requires
└── Reference/Goal/_Subject.liquid # unchanged
fhir_converter/filters.py          # ONLY if D3-B approved
data/synthetic/ccda/TD-CCDA-13_goals.ccda   # new
tests/test_renderers.py            # new tests
tests/test_filters.py              # only if D3-B approved
CHANGES.md, docs/repo-analysis.md, docs/test-cases.md
```

**Structure Decision**: existing single-project library layout. No new directories under the package.

## Complexity Tracking

No constitution violations. The optional filter is the only added complexity and is justified in D3.
