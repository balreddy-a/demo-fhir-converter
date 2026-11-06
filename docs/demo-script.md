# Demo Script: python-fhir-converter Baseline

Length: about 14 minutes plus questions. Audience assumed: engineers and QA reviewers who know Python, not necessarily FHIR. Every command and number below was run on 2026-11-04 (Windows 11, Python 3.14.10) and is recorded in [test-execution-report.md](test-execution-report.md).

| # | Segment | Time | Visual |
|---|---|---|---|
| 1 | Project overview | 1.5 min | Slide or [repo-analysis.md](repo-analysis.md) section 1 |
| 2 | Architecture | 2 min | [system-architecture.mmd](diagrams/system-architecture.mmd), [component-architecture.mmd](diagrams/component-architecture.mmd) |
| 3 | Conversion workflow | 2 min | [fhir-conversion-flow.mmd](diagrams/fhir-conversion-flow.mmd), [sequence-diagram.mmd](diagrams/sequence-diagram.mmd) |
| 4 | Live execution | 4 min | Terminal |
| 5 | Testing | 2 min | Terminal and [test-execution-report.md](test-execution-report.md) |
| 6 | Challenges and learnings | 2 min | [test-plan.md](test-plan.md) observations table |
| 7 | Close | 0.5 min | Next steps |

## Before you start (5 minutes)

1. Open PowerShell in `fhir-converter\`. Increase font size.
2. Set variables:
```powershell
$PY  = ".\.venv\Scripts\python.exe"
$OUT = "$env:TEMP\fhir-demo"
if (Test-Path $OUT) { Remove-Item $OUT -Recurse -Force }
```
3. Check the environment once: `& $PY -m fhir_converter --help` should print usage.
4. Open the diagram files in a Mermaid viewer (VS Code Mermaid extension or mermaid.live) in separate tabs. They were not rendered during preparation, so look at them first.
5. Use only `data\synthetic` files. Do not open `data\sample`: its provenance is unknown.
6. Fallback if the live run fails: show the recorded outputs in section 3 of the execution report and say so.

## 1. Project overview (1.5 min)

Say:
- This is a Python port of Microsoft's FHIR-Converter. It converts C-CDA clinical documents and FHIR STU3 resources into FHIR R4.
- Use case: feed EHR document exports into a FHIR-based pipeline without leaving Python.
- It is a library and a command line tool. There is no web service.
- About 1,090 Liquid templates hold the mapping. The Python code is the engine (nine small modules).
- Version 0.3.0. Supported inputs: C-CDA and STU3 only. HL7 v2 and plain JSON are not supported.

## 2. Architecture (2 min)

Show the system diagram, then the component diagram.

Say:
- Input goes through a renderer. `CcdaRenderer` for C-CDA, `Stu3FhirRenderer` for STU3 (both in `renderers.py`).
- The renderer parses the input to a dictionary (`parsers.py`), runs a Liquid template (`templates/`), and cleans the result (`hl7.parse_fhir`).
- Custom filters (`filters.py`) and tags (`tags.py`) extend Liquid: section lookup by templateId, code mapping, deterministic UUIDs, `evaluate` and `mergeDiff`.
- Template loading (`loaders.py`) lets your own templates sit in front of the packaged ones with `--template-dir`.
- Dependencies that matter: python-liquid, lxml, pyjson5.
- Deployment: it runs inside whatever Python process calls it. No Docker, server or infrastructure code is in the repo.

## 3. Conversion workflow (2 min)

Show the flow diagram, then the sequence diagram.

Say:
- Step 1, parse: XML becomes a nested dict. The renderer also stores the source text as `_originalData`.
- Step 2, render: `CCD.liquid` includes a header and one template per clinical section. Each section finds its data by templateId, then renders entry and resource templates.
- Step 3, clean: `parse_fhir` drops empty values and merges duplicate bundle entries.
- Step 4, write: JSON out as a string, stream or `.json` file.
- IDs are derived from a hash of the source identifiers, so the same input gives the same ids.
- Any failure becomes a `RenderingError` with the original cause attached.

## 4. Live execution (4 min)

**4.1 Convert a C-CDA file** (about 2 seconds)
```powershell
& $PY -m fhir_converter --from-file .\data\synthetic\ccda\TD-CCDA-01_valid_basic.ccda --to-dir $OUT --template-name CCD --source-format CCDA
```
Expect `RENDER SUCCESS` and a total time near 1.5 seconds (mostly start-up and template loading).

**4.2 Look at the result**
```powershell
$b = Get-Content "$OUT\TD-CCDA-01_valid_basic.json" -Raw | ConvertFrom-Json
$b.entry.resource | Select-Object resourceType, id | Format-Table -AutoSize
($b.entry.resource | Where-Object resourceType -eq Patient | Select-Object gender, birthDate | Format-List)
```
Expect five resources: Composition, Organization, Practitioner, Patient, DocumentReference. Patient shows `gender: female`, `birthDate: 1980-01-01`. Say that the source used code `F` and the value set mapped it to `female`.

**4.3 Date handling**
```powershell
foreach ($f in 'TD-CCDA-01_valid_basic','TD-CCDA-05_date_no_timezone') {
  & $PY -m fhir_converter --from-file ".\data\synthetic\ccda\$f.ccda" --to-dir $OUT --template-name CCD --source-format CCDA *> $null
  $x = Get-Content "$OUT\$f.json" -Raw | ConvertFrom-Json
  "$f -> Composition.date = " + ($x.entry.resource | Where-Object resourceType -eq Composition).date
}
```
Expect `2024-01-15T10:30:00-05:00` for the first and `2024-01-15` for the second. Say: with no time zone the converter keeps only the date. That is a design choice in the code history, and we list it as observation OBS-06.

**4.4 Failure behavior**
```powershell
& $PY -m fhir_converter --from-file .\data\synthetic\invalid\TD-INV-01_malformed.xml --to-dir "$OUT-bad" --template-name CCD --source-format CCDA *> "$env:TEMP\demo-bad.log"
"exit code: $LASTEXITCODE"
Get-Content "$env:TEMP\demo-bad.log" | Select-String 'XMLSyntaxError|RENDER' | Select-Object -Last 2
```
Expect exit code 1, `XMLSyntaxError` (tag mismatch) and `RENDER FAILURE`, and no output folder created.

**4.5 Optional: partial failure in directory mode** (use if time allows, about 15 seconds)
```powershell
& $PY -m fhir_converter --from-dir .\data\synthetic\ccda --to-dir "$OUT-dir" --template-name CCD --source-format CCDA --continue_on_error *> "$env:TEMP\demo-dir.log"
"exit code: $LASTEXITCODE"; (Get-ChildItem "$OUT-dir" -Filter *.json).Count
Get-Content "$env:TEMP\demo-dir.log" | Select-String 'RENDER (SUCCESS|FAILURE)'
```
Expect 11 output files (one input has a malformed date), exit code 0 and `RENDER SUCCESS`. Say: this is the most important finding for automation. One file failed and the run still says success.

**Cleanup:** `Remove-Item $OUT,"$OUT-bad","$OUT-dir" -Recurse -Force -ErrorAction SilentlyContinue`

## 5. Testing (2 min)

Run:
```powershell
& $PY -m pytest -q
```
Expect `475 passed` in about 7 seconds (about 10 seconds wall).

Say:
- The repo already had 475 pytest tests across seven modules. All passed. The CLI has no tests of its own.
- For this baseline we added 53 documented cases (CLI, API, mapping, date/time, invalid input) with synthetic data. 52 ran from a script that calls the real CLI and API, plus the pytest run. Result: 53 pass, 0 fail.
- Be clear about what "pass" means: the expected results were written after exploratory runs, so this is a regression baseline of current behavior, not proof of correctness against the FHIR specification.
- Output checks: ids, `fullUrl`, `request`, references, empty values and date formats were verified on all 11 generated bundles. No FHIR validator was run.

## 6. Challenges and learnings (2 min)

Challenges (all real):
- The pinned `lxml` and `pyjson5` do not install on Python 3.14. We used newer versions. `pip check` lists 8 conflicts with the pins. The pinned set on Python 3.12 has not been run.
- No Poetry, flake8 or mypy on the machine, so lint, type check and coverage were not run.
- The README sample paths did not exist in the clone. We created synthetic data.
- Two mistakes of our own: a PowerShell quoting error, and a validator that assumed gzip when the data is zlib. Both were fixed and are written down in the report.

Learnings:
- Behaviors worth a decision: a run with failed files reports success (OBS-01); every C-CDA output has `Composition.attester.party.reference = "Practitioner/"` with no id (OBS-04); a missing patient leaves `Composition.subject = "Patient/"` (OBS-08).
- Directory mode ignores STU3 JSON (OBS-02) and silently converts non-CDA XML (OBS-03).
- The `gzip` filter uses zlib (OBS-11).
- The design is easy to extend: most mapping changes are template edits.

## 7. Close (0.5 min)

Next steps, in order:
1. Decide whether OBS-01, OBS-04 and OBS-08 are defects.
2. Run the pinned dependencies on Python 3.12 and repeat the baseline.
3. Add CLI tests and run flake8, mypy and coverage.
4. Add a FHIR R4 validator step.
5. Extend test coverage to the other C-CDA and STU3 templates.

## Reviewer questions and short answers

**1. Is it production ready?**
The README says so, but this baseline cannot confirm it. The existing tests and our 53 cases pass. Open items are the observations above, no FHIR validation, no CLI tests, only synthetic data, and the pinned versions are untested on this machine.

**2. Why Liquid templates instead of Python code?**
It is a port of Microsoft's C# converter, which uses Liquid. Mapping lives in text files, so users can change it without touching Python, and existing templates can be reused with small syntax edits (commas between parameters, explicit variable names).

**3. What formats does it support?**
C-CDA to FHIR R4 and STU3 to FHIR R4. The README says HL7 v2 and plain JSON are not supported.

**4. How many templates exist and how many did you test?**
About 1,090 template files (822 C-CDA, 265 STU3). Testing here covered `CCD` and the STU3 `Patient` and `Immunization` templates. The other C-CDA document types and about 50 STU3 resources are untested in this baseline.

**5. Does it validate the FHIR output?**
No. The code has no validation step. We ran structural checks only. Conformance to R4 or US Core profiles is unknown.

**6. What is the performance?**
Measured here: the CLI takes about 1.5 seconds per run because of start-up, and a warm API render of the tiny synthetic file took 4.2 ms on average. The README quotes 10 to 50 ms on real sample files on an M1 laptop. We did not run the repo benchmark, and we have no numbers for large documents.

**7. Can it run as a service or in parallel?**
There is no API or server in the repo. It is a library, so a caller can wrap it. It is single-process, and any parallelism has to be added by the caller. Neither was tested.

**8. Are the results deterministic?**
Resource ids are. Two runs produced the same ids. `DocumentReference.date` uses the current time, so it differs between runs a second or more apart.

**9. Is it safe for patient data?**
We only used synthetic data. Real C-CDA files contain PHI, and the output embeds a compressed copy of the whole source document. Handle the outputs as PHI. Nothing here is a HIPAA, SOC 2 or other compliance statement. Privacy and security review is a separate step.

**10. What failed?**
No test and no case failed. We hit five setup or tooling problems (listed in the report) and found twelve observations. Several of those could be product defects, and we have not labeled them as defects without a decision on intended behavior.

**11. Why did you use different dependency versions?**
The pinned `lxml 5.3.0` and `pyjson5 1.6.5` could not be built on Python 3.14. The newer versions passed all 475 tests. We did not test the pinned set on an older Python.

**12. How do I change a mapping?**
Edit or copy the template and pass `--template-dir`, or change code mappings in `templates/ccda/ValueSet/ValueSet.json`. A custom template that only overrides one file works because the packaged templates remain available (verified with a one-line `Mini.liquid`).

**13. Which Python version should I use?**
Python 3.11 or newer, because the CLI uses `StrEnum`. 3.12 is what the Makefile targets. We ran 3.14. Packaging claims 3.8 support, which the CLI contradicts.

**14. What would you fix first?**
Make a partial failure return a non-zero exit code or at least say so in the banner (OBS-01), fill or drop the empty attester reference (OBS-04), and handle a missing patient explicitly (OBS-08). These are suggestions that need owner agreement.

**15. How was the output checked?**
Scripted checks: bundle type, UUID ids, `fullUrl`, `request`, unique ids, resolvable references, no empty values, FHIR date patterns, and that the attachment decodes to the source. All passed except the unresolved references noted above.

## Things not to claim

- Do not say the output is FHIR R4 valid or US Core conformant. It was not validated.
- Do not say the pinned dependencies work on Python 3.14, or that the baseline ran on the pinned set.
- Do not say real-world documents were tested. Only synthetic data was used.
- Do not call the observations defects until the intended behavior is agreed.
- Do not quote the 4.2 ms figure as a benchmark.
