# Repository Analysis: python-fhir-converter

Scope: static analysis of the source in this clone. Version per `pyproject.toml`: 0.3.0. Items marked **UNKNOWN** could not be confirmed from the code in this repo. This is an engineering summary, not a compliance or clinical validation document.

## 1. Project purpose and business use case

A Python port of Microsoft's C# FHIR-Converter (README.md). It converts healthcare documents into FHIR R4 using Liquid templates, so the conversion can run inside Python data pipelines.

Supported conversions (from `fhir_converter/renderers.py` and `__main__.py`):

| Source | Renderer | Output |
|---|---|---|
| Consolidated CDA (`.xml`, `.ccda`) | `CcdaRenderer` | FHIR R4 Bundle (type `batch`) |
| FHIR STU3 JSON | `Stu3FhirRenderer` | FHIR R4 resource or bundle |

README states JSON->FHIR, HL7v2->FHIR and FHIR->HL7v2 are not supported. Typical use: ingest C-CDA exports (CCD, discharge summary, H&P, etc.) from EHRs and produce FHIR R4 for downstream systems. The consuming business context is **UNKNOWN** (not in the repo).

## 2. Technology stack and dependencies

Pinned in `pyproject.toml` (Poetry, `poetry_core` build backend):

| Package | Pin | Role (from imports) |
|---|---|---|
| python-liquid | 1.12.1 | Template engine, loaders, custom tags/filters |
| lxml | 5.3.0 | XML parsing, XSLT (`utils.parse_etree`, `load_xslt`) |
| pyjson5 | 1.6.5 | JSON5 decode and JSON encode |
| xmltodict | 0.13.0 | Declared. No import found in `fhir_converter/` (**UNKNOWN** if used elsewhere) |
| frozendict | 2.3.10 | Immutable globals and maps |
| importlib_resources | 6.1.1 | Declared. Not imported in the modules read |
| psutil | 5.9.7 | CLI summary (memory, process start time) |
| isodate | 0.6.1 | Date handling (mypy config references it) |
| typing-extensions | 4.12.2 | `TypeAlias` |

Dev: pytest 7.4.4, pytest-mock, pytest-cov, mypy 1.8.0, flake8 7.0.0, lazydocs, mkdocs plugins, smokeshow, watchdog.

Python: `pyproject.toml` says `^3.8.1`, but `__main__.py` imports `enum.StrEnum` (3.11+). Makefile targets 3.12.

## 3. Folder structure and important files

```
fhir-converter/
  fhir_converter/            package
    __main__.py              CLI
    renderers.py             renderers, environment factory, file/dir helpers
    parsers.py               XML->dict, JSON5 parse, empty-value pruning
    loaders.py               TemplateSystemLoader (template name resolution)
    filters.py               custom Liquid filters
    tags.py                  custom Liquid tags (evaluate, mergeDiff)
    hl7.py                   HL7 datetime, C-CDA section lookup, bundle dedupe
    utils.py                 string, path, etree, XSLT helpers
    exceptions.py            RenderingError, fail
    templates/ccda/          822 files
    templates/stu3/          265 files
    __init__.py              empty
  tests/                     7 test modules + tests/data
  scripts/                   examples.py, benchmark.py
  data/                      sample/ccda, sample/stu3, templates/ccda, out
  docs/ , mkdocs.yml         MkDocs site (README include)
  .github/workflows/         python-package.yml, mkdocs-gh-pages.yml, greetings.yml
  Makefile, pyproject.toml, mypy.ini, poetry.toml, CHANGES.md
```

`poetry.toml` sets `virtualenvs.in-project = true`. No `poetry.lock` was seen in the listing (**UNKNOWN** whether intentionally excluded).

## 4. Main classes, functions and modules

**renderers.py**
- `BaseFhirRenderer` (ABC): `render_fhir_string()`, `render_fhir()`, `render_to_fhir()` (wraps errors in `RenderingError`), abstract `_render()` and `defaults()`.
- `CcdaRenderer`: `_make_globals()` loads `ValueSet/ValueSet.json` into the `code_mapping` global; optional narrative XSLT when `render_narrative` global is truthy; `_parse_cda()`; `_render()`.
- `Stu3FhirRenderer`: `_parse_stu3()`, `_render()`.
- `make_environment()`: builds a Liquid `Environment` with the template system loader and registers all filters (replacing same-named Liquid builtins) and tags.
- `render_to_dir()`, `render_files_to_dir()`: file and directory output (`.json` extension, optional `flatten`, `onerror` handler).
- Module loaders: `ccda_default_loader`, `stu3_default_loader` (`PackageLoader` on `fhir_converter.templates`).

**parsers.py**: `parse_xml()`, `parse_json()` (JSON5, prunes empty values unless `ignore_empty_fields=False`), `_xml_element_to_dict()`, `parse_xml_filter()`, `ParseXmlOpts`. XML attributes become keys, text goes under `_`, repeated children become lists, namespace prefixes are joined with `_`.

**loaders.py**: `TemplateSystemLoader`, `CachingTemplateSystemLoader`, `make_template_system_loader()`, `read_text()`. Subdirectory template names get an `_` prefix (`Section/Problem` -> `Section/_Problem`).

**filters.py** (`all_filters`): `to_json_string`, `to_array`, `match`, `gzip`, `sha1_hash`, `add_hyphens_date`, `format_as_date_time`, `now`, `date` (C# format string translation), `generate_uuid` (first 16 bytes of SHA-256 as UUID), `get_property` (code mapping lookup), `get_first_ccda_sections_by_template_id`, `get_ccda_section_by_template_id`, `batch_render`, `transform_narrative`.

**tags.py**: `EvaluateTag`/`EvaluateNode` (`{% evaluate x using 'T' a: b %}`, captures rendered output into a variable), `MergeDiffTag`/`MergeDiffNode` (`{% mergeDiff msg %}...{% endmergeDiff %}`, merges rendered JSON over the original object, with `[x]` choice-type key handling). Both ported from the C# converter's semantics.

**hl7.py**: `parse_hl7_dtm()`, `hl7_to_fhir_dtm()`, `to_fhir_dtm()`, `Hl7DtmPrecision`, `FhirDtmPrecision`; `parse_fhir()` (parses template output and merges duplicate bundle entries via `get_fhir_entry_key()` + `utils.merge_dict`); `get_ccda_section()`, `get_ccda_component3()`, `is_template_id()`, `get_template_id_key()`.

**utils.py**: `sanitize_str`, `is_undefined_none_or_blank`, `to_list_or_empty`, `merge_dict`, `read_text`, `parse_etree`, `etree_to_str`, `load_xslt`, `transform_xml_str`, `mkdir`, `del_path_quietly`, `walk_path`, `tail`.

**exceptions.py**: `RenderingError(msg, cause)`, `fail(e)`.

## 5. FHIR conversion workflow

C-CDA path (`CcdaRenderer._render`):

1. `_parse_cda()` -> `parse_xml()` turns the XML into a nested dict. `after_parse_ccda` adds `_originalData` (whole document as a sanitized string). With `render_narrative`, each section `text` element is replaced by `{"_originalData": <section xml>}`.
2. `env.get_template(template_name, globals=template_globals)` loads e.g. `CCD.liquid`.
3. `template.render({"msg": <dict>})` produces a JSON string.
4. `hl7.parse_fhir()` parses it (JSON5, empty pruning) and de-duplicates bundle entries sharing resourceType + versionId + id.
5. `render_fhir()` serializes with `pyjson5.encode_io`.

STU3 path is the same except step 1 is `parse_json(..., ignore_empty_fields=False)` and no globals are passed.

CLI wrapper (`__main__.py`): `parse_args` -> `get_renderer` -> `render` -> `render_to_dir` or `render_files_to_dir` -> `print_summary` and exit code 0/1.

## 6. Templates and mapping logic

**C-CDA templates** (`fhir_converter/templates/ccda/`):
- Root document templates: `CCD`, `ConsultationNote`, `DischargeSummary`, `HistoryandPhysical`, `OperativeNote`, `ProcedureNote`, `ProgressNote`, `ReferralNote`, `TransferSummary`, plus `Header` and `cda_narrative.xsl`. `metadata.json` is `{"type": "ccda"}`.
- Subfolders: `Section/` (30), `Entry/` (per-entry templates, nested), `Resource/` (28), `DataType/` (7), `Extension/` (Race, Ethnicity), `Utils/` (`_GenerateId`, `_SystemReference`), `ValueSet/ValueSet.json`.
- Pattern: `CCD.liquid` builds a Bundle by `include`-ing `Header` and each `Section/*`. A section template (e.g. `Section/_Problem.liquid`) finds its section with `get_ccda_section_by_template_id` using C-CDA templateId OIDs, then `batch_render`s `Entry/...` templates, which include `Resource/*` and `DataType/*`.
- IDs: `Utils/_GenerateId` hashes `root|extension` (or `root`) with `generate_uuid`, so IDs are deterministic.
- Code and status mapping: `ValueSet/ValueSet.json` (`Mapping` key) is exposed as `code_mapping` and read through the `get_property` filter. Falls back to the original code when unmapped.
- Not in `CCD.liquid`'s includes but present in other root templates: Goal, HealthConcern, Instruction and procedure/diagnosis sections (**UNKNOWN** exact per-template coverage; not traced).

**STU3 templates** (`templates/stu3/`): one file per resource type at the root (e.g. `Patient.liquid`), plus subfolders for data types and nested pieces (e.g. `Patient/Animal`). `Patient.liquid` uses `{% mergeDiff msg %}` to override only the changed fields on the input resource.

**User templates**: `--template-dir` adds a `FileExtensionLoader` ahead of the packaged loader, so custom templates (see `data/templates/ccda/LabsandVitals.liquid`, `Pampi.liquid`) can include the packaged ones.

## 7. APIs and execution entry points

- No HTTP/REST service, queue consumer or server code found.
- CLI: `fhir_converter_cli` (Poetry script -> `fhir_converter.__main__:entrypoint`) or `python -m fhir_converter`.
  Args: `--from-file`, `--from-dir`, `--to-dir` (required), `--template-name` (required), `--source-format {STU3,CCDA}` (required), `--template-dir`, `--continue_on_error`, `--flatten_to_dir` (note underscore spelling).
  Directory mode filters to `.ccda` and `.xml` only (`is_ccda_file`), so STU3 JSON cannot be batch-converted by directory.
- Library API: `CcdaRenderer().render_fhir_string("CCD", xml_in)`, `render_to_fhir()`, `render_fhir()`.
- Scripts: `scripts/examples.py`, `scripts/benchmark.py` (expect `data/sample`, `data/templates`, `data/out`).

## 8. Existing tests and configuration

Tests (pytest, collected counts from the 2026-11-04 run): `test_filters.py` 131, `test_hl7.py` 92, `test_utils.py` 105, `test_parsers.py` 59, `test_renderers.py` 44, `test_tags.py` 28, `test_loaders.py` 16. Total 475 (parametrized tests make this higher than the number of `def test_` functions); all passed in a local run on Python 3.14 with newer dependency versions than the pins. Fixtures in `tests/data` (`CCD.ccda`, `History_and_Physical.ccda`, `sample.ccda`, `empty.*`, `simple.*`).

Not covered by a test module: `__main__.py` (CLI) and the template content itself beyond what `test_renderers.py` exercises (**UNKNOWN** depth; not read).

Config:
- `Makefile`: `env`, `install`, `test`, `cov`, `type-check`, `linting-check`, `docs`, `build`.
- `mypy.ini`: `warn_redundant_casts`, `warn_unreachable`; ignores missing stubs for `isodate`, `dateutil`.
- CI `python-package.yml`: flake8 -> mypy -> pytest matrix (ubuntu/windows/macos x Python 3.8 to 3.12) -> coverage upload (smokeshow, threshold 10).
- `mkdocs-gh-pages.yml`: docs publishing (not read in detail).
- Runtime configuration beyond CLI flags and renderer `template_globals`: none found. No env vars read in the modules inspected.

## 9. Technical limitations and observations

1. **Python version mismatch.** `StrEnum` in `__main__.py` requires 3.11+, while `pyproject.toml` and CI claim 3.8 support. The CLI would fail on import under 3.8 to 3.10. Library modules may still work (**UNKNOWN**, CLI is untested).
2. **Pinned versions do not build on Python 3.14** (lxml 5.3.0, pyjson5 1.6.5 need source builds). Newer versions worked locally with all tests passing.
3. **Directory mode is C-CDA only** (`is_ccda_file`), and output is always `.json`.
4. **Template syntax differences from the C# project** (README): commas required between parameters, explicit variable names when passing to snippets. Templates copied from Microsoft's repo may need edits.
5. **Date formats**: C# format strings are only partly supported in the `date` filter.
6. **Entire document is embedded.** `_originalData` holds the full source XML as a string, and the CCD template base64-encodes a zlib-compressed copy of the document into a `DocumentReference` (the filter is named `gzip` but uses `zlib.compress`). Output bundles therefore contain the complete source document, which matters for PHI handling.
7. **Error handling**: failures are wrapped in `RenderingError`. With `--continue_on_error` the traceback is printed and processing continues, so partial output sets are possible.
8. **Observed output gaps** (from one local run on a minimal synthetic CCD, not a general finding): `Composition.attester.party.reference` came out as `Practitioner/` with no id.
9. **Unused or unclear dependencies**: `xmltodict` and `importlib_resources` are declared but not imported in `fhir_converter/`.
10. **Validation**: no FHIR R4 schema or profile validation step exists in the code. Output conformance is **UNKNOWN**.
11. **Sample data**: files under `data/sample` and `tests/data` are named like real C-CDA exports. Provenance and whether they contain real PHI is **UNKNOWN**. Confirm before sharing outputs.
