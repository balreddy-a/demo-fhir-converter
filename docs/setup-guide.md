# Setup Guide (Windows)

Derived from `pyproject.toml`, `Makefile`, `poetry.toml`, `fhir_converter/__main__.py` and a working local install on 2026-11-04. Commands are PowerShell. Items marked **UNVERIFIED** were not run.

## 1. Prerequisites

| Requirement | Detail |
|---|---|
| Python | **3.11 or newer.** `pyproject.toml` says `^3.8.1`, but `__main__.py` imports `enum.StrEnum` (3.11+), so the CLI will not start on older versions. The Makefile targets 3.12. Verified working on 3.14 |
| Python launcher | `py` (installed with python.org builds). Check with `py -0` |
| Git | Only needed to clone |
| Poetry | Optional. Not installed on the verified machine |
| C compiler | Not needed when wheels are available. On Python 3.14 the pinned `lxml 5.3.0` and `pyjson5 1.6.5` try to build from source and fail (see section 6) |

If `python` opens the Microsoft Store, use `py -3.12` (or your version) instead, or disable the alias under Settings > Apps > Advanced app settings > App execution aliases.

## 2. Install (verified path: venv + pip)

From the repo root (the folder that contains `pyproject.toml`):

```powershell
cd C:\Users\balreddy.arutla\Desktop\projects\demo-git\fhir_converter-git\fhir-converter
py -3.14 -m venv .venv          # use -3.12 or -3.11 if you prefer
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install python-liquid==1.12.1 xmltodict pyjson5 frozendict importlib_resources psutil isodate lxml typing-extensions pytest pytest-mock
.\.venv\Scripts\python.exe -m pip install -e . --no-deps
```

Calling `.\.venv\Scripts\python.exe` directly avoids PowerShell execution-policy problems with `Activate.ps1`. To activate anyway:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

Versions installed and tested (475 tests passed):

| Package | Pinned in pyproject | Installed |
|---|---|---|
| python-liquid | 1.12.1 | 1.12.1 |
| lxml | 5.3.0 | 6.1.3 |
| pyjson5 | 1.6.5 | 2.0.1 |
| xmltodict | 0.13.0 | 1.0.4 |
| frozendict | 2.3.10 | 2.5.1 |
| importlib_resources | 6.1.1 | 7.1.0 |
| psutil | 5.9.7 | 7.2.2 |
| isodate | 0.6.1 | 0.7.2 |
| typing-extensions | 4.12.2 | 4.16.0 |
| pytest / pytest-mock | 7.4.4 / 3.12.0 | 9.2.0 / 3.17.0 |

`python-liquid` was kept at its pin. `tags.py` imports internal `liquid` modules (`liquid.lex`, `liquid.expressions`), so other versions are untested and may break.

## 3. Install (alternatives)

Poetry (project's own method, **UNVERIFIED** here):

```powershell
py -3.12 -m pip install poetry
poetry config virtualenvs.in-project true
poetry env use 3.12
poetry install --no-interaction
poetry run pytest
```

Use Python 3.12 or earlier so the pinned `lxml` and `pyjson5` install from wheels. `poetry.toml` already sets `in-project = true`.

From PyPI as a dependency of another project (**UNVERIFIED**, package name from README):

```powershell
pip install python-fhir-converter
```

## 4. Verify the install

```powershell
.\.venv\Scripts\python.exe -m pytest -q
```

Expected: `475 passed` (verified on 2026-11-04, about 8 seconds).

```powershell
.\.venv\Scripts\python.exe -m fhir_converter --help
```

Expected: usage text listing `--from-file`, `--from-dir`, `--to-dir`, `--template-dir`, `--template-name`, `--source-format`, `--continue_on_error`, `--flatten_to_dir`.

## 5. First run

Synthetic sample data is in `data\synthetic` (see [test-data.md](test-data.md)).

C-CDA to FHIR R4:

```powershell
.\.venv\Scripts\python.exe -m fhir_converter `
  --from-file .\data\synthetic\ccda\TD-CCDA-01_valid_basic.ccda `
  --to-dir .\data\out `
  --template-name CCD `
  --source-format CCDA
```

STU3 to FHIR R4:

```powershell
.\.venv\Scripts\python.exe -m fhir_converter `
  --from-file .\data\synthetic\stu3\TD-STU3-01_patient_valid.json `
  --to-dir .\data\out `
  --template-name Patient `
  --source-format STU3
```

Success prints a `RENDER SUCCESS` banner (total time, finish time, memory) and exits 0. Failure prints a traceback, `RENDER FAILURE`, and exits 1. Argument errors exit 2. After activation you can use `fhir_converter_cli` instead of `python -m fhir_converter`.

Output file name is the input name with a `.json` extension, written to `--to-dir`.

## 6. Configuration

There are no config files and the modules inspected read no environment variables. All configuration is by CLI flag or Python argument.

| Setting | Where | Default |
|---|---|---|
| `--template-name` | CLI, required | none. C-CDA: `CCD`, `ConsultationNote`, `DischargeSummary`, `HistoryandPhysical`, `OperativeNote`, `ProcedureNote`, `ProgressNote`, `ReferralNote`, `TransferSummary`. STU3: a resource name such as `Patient`, `Immunization` |
| `--source-format` | CLI, required | `CCDA` or `STU3` |
| `--template-dir` | CLI | none. Adds user templates ahead of the packaged ones |
| `--continue_on_error` | CLI | off. Note the underscore |
| `--flatten_to_dir` | CLI | off. Note the underscore |
| `render_narrative` | `CcdaRenderer(template_globals={...})` | `False` |
| `code_mapping` | `template_globals` | loaded from `templates/ccda/ValueSet/ValueSet.json` |
| `cache_size` | `make_environment()` | 300 templates |
| `encoding` | render methods | `utf-8` |

Developer tooling (`Makefile`, requires Poetry): `make test`, `make cov`, `make type-check` (mypy), `make linting-check` (flake8, max line length 120). `mypy.ini` ignores missing stubs for `isodate` and `dateutil`.

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Failed to build lxml pyjson5` during `pip install -e .` | Pinned versions have no wheel for your Python (seen on 3.14) | Install the unpinned versions as in section 2, then `pip install -e . --no-deps`, or use Python 3.12 |
| `ImportError: cannot import name 'StrEnum'` | Python older than 3.11 | Use Python 3.11+ |
| `python` opens the Microsoft Store | App execution alias | Use `py -3.x` or disable the alias |
| `Activate.ps1 cannot be loaded` | Execution policy | Call `.\.venv\Scripts\python.exe` directly, or use the process-scope command in section 2 |
| `RENDER FAILURE` with `TemplateNotFound` | Wrong `--template-name` | Use a name from section 6 |
| `--from-dir` on STU3 produces no files and still says SUCCESS | Directory mode only picks `.ccda` and `.xml` | Convert STU3 files one at a time with `--from-file` |

## 8. Data handling

C-CDA files usually contain PHI, and the output embeds the whole source document. Do not put real patient data in this repo, in `data\out`, or in a shared folder. Use the synthetic data in `data\synthetic`.

This guide is a technical setup summary, not an attestation of the software's security or compliance.
