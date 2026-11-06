# Contract: Output of the CCD conversion for Goals

## Public interface: unchanged

`render_fhir_string`, `render_fhir`, `render_to_fhir`, `make_environment`, `render_to_dir`, `render_files_to_dir`, the CLI options and exit codes do not change (FR-009). The template name `CCD` stays the same. No new dependency.

## Output contract (changes only for CCD bundles that have a Goals Section)

1. For each Goals Section entry whose `statusCode.code` is exactly a code in the R4 `GoalLifecycleStatus` set, the bundle contains one `Goal` entry with that `lifecycleStatus`, a `subject.reference` equal to the document's Patient reference, and the already-mapped optional parts.
2. For each entry whose status is missing, empty or not in the set, the bundle contains no `Goal` for it, and no entry with that Goal's id.
3. No `Goal` entry lacks `lifecycleStatus` or `subject`.
4. Goal ids are stable across repeated runs and use the existing id expression.
5. A document with no Goals Section, or an empty one, produces output identical to the pre-change output.
6. Other root templates (consultation, discharge, history and physical, operative, procedure, progress, referral, transfer) are unchanged.

## Diagnostics contract

Nothing is emitted by default. Any diagnostic that is later approved may contain only the section template id and the entry position, never identifiers, names, dates, codes, display text, status values or exception text.

## Non-claims

The project runs no FHIR validator. This contract states specific structural rules, not conformance to R4 or any profile.
