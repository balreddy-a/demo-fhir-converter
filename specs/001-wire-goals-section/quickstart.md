# Quickstart: Validate the Goals feature

Use after implementation, which is blocked until developer approval (FR-012). PowerShell, from the repository root. Use only synthetic data and write output outside `data/out`.

## Prerequisites

- `.venv` present. Tools needed: pytest. flake8 and mypy only if the Python filter (D3-B) was approved and they are installed.
- Fixture `data/synthetic/ccda/TD-CCDA-13_goals.ccda` exists (task T004).

## 1. Baseline, before any change

```powershell
.\.venv\Scripts\python.exe -m pytest -q
```
Record the passed count (expected 475, unverified).

Convert the unaffected documents to a temp folder and keep them as the "before" set: `TD-CCDA-01` to `12` except `08`, `tests/data/ccda/CCD.ccda`, `tests/data/ccda/History_and_Physical.ccda`.

```powershell
$B = Join-Path $env:TEMP "goals-before"
.\.venv\Scripts\python.exe -m fhir_converter --from-file data\synthetic\ccda\TD-CCDA-01_valid_basic.ccda --to-dir $B --template-name CCD --source-format CCDA
```

## 2. After the change

Repeat step 1 into `goals-after`, then diff each pair. Expected: no differences.

```powershell
$A = Join-Path $env:TEMP "goals-after"
# same command with --to-dir $A, then:
Compare-Object (Get-Content "$B\TD-CCDA-01_valid_basic.json") (Get-Content "$A\TD-CCDA-01_valid_basic.json")
```

## 3. Goals fixture

```powershell
.\.venv\Scripts\python.exe -m fhir_converter --from-file data\synthetic\ccda\TD-CCDA-13_goals.ccda --to-dir $A --template-name CCD --source-format CCDA
```
Expected (see the fixture table in [plan.md](plan.md)): Goals A, B and E present, C and D absent, every Goal has `lifecycleStatus` and `subject`, no Goal stub without status, conversion exits 0.

## 4. Full suite after

```powershell
.\.venv\Scripts\python.exe -m pytest -q
```
Expected: baseline count plus the new tests, none failing.

## 5. Report

Record the commands and actual results. List anything not run. Update `docs/test-cases.md` statuses only for cases actually run. Do not describe the output as FHIR-valid.

Contract: [contracts/goal-output-contract.md](contracts/goal-output-contract.md). Mapping: [data-model.md](data-model.md).
