# Test Plan

Companion documents: [test-cases.md](test-cases.md), [test-data.md](test-data.md), [setup-guide.md](setup-guide.md), [api-guide.md](api-guide.md).

## 1. Purpose and scope

Verify that python-fhir-converter converts C-CDA documents and FHIR STU3 resources to FHIR R4 correctly, fails predictably on bad input, and behaves sensibly when fields are missing.

In scope:
- CLI (`python -m fhir_converter`): single file, directory, template-dir, error flags, exit codes.
- Library API: `CcdaRenderer`, `Stu3FhirRenderer`, `render_to_dir`, `render_files_to_dir`.
- Mapping behavior of the packaged `CCD` (C-CDA) and `Patient` / `Immunization` (STU3) templates, plus code mapping through `ValueSet.json`.
- Date and time conversion, default and missing values, invalid input.
- Regression using the existing pytest suite (475 tests).

Out of scope:
- Clinical correctness of every one of the ~1,090 templates. Only the templates named above are exercised here. Other C-CDA document templates and the other ~50 STU3 resource templates are **not covered** and need a follow-up plan.
- FHIR R4 profile or schema validation (no validator exists in the repo; tool choice is **UNKNOWN**).
- Performance, load and security testing. The README has a benchmark script (`scripts/benchmark.py`) that was not run.
- Non-Windows platforms, Python versions other than those listed in section 4.

## 2. Test approach

| Level | Method | Tooling |
|---|---|---|
| Regression | Existing automated tests | `pytest` (7 modules, 475 tests) |
| Functional | Manual or scripted CLI and API runs against synthetic fixtures | PowerShell, Python |
| Negative | Invalid, empty and malformed inputs, bad arguments | PowerShell, Python |
| Data/mapping | Compare selected output fields to expected values | Python one-liners or JSON diff |

Expected results in [test-cases.md](test-cases.md) were derived from the source code and checked against exploratory runs on 2026-11-04. Those runs were not formal test execution, so every new case is recorded as **Not Executed**. Only TC-REG-01 has a recorded result.

Test IDs: `TC-<AREA>-<nn>` where AREA is CLI, API, MAP, DT, INV, UTL or REG. Test data IDs: `TD-...` in [test-data.md](test-data.md).

Priorities: **P1** core conversion or data loss/corruption risk. **P2** important error handling or mapping. **P3** edge cases and observations.

Statuses: Not Executed, Pass, Fail, Blocked.

## 3. Features and risk

| Feature | Source | Risk | Cases |
|---|---|---|---|
| C-CDA to FHIR Bundle | `CcdaRenderer`, `templates/ccda/CCD.liquid` | High: main use case | TC-CLI-01, TC-API-01..04, TC-MAP-* |
| STU3 to R4 | `Stu3FhirRenderer`, `templates/stu3` | Medium | TC-CLI-02, TC-INV-06..09 |
| Date handling | `hl7.hl7_to_fhir_dtm`, `format_as_date_time`, `date` | High: silent precision loss | TC-DT-* |
| Code mapping | `get_property`, `ValueSet.json` | Medium | TC-MAP-01..03, TC-UTL-02 |
| Error handling | `RenderingError`, `onerror`, CLI exit codes | High: exit code 0 after failures | TC-CLI-07..15, TC-INV-* |
| Directory conversion | `render_files_to_dir`, `is_ccda_file` | Medium | TC-CLI-03, 04, 12..15 |
| Custom templates | `--template-dir`, `make_environment` | Low | TC-CLI-05, TC-API-06 |
| Deterministic IDs | `generate_uuid`, `Utils/_GenerateId` | Medium | TC-API-04, TC-UTL-01 |

## 4. Environment

| Item | Value |
|---|---|
| OS | Windows 11 |
| Shell | PowerShell 5.1 |
| Python | 3.14 (verified). 3.11 and 3.12 recommended for pinned dependencies, not tested |
| Install | venv + pip per [setup-guide.md](setup-guide.md) |
| Dependency versions | Newer than `pyproject.toml` pins except python-liquid 1.12.1 |
| Working directory | repo root `fhir-converter\` |
| Output folder | `$env:TEMP\fhir-test-out` (use a fresh folder per case) |

## 5. Test data

Synthetic only, in `data\synthetic\` (24 files): 12 C-CDA variants, 5 STU3 resources, 6 invalid files, 1 custom template. Details in [test-data.md](test-data.md). The pre-existing `data\sample` files are not used by the plan because their provenance is **UNKNOWN** and they may contain real patient data.

## 6. Entry and exit criteria

Entry:
- Environment set up and `pytest -q` passes (TC-REG-01).
- Fixtures present in `data\synthetic`.

Exit:
- All P1 and P2 cases executed. All P1 pass. Any P2 failure has a defect or an accepted risk note.
- No open defect that causes wrong clinical data to be emitted silently.
- Observations in section 7 triaged (fix, document, or accept).

## 7. Observations found during analysis

These came from reading the code and exploratory runs. Each is covered by a test case. **Execution result (2026-11-04): OBS-01 to OBS-09 were reproduced** by the scripted run, so they are real behaviors of the current code. None is a confirmed defect until the team agrees on the expected behavior. OBS-10 is a packaging question. OBS-11 and OBS-12 were found during execution. Full results: [test-execution-report.md](test-execution-report.md).

| ID | Observation | Case |
|---|---|---|
| OBS-01 | `--continue_on_error` run with one failed file prints `RENDER SUCCESS` and exits 0 | TC-CLI-13 |
| OBS-02 | `--from-dir` with `--source-format STU3` converts nothing, exits 0 | TC-CLI-15 |
| OBS-03 | XML that is not a C-CDA still produces a Bundle with empty references (`Patient/`) | TC-INV-03 |
| OBS-04 | `Composition.attester.party.reference` is `Practitioner/` (no id) | TC-MAP-10 |
| OBS-05 | STU3 Immunization output keeps `wasNotGiven` (an STU3 field) and adds `.000` milliseconds | TC-DT-07, TC-INV-09 |
| OBS-06 | Source datetime with no time zone is reduced to a date, losing the time | TC-DT-02 |
| OBS-07 | Unmapped gender code gives no `gender` at all (no `unknown` default) | TC-MAP-03 |
| OBS-08 | Missing `recordTarget` leaves `Composition.subject` as an invalid reference | TC-MAP-07 |
| OBS-09 | `DocumentReference.date` uses the current time, so outputs are not byte-identical across runs | TC-API-04 |
| OBS-10 | CLI requires Python 3.11+ but packaging claims 3.8 | Setup guide only (not tested on 3.8 to 3.10: no such interpreter is installed) |
| OBS-11 | The `gzip` filter and the `DocumentReference` attachment use zlib compression, not gzip format. The filter docstring says zlib, so this is by design, but the name is misleading and a consumer expecting gzip will fail to decode | Output validation |
| OBS-12 | `pip check` reports 8 conflicts between `pyproject.toml` pins and the versions installed for Python 3.14 | Environment check |

## 8. Risks and assumptions

- Expected values for mapping are what the current code does, not an external specification. A reviewer with FHIR and C-CDA knowledge should confirm that the behavior in OBS-04 to OBS-08 is acceptable.
- The newer dependency versions may differ from the pinned ones. Repeat TC-REG-01 and the P1 cases on Python 3.12 with the pinned set (**not done**).
- Schedule, owners and defect tracker are **UNKNOWN**.

## 9. Deliverables

[test-plan.md](test-plan.md), [test-cases.md](test-cases.md) with results column, [test-data.md](test-data.md), and a defect list (none created).

This plan covers software behavior only. It is not a clinical safety assessment or compliance attestation. Any use of the converter for real patient data needs separate clinical, privacy and security review.
