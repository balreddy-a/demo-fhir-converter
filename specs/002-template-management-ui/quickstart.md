# Quickstart: Validate the Template Management UI

For use after implementation. **Nothing here has been run.** Implementation, the baseline run and any tool installation are approval-gated (see [plan.md](plan.md) section 10). PowerShell, from the repository root. Use synthetic data only. Write any output to a temporary folder, never `data/out`.

The user directory option name is **TBD** (D-12), so `<user-dir option>` stands for it below.

## Prerequisites

- `.venv` present (Python 3.14.10 locally, outside the supported 3.8 to 3.12, so CI is the compatibility authority).
- pytest available. flake8, mypy and Poetry only if approved and installed (D-27).
- A small synthetic user directory, for example one containing `CCD.liquid` and `Section/_UserOnly.liquid` with invented content.

## 1. Baseline, before any change

```powershell
.\.venv\Scripts\python.exe -m pytest -q
```
Record the passed count (expected 475, unverified). Convert the named fixtures to `$env:TEMP\ui-before`, and save `python -m fhir_converter --help` output for comparison. See [plan.md](plan.md) 9.5.

## 2. Start the UI

```powershell
.\.venv\Scripts\python.exe -m fhir_converter.ui
```
Expected: one startup message with `127.0.0.1`, an automatically chosen port and the token-bearing address, and no further console output while you browse. Open the address in a browser.

With a user directory:
```powershell
.\.venv\Scripts\python.exe -m fhir_converter.ui <user-dir option> <path to synthetic folder>
```

## 3. Walk-through checks

| Step | Expected | Requirements |
|---|---|---|
| Choose C-CDA, then STU3 | Separate lists, `.liquid` files only, counts equal files on disk | FR-010, FR-014 |
| Filter by category, search a name, search source text | Correct results, cut-off note if capped | FR-011, FR-020, FR-021 |
| Open a template | Source shown as inert text with metadata | FR-030, FR-031 |
| Open a root template | Direct references, transitive templates with chains, dynamic or unsupported list, best-effort note | FR-040 to FR-045 |
| With the user directory, open `CCD` | "Overrides packaged" in C-CDA view, "user only" in STU3 view, both sources viewable | FR-050, FR-051 |
| Open the unreferenced sections view (C-CDA) | "Potentially unreferenced (best-effort)" with the note and incompleteness count when relevant. No such view for STU3 | FR-060 |
| Look at the page footer or header | Build time and a note that a restart refreshes the catalog | FR-006 |
| Request a missing name, a traversal name, a `.json` name | Fixed error message and code only | FR-070, SEC-004 |

## 4. Process and security checks

- Start two instances at once: different ports, both work (FR-005).
- Start with a missing user directory: non-zero exit, fixed message, nothing listening (FR-004).
- Send a request with a wrong Host, a wrong Origin, and no token: each rejected (SEC-002, SEC-018).
- Send POST, PUT, PATCH, DELETE: each rejected, no file changed (FR-080).
- Compare the template trees before and after the session: identical (SEC-009).

## 5. Regression

```powershell
.\.venv\Scripts\python.exe -m pytest -q
```
Expected: the baseline count plus the new tests, none failing. Convert the named fixtures to `$env:TEMP\ui-after` and compare with the before set: zero differences (FR-091). Compare `--help` output. Rerun the TC-CLI cases in `docs/test-cases.md` by hand.

## 6. Investigations and checks that need approval

- Packaging build and clean-install check (I-1, FR-094).
- python-liquid source-reading confirmation (I-2, D-19).
- Standard-library server prototype on the supported Python versions (I-4).
- Scanner calibration against the packaged templates (I-5).

## 7. Report

Record the commands and the real results. List anything not run. Update `docs/test-cases.md` statuses only for cases actually run. Do not describe templates or output as FHIR-valid.

References: [contracts/ui-contract.md](contracts/ui-contract.md), [data-model.md](data-model.md), [research.md](research.md).
