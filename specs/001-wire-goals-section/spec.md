# Feature Specification: Convert C-CDA Goals Section to FHIR Goal

**Feature Branch**: `001-wire-goals-section` (no git branch created; spec directory only)

**Created**: 2026-11-05

**Status**: Draft. Not approved for implementation: adding Goal resources to CCD bundles needs explicit developer approval first.

**Input**: User description: "Fix and wire the Goals section (2.16.840.1.113883.10.20.22.2.60) so C-CDA Goal entries convert to FHIR Goal resources. Use docs/day2/00-discovery.md as the repository discovery reference."

**Reference**: `docs/day2/00-discovery.md` section C (verified findings and the read-only probe). This spec is an engineering scope statement. It makes no claim that output is FHIR-valid or conformant to any profile, because the project runs no validator.

## Clarifications

### Session 2026-11-05

The four points below were supplied by the requester as decisions, so no question loop was run. Each is applied in the sections named.

- Q: Which FHIR version does the C-CDA conversion output, and what governs Goal status? → A: FHIR R4. Goal status MUST be limited to the R4 `GoalLifecycleStatus` value set. No source-to-target mapping beyond exact code matches is defined by this spec (FR-005, Assumptions).
- Q: What happens to a goal whose status is missing or not in the R4 set? → A: `Goal.lifecycleStatus` is 1..1 in R4 (https://hl7.org/fhir/R4/goal.html), so the goal is skipped, never emitted without a status and never given a default. This supersedes the earlier "status is omitted" wording (FR-005, FR-005a, FR-005b, US3).
- Q: Is adding Goal resources to CCD bundles already approved? → A: No. It is proposed functionality and requires explicit developer approval before implementation begins (Status, Assumptions, FR-012).
- Q: How are malformed Goal entries handled? → A: Valid entries still convert, a malformed entry is skipped, and no patient data appears in any diagnostic (FR-013, Edge Cases).
- Q: Which conversions are in scope? → A: Only the CCD root template. Other document types and the Instructions Section typo are out of scope (Assumptions).

**Repository evidence for the FHIR version (read directly, nothing executed):**
- `README.md:40-41` lists "CDA->FHIR R4" and "STU3->FHIR R4" as the supported conversions.
- `docs/architecture.md:7` says templates turn the source dictionary into "FHIR R4 JSON text".
- `docs/repo-analysis.md:13` gives the C-CDA output as a "FHIR R4 Bundle (type `batch`)".
- `fhir_converter/templates/ccda/Resource/_Goal.liquid:17` writes `lifecycleStatus`, which is the R4 element name. The STU3 Goal used `status`.
- `fhir_converter/templates/stu3/Goal.liquid` (the STU3 to R4 template) also writes `lifecycleStatus`, with values `completed` and `active`.

**Uncertainty:**
- The repository says "R4" but never names a minor version (for example 4.0.1). The spec therefore says R4 only.
- The R4 `GoalLifecycleStatus` codes are not stored in the repository. No `ValueSet/Goal...` key exists in `ValueSet.json`, and there is no copy of the official value set. The set is taken from the published HL7 FHIR R4 specification and MUST be confirmed against that source during planning. This spec does not quote the codes, to avoid recording an unchecked list.
- The set of C-CDA goal status codes is also not in the repository. The discovery's note that `aborted` has no FHIR equivalent remains an unverified assumption.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Goals in a C-CDA document appear as FHIR Goals (Priority: P1)

A developer or integrator converts a C-CDA Continuity of Care document that contains a Goals Section. Today the Goals Section is never reached by the standard conversion, and if it is reached by a custom template the conversion fails outright. After this feature, each goal in the section is emitted as a Goal resource in the output bundle, linked to the patient.

**Why this priority**: This is the whole feature. Without it, goal data in source documents is silently lost, and anyone trying to use the existing goal templates gets a hard failure.

**Independent Test**: Convert a synthetic C-CDA document that has a Goals Section with two goals. The output bundle contains two Goal resources, each with the goal description, status, due date, and a reference to the patient. The conversion completes without error.

**Acceptance Scenarios**:

1. **Given** a synthetic C-CDA document with a Goals Section holding one goal entry, **When** it is converted with the standard CCD conversion, **Then** the bundle contains exactly one Goal resource whose description, identifier, status and target date come from that entry.
2. **Given** a goal entry that has a priority, **When** it is converted, **Then** the Goal carries that priority.
3. **Given** a goal entry that has a measurable target value, **When** it is converted, **Then** the Goal target includes the measure and the quantity with its unit.
4. **Given** any converted Goal, **When** the bundle is read, **Then** the Goal refers to the same patient as the rest of the bundle.
5. **Given** the same input converted twice, **When** the outputs are compared, **Then** the Goal ids are identical.

---

### User Story 2 - Documents without goals convert exactly as before (Priority: P1)

An existing consumer converts documents that have no Goals Section. Their output must not change.

**Why this priority**: The change adds output to a widely used conversion. Consumers must be able to trust that unaffected documents produce byte-for-byte the same results.

**Independent Test**: Convert the existing synthetic fixture `TD-CCDA-01_valid_basic.ccda` (no Goals Section) before and after the change. The two outputs are identical, and the full existing test suite still passes.

**Acceptance Scenarios**:

1. **Given** a document with no Goals Section, **When** it is converted, **Then** the output contains no Goal resources and is identical to the output before this feature.
2. **Given** a document with a Goals Section that has no entries, **When** it is converted, **Then** the conversion succeeds and no Goal resources are produced.

---

### User Story 3 - Only goals with a valid status are emitted (Priority: P2)

An integrator receives goals whose C-CDA status is missing, or is a value that is not in the FHIR R4 goal lifecycle status set. In FHIR R4 every Goal must have a lifecycle status (cardinality 1..1, see https://hl7.org/fhir/R4/goal.html), so such a goal cannot be emitted as a valid Goal. It is skipped, and the other goals still convert.

**Why this priority**: The current behaviour copies the source status verbatim, and writes an empty one when it is absent. That produces a Goal that breaks the R4 rule. This is a correctness concern but affects fewer documents than Story 1.

**Independent Test**: Convert a synthetic document with one goal for each status value that matches the R4 set, one goal with no status, and one goal with an unrecognised status. Only the goals whose status exactly matches a code in the R4 set produce Goal resources, each with that status. The other two produce nothing, and the conversion completes.

**Acceptance Scenarios**:

1. **Given** a goal whose source status is exactly a code in the R4 goal lifecycle status set, **When** it is converted, **Then** the Goal is emitted with that status as its lifecycle status.
2. **Given** a goal entry with no status, **When** it is converted, **Then** no Goal is emitted for that entry, no default status is substituted, and the conversion completes.
3. **Given** a goal entry whose status is not in the R4 set, **When** it is converted, **Then** no Goal is emitted for that entry, the raw source value does not appear as a status anywhere, and the conversion completes.
4. **Given** a Goals Section with one goal that has a valid status and one that has a missing or unrecognised status, **When** it is converted, **Then** exactly one Goal is emitted, for the valid goal.
5. **Given** a goal skipped for a missing or unrecognised status, **When** any diagnostic is produced, **Then** it contains no patient or document data, including the unrecognised status value itself (FR-014).

---

### Edge Cases

- A goal entry with no priority relationship: the Goal has no priority and no empty placeholder is emitted.
- A goal entry with no target value or no due date: the Goal omits the missing parts and still converts.
- A goal entry with no status, or a status outside the R4 set: the entry is skipped, not emitted with an empty, raw or default status (FR-005a, FR-005b).
- A goal entry that has a single code with no translation versus one with a translation: the target measure is taken from the translation when present, otherwise from the code (existing behaviour, to be preserved).
- A Goals Section with several goals, some malformed: valid goals still convert. A malformed goal is skipped and must not break the whole document (FR-013).
- A malformed goal produces no Goal resource and no partial Goal. Any diagnostic about it never includes values from the source document.
- A document whose Goals Section appears more than once: only the first is used, matching how other sections behave.
- Error text produced by a failed conversion must not contain patient data (project rule).
- The same defect exists in the Instructions Section entry. It is out of scope here unless the developers decide otherwise (see Assumptions).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The standard CCD conversion MUST emit one Goal resource for each goal entry in the document's Goals Section (template id `2.16.840.1.113883.10.20.22.2.60`).
- **FR-002**: A document containing a Goals Section MUST convert without error. The existing malformed-tag defect that makes goal conversion fail MUST be corrected.
- **FR-003**: Each Goal MUST reference the patient of the document it came from.
- **FR-004**: Each Goal MUST carry its identifier(s), description, status, priority (when present), target measure, target quantity with unit (when present) and target due date (when present), as already defined by the existing goal mapping.
- **FR-005**: Every emitted Goal MUST contain a lifecycle status that is a code in the FHIR R4 `GoalLifecycleStatus` value set. `Goal.lifecycleStatus` has cardinality 1..1 in the official R4 definition (https://hl7.org/fhir/R4/goal.html), so it cannot be omitted. A source status that is not exactly a code in that set MUST NOT be copied through unchanged.
- **FR-005a**: A Goal MUST NOT be emitted when its source status is missing, empty, or not exactly a code in the R4 set. That goal entry is skipped, and the skip is handled as FR-013 and FR-014 describe.
- **FR-005b**: No default status and no source-to-target status mapping is defined by this spec. A missing status MUST NOT be replaced by a default. Any mapping (for example a C-CDA code that has a similar but differently named R4 code) requires a developer decision, a cited source and its own acceptance test before it is added, and until then such goals are skipped.
- **FR-006**: Documents with no Goals Section, or an empty one, MUST produce output identical to the output before this feature, including resource ids of all other resources.
- **FR-007**: Goal resource ids MUST be stable across repeated conversions of the same input, and the inputs used to generate ids for all existing resource types MUST NOT change.
- **FR-008**: Optional goal parts that are missing in the source MUST be omitted from the output, not emitted as empty structures.
- **FR-009**: The feature MUST NOT change the public conversion interface, the command line options, exit codes or dependencies.
- **FR-010**: A synthetic fixture with a Goals Section MUST be added, containing only invented data, and automated tests MUST cover FR-001 to FR-008.
- **FR-011**: The change MUST be recorded in the project change log as an output-shape change (new Goal resources in CCD bundles), and the incorrect statement in `docs/repo-analysis.md` about these sections being "present in other root templates" MUST be corrected.
- **FR-012**: Implementation MUST NOT begin until a developer has explicitly approved the output-shape change (Goal resources added to CCD bundles that have a Goals Section). The approval MUST be recorded in the spec or plan.
- **FR-013**: When one Goals Section entry is malformed, the other entries MUST still convert and the document conversion MUST still complete. A malformed entry means one that cannot produce a usable Goal, such as an entry with no observation content, one that cannot be rendered, or one whose status is missing or not in the R4 set (FR-005a). It is skipped, and no partial Goal is emitted. An entry that is only missing optional parts is not malformed (FR-008).
- **FR-014**: Any diagnostic about a skipped entry MUST NOT contain patient or document data (identifiers, names, dates, codes or display text). It MAY identify only the section template id and the entry's position in the section. No new output channel or logging is introduced without agreement.

### Key Entities

- **Goals Section**: The part of a C-CDA document that lists the patient's goals, identified by template id `2.16.840.1.113883.10.20.22.2.60`.
- **Goal entry**: One goal in that section, with an identifier, a description code, a status, an optional target date, an optional target value, and an optional priority.
- **Goal (FHIR)**: The output resource for one goal entry, with identifier, description, status, priority, one target and a reference to the patient.
- **Patient reference**: The link from each Goal to the document's patient resource in the same bundle.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Converting a synthetic document with N goals in its Goals Section, each with a valid R4 status, yields exactly N Goal resources, for N of 0, 1 and 3. Goals without a valid status are covered by SC-008.
- **SC-002**: 100% of emitted Goal resources reference the document's patient.
- **SC-003**: Converting the existing `TD-CCDA-01` fixture produces output with zero differences before and after the change.
- **SC-004**: The full existing automated test suite passes with the same number of passing tests as the pre-change baseline, plus the new goal tests.
- **SC-005**: 100% of emitted Goal resources have a lifecycle status that is a code in the FHIR R4 goal lifecycle status set, and 0 emitted Goals lack one.
- **SC-008**: In a synthetic document with 4 goals (1 valid status, 1 missing status, 1 unrecognised status, 1 more valid status), exactly 2 Goal resources are emitted, the conversion completes, and no emitted resource contains the unrecognised source value as a status.
- **SC-009**: 0 emitted Goals carry a default or substituted status that was not present in the source.
- **SC-007**: A document with 3 goal entries where 1 is malformed converts to exactly 2 Goal resources, completes without aborting, and any diagnostic produced contains no source data.
- **SC-006**: Converting the same goals document twice produces identical Goal ids in 100% of runs.

## Assumptions

- **Root templates in scope**: only the CCD root template includes the Goals Section. Consultation, discharge, history and physical, operative, procedure, progress, referral and transfer summary conversions are out of scope and unchanged. The discovery lists this as open question 1.
- **Output shape change is proposed, not approved**: adding Goal resources to CCD bundles that contain a Goals Section is the proposed functionality. The project rules (constitution principle II, `CLAUDE.md` section 9) require explicit developer approval before implementation. The request to write this spec is not that approval (FR-012, discovery open question 2).
- **Status handling**: the target is the FHIR R4 `GoalLifecycleStatus` set (see Clarifications for the evidence and its limits). Only exact code matches are carried through. A goal with any other status, or none, is skipped entirely, because `lifecycleStatus` is required (1..1) in R4 and no default is invented. Whether any other C-CDA code should be mapped to an R4 code is not decided here and needs developer input. The discovery flags `aborted` as a likely mismatch (assumption, not confirmed).
- **Malformed entry handling is a behavior requirement, not a design**: the repository's entry batching (`filters.py:338-340`) renders entries in a loop with no per-entry error isolation, and a bad rendering fails the whole document (seen in the discovery probe as a JSON5 parse error). Planning must show how FR-013 is met. If that needs a Python change, the change is an approval item and not assumed here.
- **Instructions Section defect**: the identical `{$` typo in the Instructions entry template remains unfixed and out of scope (discovery open question 4).
- **Health Concerns and Plan of Treatment**: out of scope, queued separately.
- **Test data**: synthetic only, under `data/synthetic/`. No data from `data/sample` is used.
- **No conformance claim**: output is not validated against FHIR or US Core by this project, and this feature does not add a validator.
- **Pre-change baseline**: the suite and a before/after diff are run during implementation. They were not run while writing this spec.
- **Reuse**: the existing goal templates already define the field mapping, so this feature fixes and connects them rather than redesigning the mapping.
