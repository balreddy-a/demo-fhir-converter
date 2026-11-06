# Test Cases

Plan: [test-plan.md](test-plan.md). Data: [test-data.md](test-data.md). Priorities: P1 core, P2 important, P3 edge.

**Status rule:** a case is **Not Executed** until it is run and the result recorded. All 53 cases were executed on 2026-11-04 (details in [test-execution-report.md](test-execution-report.md)): 52 by a script that drives the real CLI and Python API, and TC-REG-01 by `pytest`. Result: 53 Pass, 0 Fail.

How to read "Pass": the actual behavior matched the expected result written here. Those expected results were written after exploratory runs, so the suite records current behavior and is a regression baseline, not an independent specification check. Cases marked "current behavior confirmed, acceptability open" confirm an observation (OBS-nn in the test plan) whose correctness the team still has to decide.

Basis: **Code** = derived from source only. **Code+Run** = derived from source and matched by an exploratory run.

## 0. Common setup

Run from the repo root in PowerShell. Python 3.11+ venv per [setup-guide.md](setup-guide.md).

```powershell
$PY  = ".\.venv\Scripts\python.exe"
$OUT = "$env:TEMP\fhir-test-out"
$D   = ".\data\synthetic"
function Reset-Out { if (Test-Path $OUT) { Remove-Item $OUT -Recurse -Force } }
function Convert-Ccda($f, [string[]]$more) {
  & $PY -m fhir_converter --from-file "$D\ccda\$f" --to-dir $OUT --template-name CCD --source-format CCDA @more
}
function Convert-Stu3($f, $tpl) {
  & $PY -m fhir_converter --from-file "$D\stu3\$f" --to-dir $OUT --template-name $tpl --source-format STU3
}
function Get-Res($json, $type) {
  (Get-Content $json -Raw | ConvertFrom-Json).entry.resource | Where-Object resourceType -eq $type
}
```

Call `Reset-Out` before each case. "Exit code" means `$LASTEXITCODE`, read immediately after the command and before any pipe.

## 1. CLI functional

| ID | Title | Pri | Basis | Status |
|---|---|---|---|---|
| TC-CLI-01 | Convert one C-CDA file | P1 | Code+Run | Pass, 2026-11-04 |
| TC-CLI-02 | Convert one STU3 file | P1 | Code+Run | Pass, 2026-11-04 |
| TC-CLI-03 | Convert a directory, mirrored output | P2 | Code+Run | Pass, 2026-11-04 |
| TC-CLI-04 | Directory with `--flatten_to_dir` | P2 | Code+Run | Pass, 2026-11-04 |
| TC-CLI-05 | Custom template via `--template-dir` | P2 | Code+Run | Pass, 2026-11-04 |
| TC-CLI-06 | Output directory is created when missing | P3 | Code+Run | Pass, 2026-11-04 |

**TC-CLI-01** Preconditions: venv installed, `$OUT` absent, TD-CCDA-01.
Steps: `Reset-Out; Convert-Ccda TD-CCDA-01_valid_basic.ccda`. Then `Get-Res "$OUT\TD-CCDA-01_valid_basic.json" Patient`.
Expected: exit code 0 and `RENDER SUCCESS` banner. File `TD-CCDA-01_valid_basic.json` exists. Bundle `type` is `batch`. Resource types present: Composition, Organization, Practitioner, Patient, DocumentReference. Patient: gender `female`, birthDate `1980-01-01`, name Alex Example, address use `home`.

**TC-CLI-02** Preconditions: TD-STU3-01.
Steps: `Reset-Out; Convert-Stu3 TD-STU3-01_patient_valid.json Patient`.
Expected: exit code 0. `TD-STU3-01_patient_valid.json` in `$OUT` equals the input content: id `syn-p1`, gender `female`, birthDate `1980-01-01`, name Example/Alex.

**TC-CLI-03** Preconditions: temp folder `$IN` with `a\x.ccda` (copy of TD-CCDA-01) and `b\y.xml` (copy of TD-CCDA-02).
Steps: `Reset-Out; & $PY -m fhir_converter --from-dir $IN --to-dir $OUT --template-name CCD --source-format CCDA`; list `$OUT` recursively.
Expected: exit code 0. Outputs `$OUT\a\x.json` and `$OUT\b\y.json`. Both `.ccda` and `.xml` are read.

**TC-CLI-04** Preconditions and steps as TC-CLI-03, adding `--flatten_to_dir`.
Expected: exit code 0. Outputs `$OUT\x.json` and `$OUT\y.json`, no sub-folders.

**TC-CLI-05** Preconditions: `data\synthetic\templates\Mini.liquid` (TD-TPL-01), TD-CCDA-01.
Steps: `& $PY -m fhir_converter --from-file "$D\ccda\TD-CCDA-01_valid_basic.ccda" --to-dir $OUT --template-name Mini --template-dir "$D\templates" --source-format CCDA`.
Expected: exit code 0. Output is `{"resourceType":"Bundle","type":"collection","entry":[{"resource":{"resourceType":"Basic","id":"TD-01"}}]}`.

**TC-CLI-06** Preconditions: `$OUT` absent. Steps: as TC-CLI-01.
Expected: `$OUT` is created and contains the output file.

## 2. CLI negative

| ID | Title | Pri | Basis | Status |
|---|---|---|---|---|
| TC-CLI-07 | No arguments | P2 | Code+Run | Pass, 2026-11-04 |
| TC-CLI-08 | Neither `--from-file` nor `--from-dir` | P2 | Code+Run | Pass, 2026-11-04 |
| TC-CLI-09 | Invalid `--source-format` | P2 | Code+Run | Pass, 2026-11-04 |
| TC-CLI-10 | Input file does not exist | P1 | Code+Run | Pass, 2026-11-04 |
| TC-CLI-11 | Unknown `--template-name` | P1 | Code+Run | Pass, 2026-11-04 |
| TC-CLI-12 | Directory run stops at first failure | P2 | Code+Run | Pass, 2026-11-04 |
| TC-CLI-13 | `--continue_on_error` reports success despite failure | P1 | Code+Run | Pass (current behavior confirmed, acceptability open), 2026-11-04 |
| TC-CLI-14 | Directory run skips non-CDA extensions | P3 | Code+Run | Pass, 2026-11-04 |
| TC-CLI-15 | STU3 directory run converts nothing | P2 | Code+Run | Pass (current behavior confirmed, acceptability open), 2026-11-04 |

**TC-CLI-07** Steps: `& $PY -m fhir_converter`. Expected: usage text, exit code 2, no `RENDER` banner.

**TC-CLI-08** Steps: `& $PY -m fhir_converter --to-dir $OUT --template-name CCD --source-format CCDA`. Expected: message `Either --from-file or --from-dir must be specified.`, exit code 2.

**TC-CLI-09** Steps: `--source-format HL7` with a valid file. Expected: `invalid RenderSourceFormat value: 'HL7'`, exit code 2.

**TC-CLI-10** Preconditions: `$OUT` absent. Steps: `--from-file .\nofile.ccda ...`. Expected: traceback ending `RenderingError: Failed to render <path>`, `RENDER FAILURE` banner, exit code 1. `$OUT` is not left behind.

**TC-CLI-11** Preconditions: `$OUT` absent, TD-CCDA-01. Steps: `--template-name Nope`. Expected: `RenderingError: Failed to render FHIR` with cause `TemplateNotFound`, `RENDER FAILURE`, exit code 1. No output file, `$OUT` not left behind.

**TC-CLI-12** Preconditions: all TD-CCDA files. Steps: `--from-dir "$D\ccda"` without `--continue_on_error`. Expected: exit code 1, `RENDER FAILURE`. Files for TD-CCDA-01 to 07 exist in `$OUT` (alphabetical order, stopped at TD-CCDA-08). TD-CCDA-09 to 12 are not written.

**TC-CLI-13** Preconditions: all TD-CCDA files. Steps: same as TC-CLI-12 plus `--continue_on_error`; capture output with `*> $log`, read `$LASTEXITCODE` first.
Expected (current behavior): traceback for TD-CCDA-08 (`Malformed HL7 datetime not-a-date`) is printed, 11 `.json` files written (all except TD-CCDA-08), banner says `RENDER SUCCESS`, exit code 0.
Note: this is OBS-01. If the team decides a partial failure should return non-zero, mark Fail and raise a defect.

**TC-CLI-14** Preconditions: `data\synthetic\invalid` (contains `.xml`, `.json` and `.txt`). Steps: `--from-dir ... --template-name CCD --source-format CCDA --continue_on_error`. Expected: `TD-INV-06_ignore.txt` and `.json` files are not attempted. TD-INV-01 and 02 print tracebacks. `TD-INV-03_not_cda.json` is written (see TC-INV-03).

**TC-CLI-15** Preconditions: `data\synthetic\stu3`. Steps: `--from-dir ... --template-name Patient --source-format STU3`. Expected (current behavior): `RENDER SUCCESS`, exit code 0, no output files (OBS-02).

## 3. Library API

| ID | Title | Pri | Basis | Status |
|---|---|---|---|---|
| TC-API-01 | `render_fhir_string` returns FHIR JSON text | P1 | Code+Run | Pass, 2026-11-04 |
| TC-API-02 | `render_to_fhir` returns a dict | P2 | Code+Run | Pass, 2026-11-04 |
| TC-API-03 | `render_fhir` writes to a stream | P2 | Code | Pass, 2026-11-04 |
| TC-API-04 | Deterministic resource IDs | P2 | Code+Run | Pass, 2026-11-04 |
| TC-API-05 | `render_narrative` option | P3 | Code+Run | Pass, 2026-11-04 |
| TC-API-06 | Custom environment and template | P3 | Code+Run | Pass, 2026-11-04 |
| TC-API-07 | Errors are wrapped in `RenderingError` | P1 | Code+Run | Pass, 2026-11-04 |

Steps for all use `& $PY -c "<python>"` from the repo root.

**TC-API-01** `CcdaRenderer().render_fhir_string("CCD", open(TD-CCDA-01, encoding="utf-8"))`. Expected: a string that `json.loads` parses to a Bundle with 5 entries.

**TC-API-02** `CcdaRenderer().render_to_fhir("CCD", text)`. Expected: a `dict`, `["type"] == "batch"`, each entry has `resource` and `request.method == "PUT"`.

**TC-API-03** `with StringIO() as b: Stu3FhirRenderer().render_fhir("Patient", text, b)`. Expected: `b.getvalue()` contains the Patient JSON. Return value is `None`.

**TC-API-04** Render TD-CCDA-01 twice. Expected: the Patient id is identical across runs (`05cadb76-f5ab-1d6a-fae5-8539f75dd453`). Practitioner, Organization, Composition and DocumentReference ids also matched. `DocumentReference.date` follows the clock at one-second resolution (OBS-09): identical for two renders in the same second, different (`...14:47:01Z` vs `...14:47:03Z`) with a 2 second gap. The date check across a gap was a separate manual run, not part of the script.

**TC-API-05** `CcdaRenderer(template_globals={"render_narrative": True})`. Expected: `Composition.section[0].text.div` is `<div xmlns="http://www.w3.org/1999/xhtml">No known allergies</div>` and `_status.extension[0].valueCode` is `unknown`.

**TC-API-06** Environment from `make_environment(FileExtensionLoader(<folder with Mini.liquid>), additional_loaders=[CcdaRenderer.defaults()["loader"]])`. Expected: `CcdaRenderer(env).render_fhir_string("Mini", text)` returns the Basic-resource bundle with id `TD-01`.

**TC-API-07** Render `""` as C-CDA, `"{bad"` as STU3, and template `"Nope"`. Expected: each raises `RenderingError` with message `Failed to render FHIR`, and `__cause__` types `XMLSyntaxError`, `Json5EOF`, `TemplateNotFound` respectively.

## 4. Mapping, missing fields and defaults

Method for all: `Reset-Out; Convert-Ccda <file>`, then inspect with `Get-Res`.

| ID | Data | Pri | Basis | Status |
|---|---|---|---|---|
| TC-MAP-01 | TD-CCDA-01 | P1 | Code+Run | Pass, 2026-11-04 |
| TC-MAP-02 | TD-CCDA-03 | P2 | Code+Run | Pass, 2026-11-04 |
| TC-MAP-03 | TD-CCDA-04 | P2 | Code+Run | Pass (current behavior confirmed, acceptability open), 2026-11-04 |
| TC-MAP-04 | TD-CCDA-02 | P2 | Code+Run | Pass, 2026-11-04 |
| TC-MAP-05 | TD-CCDA-11 | P2 | Code+Run | Pass, 2026-11-04 |
| TC-MAP-06 | TD-CCDA-12 | P3 | Code+Run | Pass, 2026-11-04 |
| TC-MAP-07 | TD-CCDA-09 | P1 | Code+Run | Pass (current behavior confirmed, acceptability open), 2026-11-04 |
| TC-MAP-08 | TD-CCDA-10 | P3 | Code+Run | Pass, 2026-11-04 |
| TC-MAP-09 | TD-CCDA-01 | P2 | Code+Run | Pass, 2026-11-04 |
| TC-MAP-10 | TD-CCDA-01 | P2 | Code+Run | Pass (current behavior confirmed, acceptability open), 2026-11-04 |

| ID | Scenario | Expected |
|---|---|---|
| TC-MAP-01 | Gender code `F` is mapped | Patient `gender` is `female` (via `ValueSet/Gender`) |
| TC-MAP-02 | Gender element missing | Conversion succeeds. Patient has no `gender` key |
| TC-MAP-03 | Gender code `ZZ` not in mapping | Conversion succeeds. Patient has no `gender` key (no `unknown` default, OBS-07). Confirm this is acceptable |
| TC-MAP-04 | `birthTime` missing | Conversion succeeds. Patient has no `birthDate` |
| TC-MAP-05 | Patient name is `nullFlavor="UNK"` | Conversion succeeds. Patient has no `name`. Patient still has identifier and address |
| TC-MAP-06 | Address missing | Conversion succeeds. Patient has no `address` |
| TC-MAP-07 | `recordTarget` missing | Conversion succeeds (no error). No Patient entry. Composition, Organization, Practitioner, DocumentReference still written. Check `Composition.subject.reference`: expected to be an empty-id value such as `Patient/` (OBS-08, seen on a non-CDA input, not confirmed for this file) |
| TC-MAP-08 | No sections in the body | Conversion succeeds. Composition has no `section` items |
| TC-MAP-09 | Address use `HP` | Patient address `use` is `home` |
| TC-MAP-10 | Attester reference | `Composition.attester[0].party.reference` is `Practitioner/` with no id (OBS-04). Mark Fail if the team requires a populated reference |

## 5. Date and time

| ID | Title | Pri | Basis | Status |
|---|---|---|---|---|
| TC-DT-01 | Document time with time zone | P1 | Code+Run | Pass, 2026-11-04 |
| TC-DT-02 | Document time without time zone | P1 | Code+Run | Pass (current behavior confirmed, acceptability open), 2026-11-04 |
| TC-DT-03 | Date only | P2 | Code+Run | Pass, 2026-11-04 |
| TC-DT-04 | Year only | P3 | Code+Run | Pass, 2026-11-04 |
| TC-DT-05 | Malformed date | P1 | Code+Run | Pass, 2026-11-04 |
| TC-DT-06 | `hl7_to_fhir_dtm` unit values | P1 | Code+Run | Pass, 2026-11-04 |
| TC-DT-07 | STU3 date to `occurrenceDateTime` | P2 | Code+Run | Pass (current behavior confirmed, acceptability open), 2026-11-04 |
| TC-DT-08 | Empty and non-date input to `date` filter | P3 | Code+Run | Pass, 2026-11-04 |

**TC-DT-01** Data TD-CCDA-01 (`20240115103000-0500`). Expected `Composition.date` = `2024-01-15T10:30:00-05:00`.

**TC-DT-02** Data TD-CCDA-05 (`20240115103000`). Expected `Composition.date` = `2024-01-15`. The time is dropped when no zone is given (OBS-06; the CHANGES.md for version 0.1.0 describes this as intended to keep FHIR validity).

**TC-DT-03** Data TD-CCDA-06 (`20240115`). Expected `2024-01-15`.

**TC-DT-04** Data TD-CCDA-07 (`2024`). Expected `2024`.

**TC-DT-05** Data TD-CCDA-08 (`not-a-date`), single file. Expected: exit code 1, `RENDER FAILURE`, cause text `filter 'format_as_date_time': unexpected error: Malformed HL7 datetime not-a-date`. No output file.

**TC-DT-06** Call `hl7_to_fhir_dtm` for the TD-DTM-01 and TD-DTM-02 values. Expected: `2024`, `2024-01`, `2024-01-15`, `2024-01-15`, `2024-01-15T10:30:00-05:00`, `2024-01-15T10:30:00+05:30`, `2024-01-15`. Values `abc`, `""`, `2024011` raise `ValueError` "Malformed HL7 datetime".

**TC-DT-07** Data TD-STU3-05 with template `Immunization`. Expected `occurrenceDateTime` = `2024-01-15T10:30:00.000-05:00`; the original `date` key is absent. Milliseconds are added (OBS-05).

**TC-DT-08** Render `{{ v | date: 'yyyy-MM-dd' }}` with `v` = `20240115`, `20240115103000-0500`, `""`, `abc`. Expected: `2024-01-15`, `2024-01-15`, empty string, `abc` (non-date text is returned unchanged, no error).

## 6. Invalid input

| ID | Title | Data | Pri | Basis | Status |
|---|---|---|---|---|---|
| TC-INV-01 | Malformed XML | TD-INV-01 | P1 | Code+Run | Pass, 2026-11-04 |
| TC-INV-02 | Empty XML | TD-INV-02 | P1 | Code+Run | Pass, 2026-11-04 |
| TC-INV-03 | Well-formed XML that is not a C-CDA | TD-INV-03 | P1 | Code+Run | Pass (current behavior confirmed, acceptability open), 2026-11-04 |
| TC-INV-04 | Malformed JSON | TD-INV-04 | P1 | Code+Run | Pass, 2026-11-04 |
| TC-INV-05 | Empty JSON file | TD-INV-05 | P2 | Code+Run | Pass, 2026-11-04 |
| TC-INV-06 | STU3 empty object | inline `{}` | P3 | Code+Run | Pass, 2026-11-04 |
| TC-INV-07 | STU3 missing `resourceType` | TD-STU3-04 | P2 | Code+Run | Pass, 2026-11-04 |
| TC-INV-08 | STU3 empty fields | TD-STU3-03 | P2 | Code+Run | Pass, 2026-11-04 |
| TC-INV-09 | STU3-only field carried into output | TD-STU3-05 | P3 | Code+Run | Pass (current behavior confirmed, acceptability open), 2026-11-04 |

| ID | Steps | Expected |
|---|---|---|
| TC-INV-01 | Convert with `CCD`, `CCDA` | Exit code 1. Cause `XMLSyntaxError` (tag mismatch). No output file |
| TC-INV-02 | Same | Exit code 1. Cause `XMLSyntaxError: Document is empty`. No output file |
| TC-INV-03 | Same | No error, exit code 0, but output is a Bundle with a Composition whose references are empty (`Patient/`, `Practitioner/`). Current behavior (OBS-03). Decide whether this should be an error |
| TC-INV-04 | Convert with `Patient`, `STU3` | Exit code 1. Cause `Json5EOF` (unclosed object). No output file |
| TC-INV-05 | Same | Exit code 1. Cause `Json5EOF: No JSON data found`. No output file |
| TC-INV-06 | `render_fhir_string("Patient", "{}")` | Returns `{}`. No error |
| TC-INV-07 | `Patient` template | Exit code 0. Output `{"id":"syn-p4","gender":"male"}`. No `resourceType` is added and no error is raised |
| TC-INV-08 | `Patient` template | Output has only `resourceType` and `id`. The empty `name`, empty `gender` and null `birthDate` are removed |
| TC-INV-09 | `Immunization` template | Output retains `"wasNotGiven": false`, a field that does not exist in FHIR R4 (OBS-05). Mark Fail if R4 conformance is required |

## 7. Utilities and filters

| ID | Title | Pri | Basis | Status |
|---|---|---|---|---|
| TC-UTL-01 | `generate_uuid` | P2 | Code+Run | Pass, 2026-11-04 |
| TC-UTL-02 | `get_property` fallbacks | P2 | Code+Run | Pass, 2026-11-04 |
| TC-UTL-03 | Template name resolution with `_` prefix | P3 | Code | Pass, 2026-11-04 |

**TC-UTL-01** Render `{{ 'a' | generate_uuid }}` twice and `{{ '' | generate_uuid }}`. Expected: both runs return `ca978112-ca1b-bdca-fac2-31b39a23dc4d`; the blank input returns an empty string.

**TC-UTL-02** `get_property` with `F`/`ValueSet/Gender`, `ZZ`/`ValueSet/Gender`, `HP`/`ValueSet/AddressUse`, and property `system`. Expected: `female`, empty, `home`, empty. Unmapped codes return an empty string for the `code` property as well, because the mapping has a `__default__` entry (the docstring in `filters.py` describes a fallback to the original code; behavior differs here. Confirm intent).

**TC-UTL-03** Load `Section/Problem` through the default environment. Expected: resolved to `Section/_Problem.liquid`. A name ending `.json` (for example `ValueSet/ValueSet.json`) is not prefixed. Covered indirectly by `tests/test_loaders.py`.
Actual: resolved names were `Section\_Problem` and `ValueSet\ValueSet.json` (Windows path separator from `Path`), and the template source loaded. The check compared with separators normalized. Whether the backslash causes problems for non-filesystem loaders is not tested.

## 8. Regression

| ID | Title | Pri | Basis | Status |
|---|---|---|---|---|
| TC-REG-01 | Existing pytest suite | P1 | Run | **Pass** |

**TC-REG-01** Preconditions: venv installed. Steps: `& $PY -m pytest -q`. Expected: all tests pass (475 collected).
Result recorded: 475 passed, 0 failed, 0 skipped. Pytest reported 7.22 seconds (9.6 seconds wall) on 2026-11-04, Windows 11, Python 3.14.10, dependency versions as listed in the setup guide. Re-run after any dependency or template change.

## 9. Summary

| Group | Cases | Not Executed | Pass | Fail |
|---|---|---|---|---|
| CLI functional | 6 | 0 | 6 | 0 |
| CLI negative | 9 | 0 | 9 | 0 |
| API | 7 | 0 | 7 | 0 |
| Mapping | 10 | 0 | 10 | 0 |
| Date/time | 8 | 0 | 8 | 0 |
| Invalid input | 9 | 0 | 9 | 0 |
| Utilities | 3 | 0 | 3 | 0 |
| Regression | 1 | 0 | 1 | 0 |
| **Total** | **53** | **0** | **53** | **0** |

Nine of the 53 passes (TC-CLI-13, TC-CLI-15, TC-MAP-03, TC-MAP-07, TC-MAP-10, TC-DT-02, TC-DT-07, TC-INV-03, TC-INV-09) confirm behavior that the team may decide is a defect. See the observations table in [test-plan.md](test-plan.md).

Update this table when cases are executed. Coverage is limited to the `CCD` C-CDA template and the STU3 `Patient` and `Immunization` templates. These cases do not assess clinical accuracy.
