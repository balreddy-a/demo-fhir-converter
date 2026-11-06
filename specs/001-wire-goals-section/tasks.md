---

description: "Task list for the wire-goals-section feature"
---

# Tasks: Convert C-CDA Goals Section to FHIR Goal

**Input**: Design documents from `specs/001-wire-goals-section/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/goal-output-contract.md](contracts/goal-output-contract.md), [quickstart.md](quickstart.md)

**Status**: TASK LIST ONLY. Nothing is implemented. No application code, test, template or configuration file was changed while generating this list, and no command was run.

**About "approved"**: this list was requested from the "approved" spec and plan. The repository does not record that approval: `spec.md` still says "Draft. Not approved for implementation", and the plan lists decisions D1 to D5 as open. This list takes the request at face value for the spec and plan text, and keeps **GATE-D1** on every task that edits a file or runs a conversion, so a developer's approval of the output-shape change is recorded (task T004) before any work starts. It does not treat D1 to D5 as decided.

**Tests**: The spec requires tests (FR-010) and a regression diff (SC-003, SC-004), so test tasks are included, written before the change they cover.

**Organization**: Tasks are grouped by user story: **US1** Goals appear as FHIR Goals (P1), **US2** documents without goals convert exactly as before (P1), **US3** only goals with a valid status are emitted (P2). Setup, foundational, cross-cutting and documentation phases carry no story label.

## Format: `- [ ] T### [P] [US#] Description with file path`

- **[P]**: can run in parallel (different files, no dependency on an incomplete task).
- **[US#]**: user story from the spec.
- **⛔ GATE-Dn**: approval-gated. **Do not perform the task until a developer clears that gate.** See the gate table. Spec references (FR, SC, user story scenarios) follow each description.

## Approval gates

| Gate | What needs approval | Plan item |
|---|---|---|
| GATE-D1 | Start implementation: adding Goal resources to CCD bundles that have a Goals Section (the output-shape change). Required before any task that edits a file or runs a conversion or the suite | D1, FR-012 |
| GATE-D2 | Confirm CCD only. Other root templates and the Instructions typo stay out | D2 |
| GATE-D3 | Add one new opt-in Python filter so a goal that fails at render time is skipped (option B), or accept template-only isolation (option A) | D3, FR-013 |
| GATE-D4 | Amend the spec so a goal with no description code is skipped, because R4 `Goal.description` is 1..1 | D4 |
| GATE-D5 | Whether skipped goals need any diagnostic channel | D5, FR-014 |
| GATE-TOOLS | Installing flake8, mypy or Poetry. `CLAUDE.md` notes they were not installed locally | `CLAUDE.md` section 6 |

## Path conventions

Templates: `fhir_converter/templates/ccda/`. Fixture: `data/synthetic/ccda/`. Tests: `tests/test_renderers.py` (and `tests/test_filters.py` only under GATE-D3). Test commands use PowerShell, for example `.\.venv\Scripts\python.exe -m pytest tests\test_renderers.py -q`. Conversion output always goes to a temporary folder, never `data/out`. Synthetic data only (constitution III, `CLAUDE.md` section 8). The local `.venv` is Python 3.14.10, outside the supported 3.8 to 3.12, so CI is the compatibility authority for any Python change.

Note: the plan mentions placeholder task numbers (T001, T002, T004, T010). This file supersedes them: the R4 verification is T001, the fixture search is T002, the fixture is T007, and the filter prototype is T034.

---

## Phase 1: Setup and baseline

**Purpose**: Verify the facts the design rests on and measure the starting point. Results go to a new `specs/001-wire-goals-section/verification-log.md`. Record real results only, and list anything not run.

- [ ] T001 [P] Verify the R4 facts against the official pages (https://hl7.org/fhir/R4/goal.html and https://hl7.org/fhir/R4/valueset-goal-status.html): `Goal.lifecycleStatus` is 1..1, `Goal.description` is 1..1, `Goal.subject` is 1..1, the exact `GoalLifecycleStatus` codes, and that these hold for 4.0.1. Record each finding and its source in `specs/001-wire-goals-section/verification-log.md`. The plan's code list (`proposed`, `planned`, `accepted`, `active`, `on-hold`, `completed`, `cancelled`, `entered-in-error`, `rejected`) is from memory and unverified until this task is done (FR-005, plan risks R6, R7)
- [ ] T002 [P] Search `tests/`, `data/synthetic/` and `data/templates/` for the template id `2.16.840.1.113883.10.20.22.2.60` and for goal observations, and record that no existing fixture contains a Goals Section. If one does, record it and flag it for the before/after diff (SC-003, FR-006)
- [ ] T003 [P] Record a read-only `git status --short` snapshot in `specs/001-wire-goals-section/verification-log.md`, noting that `data/sample/ccda/CCD.ccda` and `data/sample/stu3/Patient.json` already show as modified and must not be touched or used (`CLAUDE.md` section 8)
- [ ] T004 ⛔ GATE-D1 Record the developers' rulings on D1 to D5 in `specs/001-wire-goals-section/plan.md` (Approvals table) and update the Status line of `specs/001-wire-goals-section/spec.md`. Until they are recorded, D3, D4 and D5 keep their plan defaults (template-only isolation, no description rule, silent skip) and the gated tasks below stay blocked (FR-012)
- [ ] T005 ⛔ GATE-D1 Run the baseline suite `.\.venv\Scripts\python.exe -m pytest -q` and record the passed count in `specs/001-wire-goals-section/verification-log.md` (expected 475, unverified) (SC-004)
- [ ] T006 ⛔ GATE-D1 Convert `data/synthetic/ccda/TD-CCDA-01` to `TD-CCDA-12` (excluding `TD-CCDA-08`, which fails by design, `docs/test-cases.md` TC-DT-05), `tests/data/ccda/CCD.ccda` and `tests/data/ccda/History_and_Physical.ccda` with `--template-name CCD --source-format CCDA` into `$env:TEMP\goals-before`, and record file hashes only in `specs/001-wire-goals-section/verification-log.md` (SC-003, FR-006)

**Checkpoint**: facts verified, baseline recorded, approval state recorded.

---

## Phase 2: Foundational

**Purpose**: The fixture and test scaffolding shared by all three stories. Blocks Phases 3 to 5.

- [ ] T007 [P] ⛔ GATE-D1 Create the synthetic fixture `data/synthetic/ccda/TD-CCDA-13_goals.ccda` from the structure of `data/synthetic/ccda/TD-CCDA-01_valid_basic.ccda` with one Goals Section (`2.16.840.1.113883.10.20.22.2.60`) holding five goal observations with invented data only: (A) a status that T001 confirms is an R4 code, with a description code, due date and priority; (B) a valid status with a quantity target (`value.low` value and unit); (C) no `statusCode`; (D) an invented status value that is certainly not an R4 code, for example `x-invented`; (E) a valid status with no priority, no target and no due date. No data from `data/sample` (FR-010, plan fixture table, constitution III)
- [ ] T008 ⛔ GATE-D1 ⛔ GATE-D4 Add goal (F) to `data/synthetic/ccda/TD-CCDA-13_goals.ccda`: a valid status with **no description code**. Only if D4 is approved (plan D4, FR-013)
- [ ] T009 ⛔ GATE-D1 Add the test scaffolding to `tests/test_renderers.py`: a new `CcdaGoalsTest` class that converts the fixture by relative path with `CcdaRenderer().render_to_fhir("CCD", ...)`, and small helpers that return the bundle's Goal entries, the Patient reference and the entries other than Goal. Match the style of the existing classes in that file (FR-010)
- [ ] T010 ⛔ GATE-D1 Reproduce the current failure outside the repository: in a temporary folder, create a root template that includes `Section/Goal` as in plan section "Existing Liquid relationships", convert the new fixture with `--template-dir`, and confirm the `pyjson5` error from the `{$ ... $}` typo. Record the result in `specs/001-wire-goals-section/verification-log.md` (FR-002, discovery C.5)

**Checkpoint**: the fixture exists, the failure is reproduced, test helpers are ready.

---

## Phase 3: User Story 1 - Goals appear as FHIR Goals (Priority: P1) 🎯 MVP part one

**Goal**: A CCD conversion of a document with a Goals Section converts without error and emits Goal resources linked to the patient.

**Independent Test**: convert the fixture. The conversion completes and the bundle contains Goal resources with the mapped fields and a patient reference.

**Release note**: US1 alone is not releasable. Until the US3 status guard exists, a goal with a missing or unrecognised status could be emitted, which breaks FR-005. Ship US1 and US3 together.

### Tests for User Story 1 (write first, confirm they fail before the fix)

- [ ] T011 [P] [US1] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that converting `data/synthetic/ccda/TD-CCDA-13_goals.ccda` with the `CCD` template completes without raising (FR-002, US1 scenario 1)
- [ ] T012 [P] [US1] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that a valid goal carries its identifier, description, priority, target measure, target quantity with unit and due date as mapped by the existing `Resource/_Goal.liquid` (goals A and B) (FR-004, US1 scenarios 1 to 3)
- [ ] T013 [P] [US1] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that every emitted Goal's `subject.reference` equals the Patient reference in the same bundle (FR-003, SC-002, US1 scenario 4)
- [ ] T014 [P] [US1] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that converting the fixture twice gives identical Goal ids (FR-007, SC-006, US1 scenario 5)
- [ ] T015 [P] [US1] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that goal (E), which has no priority, target or due date, is emitted with those parts omitted and no empty structures (FR-008)

### Implementation for User Story 1

- [ ] T016 [US1] ⛔ GATE-D1 Fix the typo on line 3 of `fhir_converter/templates/ccda/Entry/Goal/_entry.liquid`: change `{$ include 'Reference/Goal/Subject' ID: observationId, REF: fullPatientId -$}` to `{% include 'Reference/Goal/Subject' ID: observationId, REF: fullPatientId -%}`. Keep the `observationId` assignment on line 1 byte-identical so ids do not change (FR-002, FR-007, plan Design)
- [ ] T017 [US1] ⛔ GATE-D1 ⛔ GATE-D2 Add `{% include 'Section/Goal' -%}` to `fhir_converter/templates/ccda/CCD.liquid` inside the Bundle `entry` array, after `Section/Payer` and before the `DocumentReference` lines. Edit no other root template (FR-001, plan research R-7, D2)
- [ ] T018 [US1] ⛔ GATE-D1 Run `.\.venv\Scripts\python.exe -m pytest tests\test_renderers.py -q` and confirm T011 to T015 pass. Record that status-less Goals may still appear until Phase 5 (US1 independent test)

**Checkpoint**: US1 converts and maps goals. Do not release before Phase 5.

---

## Phase 4: User Story 2 - Documents without goals convert exactly as before (Priority: P1)

**Goal**: Existing consumers see no change for documents that have no Goals Section.

**Independent Test**: convert `TD-CCDA-01` and the other named fixtures before and after the change. The outputs are identical, and a document with an empty Goals Section converts with no Goal.

- [ ] T019 [P] [US2] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that converting `data/synthetic/ccda/TD-CCDA-01_valid_basic.ccda` produces no `Goal` resource (FR-006, US2 scenario 1)
- [ ] T020 [P] [US2] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that a document whose Goals Section has no entries (built in the test from the fixture text by removing the goal entries) converts successfully with no `Goal` resource (FR-006, US2 scenario 2)
- [ ] T021 [P] [US2] ⛔ GATE-D1 ⛔ GATE-D2 Add a test in `tests/test_renderers.py` that converting the fixture with another root template, for example `ConsultationNote`, produces no `Goal` resource, confirming CCD-only scope (plan scope, D2)
- [ ] T022 [P] [US2] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that all non-Goal bundle entries (resource type, id and content) are identical between the fixture and the same fixture with its Goals Section removed (built in the test), so no existing resource id changes (FR-006, FR-007)
- [ ] T023 [US2] ⛔ GATE-D1 Interim regression check: re-convert the same fixtures as T006 into `$env:TEMP\goals-after` and diff each file against `$env:TEMP\goals-before`. Expect zero differences. Record the result (SC-003, FR-006)

**Checkpoint**: US2 verified for the current state. The diff is repeated in Phase 7 after the guard.

---

## Phase 5: User Story 3 - Only goals with a valid status are emitted (Priority: P2)

**Goal**: A Goal is emitted only when its status is exactly an R4 `GoalLifecycleStatus` code. Missing or unrecognised statuses are skipped, with no stub, no default and no mapping.

**Independent Test**: convert the fixture. Goals A, B and E appear, goals C and D do not, every emitted Goal has a valid `lifecycleStatus` and a `subject`, and the unrecognised value appears nowhere as a status.

### Tests for User Story 3 (write first, confirm they fail before the guard)

- [ ] T024 [P] [US3] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that a goal whose status is an R4 code is emitted with exactly that `lifecycleStatus` (goals A, B, E) (FR-005, US3 scenario 1)
- [ ] T025 [P] [US3] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that goal (C), with no status, produces no `Goal` resource and no entry carrying its id, so there is no orphan stub (FR-005a, US3 scenario 2)
- [ ] T026 [P] [US3] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that goal (D), with an invented status, produces no `Goal` resource and the invented value is not used as a status by any Goal entry (FR-005a, US3 scenario 3). Do not search the embedded source copy, which legitimately contains the value
- [ ] T027 [P] [US3] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that the fixture yields exactly the expected Goals: with goals A, B, C, D and E the result is three Goals, matching the 2-of-4 pattern in SC-008 for the valid and invalid mix (SC-001, SC-008, US3 scenario 4)
- [ ] T028 [P] [US3] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that every emitted Goal has a `lifecycleStatus` in the allowed set confirmed by T001 and a `subject`, and that none carries a default or substituted status (SC-005, SC-009, FR-005b)

### Implementation for User Story 3

- [ ] T029 [US3] ⛔ GATE-D1 Add the status guard to `fhir_converter/templates/ccda/Entry/Goal/_entry.liquid`: read `entry.observation.statusCode.code`, test it for exact membership in the allowed set confirmed by T001 (a fixed string split and `contains`, or a `case` block, kept in this one template), and render **both** the `Resource/Goal` include and the `Reference/Goal/Subject` include only on a match, so a skipped goal leaves no stub. Leave the `observationId` line unchanged. Add no mapping and no default (FR-005, FR-005a, FR-005b, plan Design, risk R5, R9)
- [ ] T030 [US3] ⛔ GATE-D1 Run `.\.venv\Scripts\python.exe -m pytest tests\test_renderers.py -q` and confirm T024 to T028 and the Phase 3 and 4 tests all pass
- [ ] T031 [P] [US3] ⛔ GATE-D1 Add a test in `tests/test_renderers.py` that converting the fixture prints nothing extra to standard output or standard error for the skipped goals, and that no diagnostic contains any source value (FR-014, SC-007)
- [ ] T032 [US3] ⛔ GATE-D1 ⛔ GATE-D4 If D4 is approved: amend `specs/001-wire-goals-section/spec.md` so a goal with no description code is a malformed entry (extend FR-013 and add an acceptance scenario), add the matching condition to the guard in `fhir_converter/templates/ccda/Entry/Goal/_entry.liquid`, and add a test in `tests/test_renderers.py` using goal (F) from T008 that it is skipped. If D4 is not approved, do nothing and leave plan risk visible (plan D4, R-6)
- [ ] T033 [US3] ⛔ GATE-D1 ⛔ GATE-D5 If a diagnostic channel is approved: design and implement a diagnostic containing only the section template id and the entry position, with no identifiers, names, dates, codes, status values or exception text, and add a test for it. If not approved, do nothing: skipped goals stay silent (FR-014, plan Diagnostics, D5)

**Checkpoint**: US1, US2 and US3 together are releasable for the template-only option.

---

## Phase 6: Render-time isolation (US1, FR-013) ⛔ only if GATE-D3 option B is approved

**Goal**: A goal that fails while rendering, for example one with a malformed `effectiveTime`, is skipped and the other goals still convert. This is the only Python change in the feature. If D3 is not approved, skip this whole phase and do T049.

- [ ] T034 [US1] ⛔ GATE-D1 ⛔ GATE-D3 Prototype the per-entry buffer approach in a temporary folder outside the repository: confirm how python-liquid 1.12.1 `Context.get_buffer` and `Context.extend` (the pieces `batch_render` uses at `fhir_converter/filters.py:338-340`) behave when each entry renders into its own buffer and one entry raises. Record the result in `specs/001-wire-goals-section/verification-log.md`. Do not touch `fhir_converter/filters.py` yet (plan I-prototype, risk R8)
- [ ] T035 [P] [US1] ⛔ GATE-D1 ⛔ GATE-D3 Create the fixture `data/synthetic/ccda/TD-CCDA-14_goals_bad_date.ccda` with three goals, one of which has a malformed `effectiveTime` (invented data only) (FR-013, SC-007)
- [ ] T036 [P] [US1] ⛔ GATE-D1 ⛔ GATE-D3 Add unit tests in `tests/test_filters.py` for the new filter: a failing entry is skipped, the others render, the trailing-comma rule matches `batch_render`, the filter never records or surfaces exception text, and `batch_render` itself is unchanged (FR-013, FR-014)
- [ ] T037 [US1] ⛔ GATE-D1 ⛔ GATE-D3 Add the new opt-in filter, for example `batch_render_isolated`, to `fhir_converter/filters.py` beside `batch_render`: render each entry into its own buffer, on an exception discard that entry's partial output, append successful buffers with the same comma rule, never read `str(exception)`. Fully type-annotated, `Args:/Returns:/Raises:` docstring, Python 3.8 syntax only, flake8 limits (120 columns, complexity 10). If registering it needs an edit to `fhir_converter/renderers.py` beyond the existing builtin-filter pattern, stop and raise it, because that touches a public module (plan Design, `CLAUDE.md` sections 3, 4)
- [ ] T038 [US1] ⛔ GATE-D1 ⛔ GATE-D3 Change `fhir_converter/templates/ccda/Section/_Goal.liquid` to call the new filter instead of `batch_render`. Leave every other section on `batch_render` (plan Design)
- [ ] T039 [US1] ⛔ GATE-D1 ⛔ GATE-D3 Add a test in `tests/test_renderers.py` that converting `TD-CCDA-14_goals_bad_date.ccda` completes, the two valid goals are emitted, the bad one is not, and no date text from the bad goal appears in any diagnostic or output other than the embedded source copy (FR-013, FR-014, SC-007)
- [ ] T040 [US1] ⛔ GATE-D1 ⛔ GATE-D3 Confirm in `tests/test_filters.py` that existing `batch_render` tests still pass unchanged, and add one test that a render failure in any other section still fails the document as before (FR-009, plan risk R4)
- [ ] T041 ⛔ GATE-D1 ⛔ GATE-D3 ⛔ GATE-TOOLS Run flake8 and mypy over `fhir_converter/filters.py` and the new tests with the project settings if installed and approved, otherwise record "not run, CI covers it". Confirm the CI matrix (Python 3.8 to 3.12, Ubuntu, Windows, macOS) is green before merge, since the local `.venv` is 3.14.10 (plan Python note)

**Checkpoint**: render-time failures are isolated and the rest of the conversion is untouched.

---

## Phase 7: Regression and verification

**Purpose**: Prove nothing existing changed. No story label, because it spans all stories.

- [ ] T042 ⛔ GATE-D1 Run the full suite `.\.venv\Scripts\python.exe -m pytest -q` and compare the passed count with T005: the baseline plus the new tests, none failing (SC-004)
- [ ] T043 ⛔ GATE-D1 Final before/after diff: re-convert the same fixtures as T006 into `$env:TEMP\goals-after` and diff each against `$env:TEMP\goals-before`. Expect zero differences and record the result. If Phase 6 was done, include `Section/_Goal.liquid`'s effect (no other section changes) (SC-003, FR-006, plan regression)
- [ ] T044 ⛔ GATE-D1 Check Goal id stability and structure on the fixture output: Goal ids are identical across two runs, every Goal has `lifecycleStatus` and `subject`, no entry is a status-less stub. State the result as a check of these specific R4 rules only, never as FHIR validity or conformance (SC-005, SC-006, plan "no FHIR-valid claim")
- [ ] T045 Verify the change set is limited to the files the plan lists: compare `git status --short` with the T003 snapshot and confirm only the expected templates, fixture(s), `tests/test_renderers.py` (and `tests/test_filters.py` and `fhir_converter/filters.py` only under D3 option B), `CHANGES.md`, `docs/repo-analysis.md` and `docs/test-cases.md` changed, and that `data/sample` files were not touched, no dependency or pin changed, and `Instructions` entry template and all other root templates are untouched (FR-009, plan "Not changed" list)

**Checkpoint**: regression and scope verified, anything not run is listed.

---

## Phase 8: Documentation

- [ ] T046 [P] ⛔ GATE-D1 Add a `CHANGES.md` entry (format: version heading, bullet per change) recording the output-shape change: CCD bundles with a Goals Section now contain Goal resources, the entry-template typo was fixed, goals with a missing or unrecognised status are skipped with no default or mapping, and that skipped goals leave no trace by default (FR-011, plan risk R3, R4)
- [ ] T047 [P] ⛔ GATE-D1 Read line 111 of `docs/repo-analysis.md` and correct only the incorrect statement that the Goals, Health Concerns and Instructions sections are "present in other root templates", since a search shows no root template includes them. Make no other edit to that file (FR-011)
- [ ] T048 [P] ⛔ GATE-D1 Add `TC-MAP` style cases for `TD-CCDA-13_goals.ccda` to `docs/test-cases.md` and set a status only for cases that were actually run, leaving the others unset (`CLAUDE.md` section 6)
- [ ] T049 ⛔ GATE-D1 If D3 option A was chosen (template-only), record in `CHANGES.md` and `specs/001-wire-goals-section/spec.md` that a goal that fails at render time, for example with a malformed date, still fails the whole document as every other section does today, so FR-013 is met only for data-level cases (plan D3, FR-013)
- [ ] T050 ⛔ GATE-D1 Run every step of `specs/001-wire-goals-section/quickstart.md`, record real results and anything not run in `specs/001-wire-goals-section/verification-log.md`, update the Decision outcomes and `specs/001-wire-goals-section/checklists/requirements.md`, and make no FHIR-valid or conformance statement

---

## Dependencies and execution order

### Phase dependencies

- **Phase 1**: T001, T002, T003 are read-only and can start immediately. T004 must precede every GATE-D1 task. T005 and T006 must run before any edit so the baseline and the before-conversions come from unchanged code.
- **Phase 2**: depends on T001 (valid status codes for the fixture) and T004. T007 before T009 and T010.
- **Phase 3 (US1)**: depends on Phase 2. Tests T011 to T015 before the fix T016 and T017.
- **Phase 4 (US2)**: depends on Phase 3 for the interim diff (T023), and the tests can be written in parallel with Phase 3.
- **Phase 5 (US3)**: depends on Phase 2 and T016. T029 edits the same file as T016, so run them in order. T032 and T033 depend on their gates only.
- **Phase 6**: depends on Phases 3 and 5 and on GATE-D3. Skippable as a whole.
- **Phase 7**: depends on Phases 3 to 5, and on Phase 6 if it was done.
- **Phase 8**: documentation edits can start once the behavior is settled. T049 applies only if Phase 6 is skipped.

### User story dependencies

- **US1** (P1): needs Phase 2. Not releasable without US3.
- **US2** (P1): independent test needs only the unchanged documents, and the diff needs US1's wiring.
- **US3** (P2): needs US1's typo fix in the same template file. Release together with US1.

### Parallel opportunities

- Phase 1: T001, T002, T003 together.
- Phase 3: T011 to T015 together (same file, different test methods, so write them in one sitting if one author).
- Phase 4: T019 to T022 together.
- Phase 5: T024 to T028 together, then T031.
- Phase 6: T035 and T036 together.
- Phase 8: T046, T047, T048 together.

### Parallel example (Phase 3 tests)

```text
Task: "Add test that the fixture converts without error in tests/test_renderers.py"            (T011)
Task: "Add test that a valid goal carries the mapped fields in tests/test_renderers.py"         (T012)
Task: "Add test that every Goal references the Patient in tests/test_renderers.py"              (T013)
```

---

## Implementation strategy

### MVP (US1 plus US3 together)

1. Clear GATE-D1 and run Phase 1 (verification, baseline, before-conversions).
2. Complete Phase 2 (fixture and scaffolding).
3. Complete Phase 3 (US1) and Phase 5 (US3) before releasing anything, because US1 alone can emit invalid Goals.
4. Complete Phase 4 (US2) checks and Phase 7 regression.
5. **Stop and validate** with `quickstart.md`, then review with the developers.
6. Decide D3, D4 and D5, then add Phase 6 or the T049 note, and finish Phase 8.

### Rules while implementing

- Edit only the files the plan lists. Do not touch `Section/_Goal.liquid` (unless GATE-D3), `Reference/Goal/_Subject.liquid`, `Utils/_GenerateId`, `ValueSet.json`, other root templates, `Entry/Instruction/_entry.liquid`, `pyproject.toml`, `__main__.py` or `renderers.py`.
- Keep the `observationId` expression and everything hashed for ids unchanged.
- Do not add a status mapping or a default. Do not invent R4 codes: use only what T001 confirms.
- Do not perform any gated task before its gate is cleared, and do not resolve an open decision by choosing silently.
- Use synthetic data only and never `data/sample`. Write conversion output to a temporary folder.
- Report real results and list anything not run. Make no FHIR-valid or conformance claim.

---

## Notes

- Task IDs run T001 to T050.
- US1 spec scenarios and FR, SC and plan item references are those in `spec.md` and `plan.md` of this feature.
- This list was written without invoking the `speckit-tasks` skill, because that skill resolves its feature directory from `.specify/feature.json`, which currently points at the `002-template-management-ui` feature. Pointing it at this feature would have changed that file, which was out of scope.
