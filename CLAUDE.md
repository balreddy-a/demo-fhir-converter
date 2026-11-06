# CLAUDE.md

Guidance for working in this repository. Details live in `docs/`; this file lists only what you need to act correctly.

## 1. Overview and architecture

`python-fhir-converter` (v0.3.0) converts C-CDA documents and FHIR STU3 JSON into FHIR R4 JSON using Liquid templates. It is a library plus a CLI. There is no HTTP API, database or server.

Pipeline (see `fhir_converter/renderers.py`):
1. Parse input to a dict (`parsers.parse_xml` or `parse_json`). The result is exposed to templates as `msg`.
2. Render a Liquid template with python-liquid (`env.get_template(name).render({"msg": ...})`).
3. `hl7.parse_fhir` parses the text output, drops empty values, merges duplicate bundle entries.
4. `render_fhir` writes JSON with `pyjson5.encode_io`.

Mapping logic lives in templates, not in Python. Prefer a template change over Python code when changing what is emitted. Diagrams: `docs/diagrams/*.mmd`. Deeper reading: `docs/repo-analysis.md`, `docs/architecture.md`.

## 2. Layout and responsibilities

| Path | Responsibility |
|---|---|
| `fhir_converter/__main__.py` | CLI only: argument parsing, summary banner, exit codes |
| `fhir_converter/renderers.py` | `BaseFhirRenderer`, `CcdaRenderer`, `Stu3FhirRenderer`, `make_environment`, `render_to_dir`, `render_files_to_dir` |
| `fhir_converter/parsers.py` | XML/JSON to dict, empty-value pruning |
| `fhir_converter/hl7.py` | HL7 dates (`hl7_to_fhir_dtm`), C-CDA section lookup, `parse_fhir` |
| `fhir_converter/filters.py`, `tags.py` | Custom Liquid filters and tags (`evaluate`, `mergeDiff`) |
| `fhir_converter/loaders.py` | Template loaders and name resolution |
| `fhir_converter/utils.py`, `exceptions.py` | Helpers, `RenderingError`, `fail` |
| `fhir_converter/templates/ccda/` | C-CDA templates (`Section/`, `Entry/`, `Resource/`, `DataType/`, `Extension/`, `Utils/`, `ValueSet/ValueSet.json`, `cda_narrative.xsl`) |
| `fhir_converter/templates/stu3/` | STU3 to R4 templates, one per resource type |
| `tests/` | pytest suite, fixtures in `tests/data` |
| `data/synthetic/` | Synthetic fixtures used by `docs/test-cases.md` |
| `scripts/` | `examples.py`, `benchmark.py` (expect `data/sample`, `data/templates`, `data/out`) |

## 3. Conversion and template rules

- Template names in sub-folders resolve to files with a `_` prefix: `{% include 'Section/Problem' %}` loads `Section/_Problem.liquid` (`loaders.TemplateSystemLoader`). Root templates (`CCD.liquid`, `Header.liquid`) and `.json` names have no prefix. Create new sub-folder templates with the `_` prefix.
- python-liquid differs from the C# original: commas are required between parameters, and snippet variables must be named explicitly (README "Limitations"). Templates copied from Microsoft's repo need edits.
- C-CDA sections are found by templateId OID with `get_ccda_section_by_template_id`, then each entry is rendered with `batch_render: 'Entry/...', 'entry'`.
- Resource ids come from `Utils/_GenerateId` plus `generate_uuid` (SHA-256 based). Do not change what is hashed without approval: it changes every consumer's ids.
- Code translations belong in `templates/ccda/ValueSet/ValueSet.json` (read via `get_property`, keys like `ValueSet/Gender`). Unmapped codes currently return an empty string.
- Template output is JSON text that `parse_fhir` prunes, so empty values and trailing commas are tolerated. Do not rely on it to hide a wrong mapping.
- Date rule: a datetime without a time zone is reduced to a date (`hl7_to_fhir_dtm`, CHANGES.md 0.1.0). Keep it unless told otherwise.
- The converter embeds the whole source document in output (`_originalData`, `Resource/_DocumentReference.liquid` using the `gzip` filter, which is zlib).
- Public API to keep stable: `BaseFhirRenderer.render_fhir_string`, `render_fhir`, `render_to_fhir`, `make_environment`, `render_to_dir`, `render_files_to_dir`.
- Record behavior changes and bug fixes in `CHANGES.md` (format: version heading, bullet per change).

## 4. Python coding standards

- Type-annotate everything. `mypy.ini` enables `warn_redundant_casts` and `warn_unreachable`.
- flake8 settings (from `Makefile`): max line length 120, ignore `E203,E721`, max complexity 10.
- README shows a Black badge, but no Black configuration exists in the repo. Match the formatting of the surrounding code and do not reformat files you are not changing.
- Docstrings use `Args:`, `Returns:`, `Raises:` sections. Match them.
- Python version: `pyproject.toml` says `^3.8.1` and CI tests 3.8 to 3.12, but `__main__.py` uses `StrEnum` (3.11+). Keep library modules 3.8 compatible; do not add 3.9+ only syntax outside `__main__.py` without asking.
- `tags.py` uses internal python-liquid modules. Treat it as fragile.

## 5. Error handling and logging

- Failures surface as `RenderingError(msg, cause)` (`exceptions.py`); keep the original error as the cause. Do not swallow exceptions.
- File and directory helpers take an `onerror` callable. Default is `fail` (re-raise). The CLI swaps in a printing handler for `--continue_on_error`.
- There is no `logging` usage. The CLI prints with `print_exception` and `print_summary`. Do not introduce logging or new output channels without agreement.
- Known behaviors (see `docs/test-plan.md` OBS-01 to OBS-12): `--continue_on_error` still prints `RENDER SUCCESS` and exits 0 after a failed file; `--from-dir` only reads `.ccda` and `.xml`. `docs/test-cases.md` records these as current behavior, so changing them is a deliberate decision, not a side effect.
- Never put patient data into exception messages.

## 6. Testing

Shell is PowerShell on Windows. The virtual environment is `.venv`.

```powershell
.\.venv\Scripts\python.exe -m pytest -q                      # full suite, expect 475 passed
.\.venv\Scripts\python.exe -m pytest tests\test_hl7.py -q    # one module
```
Project targets (need Poetry): `make test`, `make cov`, `make type-check` (mypy), `make linting-check` (flake8). CI runs flake8, then mypy, then pytest (Ubuntu, Windows, macOS; Python 3.8 to 3.12), then coverage. flake8, mypy, Poetry and pytest-cov were not installed in the local venv when last checked.

Requirements:
- Run the full suite before and after any change. Report real results, never assumed ones.
- Add or update tests in the matching `tests/test_<module>.py`. Template changes need a conversion check on a synthetic fixture.
- The CLI (`__main__.py`) has no unit tests. Check CLI changes manually with `docs/test-cases.md` (TC-CLI-*).
- Manual test data: `data/synthetic/`. Test IDs and statuses live in `docs/test-cases.md`; update them only after actually running.

## 7. Dependency management

- Poetry with exact pins in `pyproject.toml`. No lockfile was seen in the clone.
- The pinned `lxml 5.3.0` and `pyjson5 1.6.5` do not build on Python 3.14. The local venv uses newer versions (see `docs/setup-guide.md`); this is a known deviation, not something to "fix" by editing pins.
- Do not change pins or add dependencies without asking. `python-liquid` stays at 1.12.1 (internal imports in `tags.py`).
- `xmltodict` and `importlib_resources` are declared but not imported by the package. `dateutil` is imported by `filters.py` but only arrives through python-liquid.

## 8. Security and test data

- Use only synthetic data in tests, docs, examples, commits and chat. Ids and names in `data/synthetic/` are invented. Keep it that way.
- Do not use or copy files from `data/sample` (provenance unknown, may contain real patient data) into tests, docs or commits.
- Converter output contains the full source document. Treat anything under `data/out` as sensitive; it is gitignored (as is `data/expected`). Write test output to a temp folder.
- Do not write compliance claims (HIPAA, SOC 2, HITRUST, ISO) or describe output as FHIR-valid or US Core conformant: no validator is run.
- Do not commit secrets, real identifiers (NPI, MRN, DOB) or real addresses.

## 8.1 Git and code modification

- `.git` is in this folder. Commit or push only when asked.
- Keep diffs small and scoped. With about 1,090 template files, change only the templates the task needs and never mass-reformat or rename them.
- Do not edit generated or ignored paths (`htmlcov/`, `site/`, `_site/`, `docs/docstrings/`, `.venv`).
- Do not modify application code when the task is documentation or analysis.
- Update `CHANGES.md` for user-visible behavior changes.

## 9. Before implementing a change

1. Read the module and its test file. For template work, trace the include chain from the root template (`CCD.liquid`, then `Section/`, `Entry/`, `Resource/`, `DataType/`) before editing.
2. Search for existing behavior (`Grep` for the template, filter or function name) and check `docs/test-plan.md` observations that touch it.
3. Run the full test suite to get a baseline.
4. State the plan and which files will change. Ask when the change alters output shape, ids, exit codes, public API or dependencies.
5. For output changes, convert a synthetic fixture before and after, and diff the JSON.
6. Run tests again, report results with commands, and list anything not run.
7. Do not fabricate results or citations. If something could not be verified, say so.
