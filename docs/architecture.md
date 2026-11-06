# Architecture: python-fhir-converter

Based on [repo-analysis.md](repo-analysis.md) and the source in `fhir_converter/`. Every component below exists in the code. Where the repo says nothing, this document says so instead of guessing.

## 1. Architecture in one slide

python-fhir-converter is a **template-driven transformation library with a CLI**. It is not a service. A source document is parsed into a plain dictionary, a Liquid template turns that dictionary into FHIR R4 JSON text, and a final pass cleans up and de-duplicates the result.

```
Source file -> Parse to dict -> Render Liquid template -> Clean and merge -> FHIR R4 JSON
```

The mapping logic lives in templates (about 1,090 files), not in Python. The Python code is the engine: parsing, template loading, custom Liquid filters and tags, and file handling.

## 2. Diagram index

Render the `.mmd` files with any Mermaid viewer (VS Code Mermaid extension, mermaid.live, or `mmdc`).

| Diagram | File | Shows |
|---|---|---|
| System architecture | [system-architecture.mmd](diagrams/system-architecture.mmd) | Inputs, entry points, package, libraries, output |
| Component architecture | [component-architecture.mmd](diagrams/component-architecture.mmd) | Modules, classes and their dependencies |
| Conversion flow | [fhir-conversion-flow.mmd](diagrams/fhir-conversion-flow.mmd) | Data transformation steps and branches |
| Sequence | [sequence-diagram.mmd](diagrams/sequence-diagram.mmd) | One CLI run converting a C-CDA file |
| Deployment | [deployment-diagram.mmd](diagrams/deployment-diagram.mmd) | How it runs and ships today |

## 3. Components

| Component | File | Responsibility | Depends on |
|---|---|---|---|
| CLI | `__main__.py` | Parse arguments, choose renderer by `RenderSourceFormat`, optionally add a user template directory, run file or directory conversion, print a summary, exit 0 or 1 | `renderers`, `utils`, `exceptions`, python-liquid, psutil |
| Renderer base | `renderers.py` `BaseFhirRenderer` | Public API (`render_fhir_string`, `render_fhir`, `render_to_fhir`). Wraps failures in `RenderingError`. Builds the Liquid environment if none is supplied | `make_environment`, `hl7`, `exceptions`, pyjson5 |
| C-CDA renderer | `CcdaRenderer` | Parses C-CDA XML, loads the `code_mapping` global from `ValueSet.json`, optionally prepares the narrative XSLT, renders the chosen template | `parsers`, `loaders.read_text`, `utils`, `templates/ccda` |
| STU3 renderer | `Stu3FhirRenderer` | Parses STU3 JSON (keeping empty fields) and renders the chosen template | `parsers`, `templates/stu3` |
| File helpers | `render_to_dir`, `render_files_to_dir` | Read input file, write `<name>.json`, delete a partial output on failure, mirror or flatten directories, route errors to an `onerror` handler | `utils`, `exceptions` |
| Parsers | `parsers.py` | XML to nested dict (attributes as keys, text under `_`, repeated children as lists). JSON5 parsing. Removal of empty values | lxml, pyjson5, `utils` |
| Template loaders | `loaders.py` | Load templates from a primary location plus fallbacks. Adds the `_` prefix convention so `Section/Problem` resolves to `Section/_Problem`. Optional caching (default 300 entries) | python-liquid |
| Filters | `filters.py` | Liquid functions used by templates: section lookup by templateId, code mapping (`get_property`), `batch_render`, deterministic `generate_uuid`, C#-style `date`, `transform_narrative`, others | `hl7`, `utils`, python-liquid |
| Tags | `tags.py` | `evaluate` (render a template into a variable) and `mergeDiff` (overlay rendered JSON onto an existing object, handling `[x]` choice fields) | python-liquid, pyjson5 |
| HL7 and FHIR helpers | `hl7.py` | HL7 datetime to FHIR datetime with precision, C-CDA section search, `parse_fhir` (parse output and merge duplicate bundle entries) | `parsers`, `utils` |
| Utilities | `utils.py` | String cleanup, null/blank checks, `merge_dict`, path and directory helpers, lxml and XSLT wrappers | lxml |
| Errors | `exceptions.py` | `RenderingError`, `fail` | none |
| Templates | `templates/ccda`, `templates/stu3` | Mapping logic. C-CDA: document templates, `Section/`, `Entry/`, `Resource/`, `DataType/`, `Extension/`, `Utils/`, `ValueSet/ValueSet.json`, `cda_narrative.xsl`. STU3: one template per resource type plus nested parts | python-liquid (executed by it) |

## 4. Data flow

1. **Input.** A path or text stream, plus a template name such as `CCD` or `Patient`.
2. **Parse.** C-CDA XML becomes a nested dict. The renderer adds `_originalData` (the whole source XML as a string). STU3 JSON is parsed as is.
3. **Context.** The dict is passed to the template as `msg`. For C-CDA, `code_mapping` is available as a global.
4. **Render.** The root template (for example `CCD.liquid`) includes a header and one template per clinical section. Each section finds its data by C-CDA templateId, then renders entry templates, which render FHIR resource and data type templates.
5. **Post-process.** `hl7.parse_fhir` parses the text output, drops empty values, and merges bundle entries that share resource type, version and id.
6. **Output.** Serialized with pyjson5 to a string, stream or `.json` file.

Errors at render or serialize time raise `RenderingError`. In directory mode with `--continue_on_error` the error is printed and the run continues.

## 5. Key design decisions visible in the code

- **Logic in templates, engine in Python.** Supporting a new section or resource is mainly a template change, and users can add templates with `--template-dir` without touching Python.
- **Ported behavior from the C# converter.** `evaluate`, `mergeDiff`, template naming with `_` prefixes and C# date format strings exist to keep Microsoft templates reusable.
- **Deterministic IDs.** `generate_uuid` hashes identifiers (SHA-256), so converting the same document twice gives the same resource IDs.
- **Pluggable environment.** Callers can pass their own Liquid `Environment`. Built-in filters and tags are added unless a same-named one is already registered.
- **Immutable globals.** Template globals are held in a `frozendict`.

## 6. Deployment view

The repo contains no Dockerfile, server, queue consumer or infrastructure code. The package runs inside whatever Python process invokes it.

- **As a CLI:** installed in a virtualenv, reads and writes the local file system.
- **As a library:** imported by a host application or pipeline.
- **Distribution:** PyPI package name `python-fhir-converter` (per `pyproject.toml` and README).
- **Build and docs:** GitHub Actions run lint, type check and tests (Ubuntu, Windows, macOS, Python 3.8 to 3.12 in the matrix), publish coverage to smokeshow, and build the MkDocs site.
- **Production hosting, scaling and monitoring:** **UNKNOWN**, not defined in this repo.

## 6.1 Not present

No REST API, authentication, database, message queue, logging framework configuration, FHIR server client or FHIR validation. The CLI prints a summary and tracebacks to the console only.

## 7. Risks and attention points for the architecture

| Area | Observation |
|---|---|
| Runtime version | CLI needs Python 3.11+ (`StrEnum`) while packaging and CI claim 3.8. Needs a decision: raise the minimum or change the CLI |
| Dependency pins | `lxml 5.3.0` and `pyjson5 1.6.5` do not build on Python 3.14. Newer versions passed all 475 tests locally |
| Data sensitivity | Output embeds the full source document (`_originalData`, `DocumentReference` content). Treat outputs as PHI when inputs are |
| Output correctness | No schema or profile validation. A local run on a small synthetic CCD produced `Practitioner/` with an empty id in `Composition.attester`. Not a confirmed general defect |
| Scaling | Single process, whole document held in memory. Batch parallelism, if needed, must be added by the caller. Throughput numbers are only in the README benchmark |
| Test coverage | The CLI (`__main__.py`) has no test module |

This is a technical summary of the code. It is not a security, compliance or clinical-safety assessment.
