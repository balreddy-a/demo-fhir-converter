# Test Execution Report: Baseline Validation

| Item | Value |
|---|---|
| Date | 2026-11-04, about 20:14 to 20:20 local time |
| Scope | Environment check, existing pytest suite, sample conversions, output validation, execution of the 53 cases in [test-cases.md](test-cases.md) |
| Result | **Baseline passes.** 475 of 475 existing tests passed. 53 of 53 documented cases passed. 0 product failures. 12 observations recorded (OBS-01 to OBS-12) |
| Code under test | Unmodified. The newest timestamp under `fhir_converter\`, `tests\` and `scripts\` is 18:01:57, before the first command of this work (about 18:08) |
| Not covered | See section 8. Notably: no lint, type check or coverage run, no FHIR schema validation, no pinned-dependency run, only synthetic data |

Every number below comes from a command run in this session. Raw outputs were captured in the session scratchpad (`run_cases.py` results, `validation_output.txt`, `junit.xml`) and are not part of the repo.

## 1. Environment

| Item | Actual |
|---|---|
| OS | Windows 11 Enterprise |
| Shell | PowerShell 5.1.26100.9711 |
| Python | 3.14.10 (venv at `.venv`) |
| pip | 26.3 |
| Package install | `pip install -e . --no-deps` after installing dependencies separately (see below) |
| Import check | `fhir_converter.__main__` and `fhir_converter.renderers` import without error |
| Poetry, git, flake8, mypy | Not on PATH. flake8 and mypy are not in the venv |

### Dependencies

`python -m pip check` reports **8 conflicts**: the installed versions differ from the exact pins in `pyproject.toml`. Python 3.14 cannot build the pinned `lxml 5.3.0` and `pyjson5 1.6.5` (see [troubleshooting.md](troubleshooting.md)), so newer versions were installed.

| Package | Pinned | Installed | Match |
|---|---|---|---|
| python-liquid | 1.12.1 | 1.12.1 | yes |
| xmltodict | 0.13.0 | 1.0.4 | no |
| pyjson5 | 1.6.5 | 2.0.1 | no |
| frozendict | 2.3.10 | 2.5.1 | no |
| importlib_resources | 6.1.1 | 7.1.0 | no |
| psutil | 5.9.7 | 7.2.2 | no |
| isodate | 0.6.1 | 0.7.2 | no |
| lxml | 5.3.0 | 6.1.3 | no |
| typing-extensions | 4.12.2 | 4.16.0 | no |
| pytest | 7.4.4 | 9.2.0 | no |
| pytest-mock | 3.12.0 | 3.17.0 | no |

Dev packages **not installed**: pytest-cov, types-xmltodict, types-psutil, types-lxml, lxml-stubs, flake8, mypy, smokeshow, lazydocs, mkdocs plugins, watchdog. So lint, type check, coverage and docs build were not run.

Verdict: the runtime environment is working. It is not the pinned environment, so results apply to the versions above only.

## 2. Existing test suite

Command: `.\.venv\Scripts\python.exe -m pytest -q -p no:cacheprovider --junitxml=<file>`

| Metric | Result |
|---|---|
| Collected | 475 |
| Passed | 475 |
| Failed / errors / skipped | 0 / 0 / 0 |
| Duration | 7.22 s reported by pytest (9.6 s wall). Re-runs: 7.34 s and 9.7 s wall |
| Exit code | 0 |

Tests per module (from the JUnit file): `test_filters` 131, `test_hl7` 92, `test_utils` 105, `test_parsers` 59, `test_renderers` 44, `test_tags` 28, `test_loaders` 16.

Gap: the CLI (`__main__.py`) has no test module. The scripted cases in section 4 cover it for this baseline.

## 3. Sample conversions

| Run | Command (abbreviated) | Result |
|---|---|---|
| C-CDA | `python -m fhir_converter --from-file ...TD-CCDA-01_valid_basic.ccda --to-dir <out> --template-name CCD --source-format CCDA` | Exit 0, `RENDER SUCCESS`. Total time shown 1.52 s, wall 1.63 s, memory 42M. Output `TD-CCDA-01_valid_basic.json`, 4,026 bytes, entries: Composition, Organization, Practitioner, Patient, DocumentReference |
| STU3 Patient | `--from-file ...TD-STU3-01_patient_valid.json --template-name Patient --source-format STU3` | Exit 0. Output equals expected Patient |
| STU3 Immunization | `...TD-STU3-05_immunization.json --template-name Immunization` | Exit 0. `occurrenceDateTime` `2024-01-15T10:30:00.000-05:00` |
| Bad XML | `...TD-INV-01_malformed.xml` | Exit 1, `RENDER FAILURE`, `XMLSyntaxError`, wall 1.43 s, no output folder left behind |
| Warm API render | `CcdaRenderer.render_fhir_string("CCD", ...)`, 20 repeats | Average 4.2 ms on a tiny synthetic document. Not a benchmark for real documents |

The repo's own `scripts/benchmark.py` was **not run**.

## 4. Case execution

All 53 cases ran. Status per case is in [test-cases.md](test-cases.md).

| Group | Cases | Pass | Fail | Blocked |
|---|---|---|---|---|
| CLI functional | 6 | 6 | 0 | 0 |
| CLI negative | 9 | 9 | 0 | 0 |
| API | 7 | 7 | 0 | 0 |
| Mapping | 10 | 10 | 0 | 0 |
| Date/time | 8 | 8 | 0 | 0 |
| Invalid input | 9 | 9 | 0 | 0 |
| Utilities | 3 | 3 | 0 | 0 |
| Regression (pytest) | 1 | 1 | 0 | 0 |
| **Total** | **53** | **53** | **0** | **0** |

Method: 52 cases were run by one script (about 51 seconds) that calls the real CLI as a subprocess and the Python API, then compares actual output with the expected result. TC-REG-01 was the pytest run in section 2.

Important limit on what "Pass" means: the expected results were written after exploratory runs of the same code, so the suite confirms **current behavior**. It would catch regressions. It does not prove the behavior is correct against the FHIR or C-CDA specifications. Nine passes confirm behavior that may be a defect (section 6).

## 5. Output validation

Scope: the 11 C-CDA files that convert successfully plus the 5 STU3 files. Checks were structural, written for this report. **No FHIR validator or R4 profile check was run.** None is available offline in this repo.

Checks on each bundle: type is `batch`; every entry has `resourceType` and a UUID `id`; `fullUrl` is `urn:uuid:<id>`; `request` is `PUT <Type>/<id>`; ids are unique; every `reference` resolves to an entry; no empty strings, nulls, empty lists or objects; `birthDate` and `date` values match the FHIR date patterns; the `DocumentReference` attachment decodes and contains the source document id.

| Result | Count |
|---|---|
| Files checked | 17 (11 C-CDA bundles + 5 STU3 + the one C-CDA file that cannot be generated, TD-CCDA-08, which failed as expected) |
| Bundle structure, ids, `fullUrl`, `request`, uniqueness, empty values, date formats, attachment decode | 0 issues in all 11 generated bundles |
| Unresolved reference `Composition.attester[0].party.reference = "Practitioner/"` | 11 of 11 bundles |
| Unresolved reference `Composition.subject.reference = "Patient/"` | 1 (TD-CCDA-09, which has no recordTarget) |
| STU3 `Immunization` carries `wasNotGiven`, which is not an R4 element | 1 |
| STU3 file with no `resourceType` passes through without one | 1 (TD-STU3-04, expected pass-through) |

The attachment decodes with zlib (not gzip) to text containing the source document id. It is a re-serialized copy of the source XML, not a byte-for-byte copy.

## 6. Actual failures and observations

### 6.1 Test failures

None. No pytest test and no documented case failed.

### 6.2 Problems encountered during the run (my own errors and environment issues)

| # | What happened | Cause | Resolution |
|---|---|---|---|
| E1 | The first environment-check command failed with a PowerShell parse error | Inline Python with quotes inside a PowerShell command | Moved the code to a script file and re-ran. Not a project problem |
| E2 | The first output-validation run reported "attachment not decodable (BadGzipFile)" for every C-CDA bundle | My validator assumed gzip. The data starts `x\x9c`, a zlib header. `filters.py` line 133 documents zlib | Fixed the validator to use zlib and re-ran. Recorded as OBS-11 because the filter name `gzip` is misleading |
| E3 | An earlier exploratory directory run showed 9 output files instead of 11 | Most likely my command: piping through `Select-Object -First` can stop the converter before it finishes (**not proven**). The scripted run and a re-run both produced 11 | See section 7 |
| E4 | `$LASTEXITCODE` appeared as -1 in one exploratory run | The exit code was read after piping through `Select-String` | Exit codes in the scripted run are read directly from the process |
| E5 | `pip install -e .` failed on Python 3.14 | Pinned `lxml 5.3.0` and `pyjson5 1.6.5` build from source and the `pyjson5` build errors out | Installed newer versions (see section 1) |

### 6.3 Observations about the product (reproduced in this run)

Not confirmed defects until the team decides the intended behavior.

| ID | Observation | Evidence | Suggested severity if treated as a defect |
|---|---|---|---|
| OBS-01 | A `--continue_on_error` directory run with a failing file prints `RENDER SUCCESS` and exits 0 | TC-CLI-13: 11 outputs, TD-CCDA-08 traceback printed, exit 0 | High for automation (a pipeline cannot detect partial failure) |
| OBS-02 | `--from-dir` with STU3 converts nothing and still exits 0 | TC-CLI-15 | Medium |
| OBS-03 | Non-CDA XML converts without error into a Bundle with empty references | TC-INV-03 | Medium |
| OBS-04 | `Composition.attester.party.reference` is `Practitioner/` with no id, in all 11 bundles | TC-MAP-10, section 5 | Medium to High (invalid reference in every C-CDA output) |
| OBS-05 | STU3 `Immunization` keeps `wasNotGiven` and adds `.000` milliseconds | TC-DT-07, TC-INV-09 | Low to Medium |
| OBS-06 | Time without a time zone is reduced to a date | TC-DT-02 | Medium (the code history says this is intended) |
| OBS-07 | Unmapped gender code yields no `gender` | TC-MAP-03 | Medium |
| OBS-08 | Missing `recordTarget` leaves `Composition.subject` as `Patient/` | TC-MAP-07, section 5 | Medium |
| OBS-09 | `DocumentReference.date` follows the clock (one-second resolution), so output is not byte-identical across runs | Two renders 2 seconds apart gave `14:47:01Z` and `14:47:03Z`. Same-second renders matched | Low |
| OBS-10 | CLI needs Python 3.11 or newer while packaging says 3.8 | Code reading only, not run on an older Python | Medium |
| OBS-11 | `gzip` filter produces zlib data | E2 | Low |
| OBS-12 | `pip check` shows 8 pin conflicts on Python 3.14 | Section 1 | Low |

## 7. Open items

- E3 is unexplained with certainty. A repeat of the directory run through the CLI without piping gave 11 files, so no product issue is indicated.
- Whether OBS-01 to OBS-08 are defects needs a decision from someone who knows the downstream consumers.

## 8. Not executed and limitations

- Lint (flake8), type check (mypy), coverage and the docs build: tools not installed.
- The pinned dependency set and Python 3.8 to 3.13: not run. The CI matrix claims 3.8 to 3.12.
- The repo benchmark script.
- Windows only. No Linux or macOS run.
- Only the `CCD` C-CDA template and STU3 `Patient` and `Immunization` templates were exercised by these cases. The other 8 C-CDA document templates and about 50 STU3 templates were not.
- Only synthetic data. The existing `data\sample` files were not used because their provenance is unknown and they may contain real patient data.
- No FHIR R4 schema or profile validation. Structural checks only.
- No performance, load or security testing.

## 9. Conclusion

The baseline is **healthy for the exercised paths**: environment works with newer dependency versions, the existing suite passes completely, and the CLI and API convert synthetic C-CDA and STU3 input and fail predictably on malformed input. Before relying on the output downstream, review OBS-01, OBS-04 and OBS-08, and run the pinned environment on Python 3.12.

This report is a software test summary. It is not a clinical validation or a security or compliance attestation.
