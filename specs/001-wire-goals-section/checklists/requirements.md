# Specification Quality Checklist: Convert C-CDA Goals Section to FHIR Goal

**Purpose**: Validate specification completeness and quality before proceeding to planning
**Created**: 2026-11-05
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs)
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

## Requirement Completeness

- [x] No [NEEDS CLARIFICATION] markers remain
- [x] Requirements are testable and unambiguous
- [x] Success criteria are measurable
- [x] Success criteria are technology-agnostic (no implementation details)
- [x] All acceptance scenarios are defined
- [x] Edge cases are identified
- [x] Scope is clearly bounded
- [x] Dependencies and assumptions identified

## Feature Readiness

- [x] All functional requirements have clear acceptance criteria
- [x] User scenarios cover primary flows
- [x] Feature meets measurable outcomes defined in Success Criteria
- [x] No implementation details leak into specification

## Notes

- The spec names the C-CDA template id, the CCD conversion, the change log and one repo doc because the feature is defined in those terms. No code, file paths of templates, or tag syntax appear.
- Three scope defaults are recorded under Assumptions rather than as clarification markers: CCD only, status mapping table decided at planning, Instructions typo deferred. Revisit with `/speckit-clarify` if the developers want a different scope.
- The spec checklist is a self-review, not an attestation of correctness. Nothing was run (tests, conversion, validator) while writing the spec.

## Clarification Review (2026-11-05)

- Checklist items re-evaluated against the updated spec: all 16 still pass, no state changes. This is a self-review of the document only.
- FHIR version: R4, evidenced by `README.md:40-41`, `docs/architecture.md:7`, `docs/repo-analysis.md:13` and the `lifecycleStatus` element in `Resource/_Goal.liquid`. The minor version is not stated in the repository.
- Uncertainty carried forward: the R4 `GoalLifecycleStatus` codes and the C-CDA goal status codes are not stored in the repository and were not checked against the official sources. Planning must confirm them. No mapping table was invented.
- Approval: the spec now states the output-shape change is proposed and needs explicit developer approval (FR-012). The earlier "approved by this request" wording was removed.
- Malformed entries: FR-013, FR-014 and SC-007 added. Feasibility is open because entry batching has no per-entry error isolation (`filters.py:338-340`). Planning must resolve it, and any Python change needs approval.
- Scope: CCD only. Other document types and the Instructions typo are confirmed out of scope.
- Not run: tests, conversions, validators. `check-prerequisites.ps1` was blocked by the PowerShell execution policy, so the feature directory was taken from `.specify/feature.json`.

## Status Cardinality Review (2026-11-05)

- Requester finding applied: `Goal.lifecycleStatus` is 1..1 in the HL7 FHIR R4 Goal definition (https://hl7.org/fhir/R4/goal.html), so an emitted Goal cannot omit it. The spec cites that page. I did not fetch it in this session, so the cardinality rests on the requester's statement and must be confirmed against the page during planning.
- Spec changes: FR-005 rewritten, FR-005a and FR-005b added (skip goals with a missing or unrecognised status, no defaults, no invented mappings), FR-013 extended, user story 3 and its five acceptance scenarios rewritten, SC-001 narrowed, SC-005 tightened, SC-008 and SC-009 added. The earlier "omit the status" wording was removed.
- Checklist items re-evaluated: all 16 still pass, no state changes. Self-review of the document only.
- Consequence to plan for: a goal with no mappable status is dropped, so source goal data can be lost without an error. Planning must decide how that is made visible, within the FR-014 privacy limit and without a new output channel unless approved.
- Not run: tests, conversions, validators.
