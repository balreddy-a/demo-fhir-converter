# Troubleshooting

Each entry says whether it was **observed** in this project's baseline work (2026-11-04) or is **expected from the code** but not seen. Related: [setup-guide.md](setup-guide.md), [test-execution-report.md](test-execution-report.md).

## 1. Install and environment

### 1.1 `pip install -e .` fails building `lxml` and `pyjson5` (observed)
Symptom: `Failed to build lxml pyjson5`, with `Cython.Compiler.Errors.CompileError: pyjson5.pyx` in the log.
Cause: `pyproject.toml` pins `lxml 5.3.0` and `pyjson5 1.6.5`. On Python 3.14 there is no prebuilt wheel, so pip builds from source, and the `pyjson5` build fails.
Fix (what worked):
```powershell
.\.venv\Scripts\python.exe -m pip install python-liquid==1.12.1 xmltodict pyjson5 frozendict importlib_resources psutil isodate lxml typing-extensions pytest pytest-mock
.\.venv\Scripts\python.exe -m pip install -e . --no-deps
```
Alternative (not tried): use Python 3.12, which the Makefile targets.
Side effect: `pip check` then lists 8 version conflicts with the pins. That is expected.

### 1.2 `python` opens the Microsoft Store or prints "Python was not found" (observed)
Cause: the Windows app execution alias.
Fix: use the launcher, `py -3.14` (or `py -3.12`), or turn the alias off in Settings > Apps > Advanced app settings > App execution aliases. Use `py -0` to list installed versions.

### 1.3 `poetry` is not recognized (observed)
Poetry is not installed. The Makefile and CI use it, but the project also installs with venv and pip (see section 1.1). To use Poetry: `py -3.12 -m pip install poetry`.

### 1.4 `ImportError: cannot import name 'StrEnum'` (expected from code, not seen)
`fhir_converter/__main__.py` imports `enum.StrEnum`, which exists from Python 3.11. Use Python 3.11 or newer for the CLI.

### 1.5 `Activate.ps1 cannot be loaded` (expected, not seen)
PowerShell execution policy. Skip activation and call `.\.venv\Scripts\python.exe` directly, or run `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` first.

### 1.6 flake8, mypy and coverage do not run (observed)
They are dev dependencies and were not installed in the baseline. Install with `pip install flake8 mypy pytest-cov` (versions untested here). The Makefile targets need Poetry.

### 1.7 README example paths do not exist (observed)
README and `scripts/examples.py` use `data/sample/ccda/CCD.ccda`. A fresh clone did not contain `data/sample`. Use `data\synthetic` from [test-data.md](test-data.md).

## 2. Running conversions

### 2.1 `RENDER FAILURE` with `TemplateNotFound` (observed)
Cause: wrong `--template-name`. The name must match a template file in the folder for the chosen format (for example `CCD` for C-CDA, `Patient` for STU3). Match the file name exactly; case handling was not tested.
Fix: list templates with `Get-ChildItem .\fhir_converter\templates\ccda -File` (root-level files are the document templates).

### 2.2 `XMLSyntaxError: Document is empty` or tag mismatch (observed)
Cause: the input file is empty or not well-formed XML.
Fix: open the file, check it is complete, and that it is UTF-8. Nothing is written to `--to-dir`.

### 2.3 `Json5EOF: No JSON data found` or `Unclosed object` (observed)
Cause: STU3 input is empty or truncated JSON.

### 2.4 `Malformed HL7 datetime <value>` (observed)
Full text: `filter 'format_as_date_time': unexpected error: Malformed HL7 datetime not-a-date`.
Cause: a date in the C-CDA is not a valid HL7 date (digits such as `20240115` or `20240115103000-0500`). Values tested as valid: 4, 6, 8 and 14 digit forms, with or without a `+/-hhmm` zone, and a fractional second. Values tested as invalid: `abc`, empty, `2024011`. The whole file fails.
Fix: correct the value in the source document. With `--continue_on_error` in directory mode the other files still convert.

### 2.5 Run says `RENDER SUCCESS` but some files are missing (observed, OBS-01)
Cause: `--continue_on_error` prints the traceback and carries on, and the final banner and exit code still say success.
Fix: scan the output for `RenderingError`, and compare the count of input and output files. A script example:
```powershell
(Get-ChildItem $IN -Recurse -Include *.ccda,*.xml).Count
(Get-ChildItem $OUT -Recurse -Filter *.json).Count
```

### 2.6 `--from-dir` for STU3 produces no files and exits 0 (observed, OBS-02)
Cause: directory mode only reads `.ccda` and `.xml` files.
Fix: convert STU3 files one at a time with `--from-file`, in a loop if needed.

### 2.7 Output has `Practitioner/` or `Patient/` references with no id (observed, OBS-03, OBS-04, OBS-08)
Cause: the source has no matching element (for example no `recordTarget`) or, for the attester, the current template output. Some systems reject such references.
Fix: review the output before loading it into a FHIR server. This is a data-quality check, not something a flag can turn off.

### 2.8 Time of day missing from a date (observed, OBS-06)
Cause: the C-CDA value has no time zone. The converter keeps only the date.
Fix: supply the time zone in the source (`20240115103000-0500`).

### 2.9 A code is missing in the output, for example no `gender` (observed, OBS-07)
Cause: the code is not in `templates/ccda/ValueSet/ValueSet.json` and the mapping default is empty.
Fix: add the code to the value set in your own copy of the templates, or fix the source code value.

### 2.10 Cannot decode `DocumentReference.content.attachment.data` as gzip (observed, OBS-11)
Cause: the filter is named `gzip` but the data is zlib-compressed and then base64-encoded.
Fix:
```python
import base64, zlib
text = zlib.decompress(base64.b64decode(data)).decode("utf-8")
```

### 2.11 Output differs between two runs (observed, OBS-09)
`DocumentReference.date` uses the current time. Everything else compared (resource ids) was identical. Ignore that field when diffing.

### 2.12 Exit codes (observed)
0 success, 1 conversion failure, 2 argument error. Reading `$LASTEXITCODE` after a pipe such as `| Select-String` gives the last command in the pipe, not the converter.

## 3. PowerShell tips (observed while testing)

- **Red `NativeCommandError` text on usage errors.** Windows PowerShell 5.1 wraps a program's stderr. For an argument error, the text is the converter's normal usage message and exit code 2. Not a separate failure.
- **Parse errors when running Python inline.** Quotes and braces inside `python -c "..."` collide with PowerShell. Put the code in a `.py` file instead.
- **Truncated or partial runs when piping to `Select-Object -First N`.** The pipeline can stop the upstream program early. Redirect to a file (`*> run.log`) and read the file afterwards.
- **`Select-String -Path <directory>`** raises "cannot be read: Access denied" because a folder was passed as a file. Use `Get-ChildItem -File` first.

## 4. Tests

### 4.1 Suite passes with 475 tests (observed)
If you see a different number, the dependency set or the test files changed. The pytest count is higher than the number of `def test_` functions because some tests are parametrized.

### 4.2 Tests behave differently on pinned dependencies (not tested)
The baseline used newer versions than the pins. If a failure appears only under the pins, compare with the table in [test-execution-report.md](test-execution-report.md).

## 5. When to escalate

Raise an issue when you see: a crash that is not a `RenderingError`, a converted file whose content differs from the source in a clinically meaningful way, or output that a FHIR validator rejects for reasons beyond sections 2.7 to 2.9. Include the input (synthetic or de-identified), the command, the Python and package versions, and the full traceback. Never attach real patient data.
