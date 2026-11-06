# API and Interface Guide

There is no HTTP API. The interfaces are a Python library and a CLI. Everything below is from `fhir_converter/renderers.py`, `__main__.py`, `filters.py`, `tags.py` and `loaders.py`. Sample inputs and outputs are synthetic and were produced by running the code on 2026-11-04. Examples use the Windows venv: `.\.venv\Scripts\python.exe`.

## 1. Library API

### 1.1 Renderers

```python
from fhir_converter.renderers import CcdaRenderer, Stu3FhirRenderer
```

| Class | Input | Template set |
|---|---|---|
| `CcdaRenderer(env=None, template_globals=None)` | C-CDA XML | `templates/ccda` |
| `Stu3FhirRenderer(env=None)` | FHIR STU3 JSON | `templates/stu3` |

Both inherit three public methods from `BaseFhirRenderer`:

| Method | Returns | Notes |
|---|---|---|
| `render_fhir_string(template_name, data_in, encoding="utf-8")` | `str` | FHIR JSON text |
| `render_to_fhir(template_name, data_in, encoding="utf-8")` | mapping | Python dict, useful for further processing |
| `render_fhir(template_name, data_in, data_out, encoding="utf-8")` | `None` | Writes JSON to a text stream |

`data_in` is a string or file-like object. Any failure is raised as `fhir_converter.exceptions.RenderingError` ("Failed to render FHIR" or "Failed to serialize FHIR"); the original exception is in `__cause__`.

### 1.2 C-CDA example

```python
from fhir_converter.renderers import CcdaRenderer

with open("data/synthetic/ccda/TD-CCDA-01_valid_basic.ccda", encoding="utf-8") as xml_in:
    fhir_json = CcdaRenderer().render_fhir_string("CCD", xml_in)
```

Output is a Bundle with `type: "batch"`. For the synthetic input the entries are Composition, Organization, Practitioner, Patient and DocumentReference. Abbreviated Patient entry (verified output):

```json
{
  "resourceType": "Patient",
  "id": "05cadb76-f5ab-1d6a-fae5-8539f75dd453",
  "meta": {"profile": ["http://hl7.org/fhir/us/core/StructureDefinition/us-core-patient"]},
  "identifier": [{"system": "urn:oid:2.16.840.1.113883.19.5", "value": "SYN-0001"}],
  "name": [{"use": "usual", "family": "Example", "given": ["Alex"]}],
  "birthDate": "1980-01-01",
  "gender": "female",
  "address": [{"use": "home", "line": ["1 Example Street"], "city": "Exampleville",
               "state": "CA", "country": "US", "postalCode": "90000"}]
}
```

Resource IDs are UUIDs derived from a SHA-256 hash of the source identifiers, so the same input gives the same IDs. Each entry carries `request: {"method": "PUT", "url": "<Type>/<id>"}`.

### 1.3 STU3 example

```python
from fhir_converter.renderers import Stu3FhirRenderer

text = open("data/synthetic/stu3/TD-STU3-05_immunization.json", encoding="utf-8").read()
print(Stu3FhirRenderer().render_fhir_string("Immunization", text))
```

Input (synthetic):

```json
{"resourceType":"Immunization","id":"syn-i1","status":"completed",
 "date":"2024-01-15T10:30:00-05:00",
 "vaccineCode":{"coding":[{"system":"http://hl7.org/fhir/sid/cvx","code":"140","display":"Influenza, seasonal"}]},
 "patient":{"reference":"Patient/syn-p1"},"wasNotGiven":false,"primarySource":true}
```

Output (verified):

```json
{"resourceType":"Immunization","id":"syn-i1","status":"completed",
 "vaccineCode":{"coding":[{"system":"http://hl7.org/fhir/sid/cvx","code":"140","display":"Influenza, seasonal"}]},
 "patient":{"reference":"Patient/syn-p1"},"wasNotGiven":false,"primarySource":true,
 "occurrenceDateTime":"2024-01-15T10:30:00.000-05:00"}
```

Observed behavior: `date` became `occurrenceDateTime` and gained milliseconds, while the STU3-only field `wasNotGiven` was passed through unchanged.

### 1.4 Options

Narrative text. With `render_narrative`, each section's text is transformed with `templates/ccda/cda_narrative.xsl` into `Composition.section.text`:

```python
CcdaRenderer(template_globals={"render_narrative": True})
# section text: {'div': '<div xmlns="http://www.w3.org/1999/xhtml">No known allergies</div>',
#                '_status': {...data-absent-reason unknown...}}
```

Custom templates. Build an environment that searches your folder first, then the packaged templates:

```python
from liquid import FileExtensionLoader
from fhir_converter.renderers import CcdaRenderer, make_environment

env = make_environment(
    loader=FileExtensionLoader(search_path=r"C:\my\templates"),
    additional_loaders=[CcdaRenderer.defaults()["loader"]],
)
CcdaRenderer(env).render_fhir_string("Mini", xml_text)
```

Verified with a one-line template `Mini.liquid`, which produced a `Basic` resource whose id came from `msg.ClinicalDocument.id.extension`.

Custom filters and tags: register them on your own `Environment` before passing it in. Built-ins from `all_filters` and `all_tags` are added unless a same-named one already exists.

### 1.5 File and directory helpers

| Function | Purpose |
|---|---|
| `render_to_dir(render, from_file, to_dir, extension=".json", encoding="utf-8", onerror=fail)` | One file to `<to_dir>/<name>.json`. Deletes a partial output file on failure |
| `render_files_to_dir(render, from_dir, to_dir, flatten=False, ..., path_filter=None)` | Walks a directory, mirrors sub-folders unless `flatten`, removes empty output folders afterwards |

`render` is a `DataRenderer`: `Callable[[data_in, data_out, encoding], None]`, for example `functools.partial(renderer.render_fhir, "CCD")`. `onerror` receives a `RenderingError`; the default `fail` re-raises it.

## 2. Command line interface

Invoke as `python -m fhir_converter` or `fhir_converter_cli`.

| Argument | Required | Description |
|---|---|---|
| `--from-file <Path>` | one of the two | File to convert. Ignored when `--from-dir` is given |
| `--from-dir <Path>` | one of the two | Directory to convert. Only `.ccda` and `.xml` files are read |
| `--to-dir <Path>` | yes | Output directory. Created if missing, removed again if empty after a failure |
| `--template-name <str>` | yes | Template to render with |
| `--source-format {STU3,CCDA}` | yes | Source format |
| `--template-dir <Path>` | no | Extra template folder searched first |
| `--continue_on_error` | no | Print the traceback and carry on with the next file |
| `--flatten_to_dir` | no | Write all outputs directly into `--to-dir` |

Exit codes (verified): 0 success, 1 conversion failure, 2 argument error.

Sample C-CDA run:

```powershell
.\.venv\Scripts\python.exe -m fhir_converter --from-file .\data\synthetic\ccda\TD-CCDA-01_valid_basic.ccda --to-dir .\data\out --template-name CCD --source-format CCDA
```

```
--------------------------------------------------------------------------------
RENDER SUCCESS
--------------------------------------------------------------------------------
Total time: 1.49s
Finished at: 2026-11-04 18:08:53.958867
Final Memory: 42M
--------------------------------------------------------------------------------
```

Behaviors worth knowing (all observed):
- With `--continue_on_error`, a run where one file fails still prints `RENDER SUCCESS` and exits 0. Check the output count or the traceback text to detect failures.
- `--from-dir` with `--source-format STU3` converts nothing, prints `RENDER SUCCESS`, and exits 0, because JSON files are filtered out.
- Without `--continue_on_error`, a directory run stops at the first failing file. Files already written stay on disk.

## 3. Template language extensions

Tags (`tags.py`):

| Tag | Syntax | Effect |
|---|---|---|
| `evaluate` | `{% evaluate patientId using 'Utils/GenerateId' obj: msg.ClinicalDocument.recordTarget.patientRole -%}` | Renders a template and stores trimmed output in a variable |
| `mergeDiff` | `{% mergeDiff msg -%} { ... } {% endmergeDiff -%}` | Renders JSON and overlays it on the named object. Keys ending `[x]` replace the matching choice field |

Filters (`filters.py`, `all_filters`):

| Filter | Use |
|---|---|
| `to_json_string`, `to_array` | Serialize or wrap values |
| `generate_uuid` | Deterministic UUID from a string. Blank input gives `""` |
| `get_property: mapping_key, property` | Look up `code_mapping`. Keys look like `ValueSet/Gender`. Property defaults to `code` |
| `get_ccda_section_by_template_id`, `get_first_ccda_sections_by_template_id` | Find sections by templateId OID |
| `batch_render: 'Template', 'argName'` | Render a template for each item in a list |
| `date`, `format_as_date_time`, `add_hyphens_date` | Date handling (C# style format strings in `date`) |
| `match`, `gzip`, `sha1_hash`, `now`, `transform_narrative` | Utilities |

Template names: a name with a folder gets an `_` prefix on its file (`Section/Problem` loads `Section/_Problem.liquid`). Names ending `.json` and names already starting `_` are left alone.

## 4. Error reference (all observed)

| Situation | Exception chain |
|---|---|
| Empty XML | `RenderingError` caused by lxml `XMLSyntaxError: Document is empty` |
| Malformed XML | `RenderingError` caused by `XMLSyntaxError` (tag mismatch) |
| Malformed or empty JSON | `RenderingError` caused by `Json5EOF` |
| Unknown template name | `RenderingError` caused by `TemplateNotFound` |
| Malformed HL7 date in a C-CDA field | `RenderingError` caused by `liquid.exceptions.Error: filter 'format_as_date_time': unexpected error: Malformed HL7 datetime <value>` |

Not an error, but surprising: XML that is not a C-CDA (for example `<note>`) still renders. The result is a Bundle with a Composition containing empty references such as `Patient/`.

No validation against FHIR R4 profiles is performed on the output.

This guide is a technical summary of the code and not a substitute for review against approved specifications.
