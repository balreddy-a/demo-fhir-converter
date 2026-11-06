# Technical Glossary

Plain-English definitions for this project. "In this repo" notes point to where the idea appears. Definitions of external standards are general descriptions, not quotations from the specifications.

## Healthcare data standards

| Term | Meaning | In this repo |
|---|---|---|
| **HL7** | Health Level Seven, the standards body behind several healthcare data formats | `hl7.py` holds HL7 date and C-CDA helpers |
| **FHIR** | Fast Healthcare Interoperability Resources. A standard for exchanging health data as JSON or XML "resources" | Output format of the converter |
| **FHIR R4** | Release 4 of FHIR. The target version here | All outputs |
| **STU3** | An older FHIR release (Standard for Trial Use 3) | Input format for `Stu3FhirRenderer`, converted to R4 |
| **CDA** | Clinical Document Architecture. An XML format for clinical documents | Input document structure |
| **C-CDA** | Consolidated CDA. A set of CDA document types and templates used widely in the US | Main input (`CcdaRenderer`) |
| **CCD** | Continuity of Care Document. A C-CDA summary document type | Template `CCD.liquid` |
| **Resource** | One FHIR object such as Patient or Composition, with a `resourceType` and `id` | Created by templates in `Resource/` |
| **Bundle** | A FHIR resource that holds a list of other resources in `entry` | C-CDA output is a Bundle of `type: batch` |
| **Bundle type `batch`** | A set of independent actions, each with a `request` | Each entry has `request: PUT <Type>/<id>` |
| **Composition** | FHIR resource describing a document, its sections and author | Created from the CDA header and sections |
| **DocumentReference** | FHIR resource pointing at a document | Holds a compressed copy of the source CDA |
| **Patient / Practitioner / Organization** | FHIR resources for a person receiving care, a clinician, and a body such as a clinic | Created from `recordTarget`, `author`, `custodian` |
| **recordTarget** | CDA element holding the patient | A missing one means no Patient entry |
| **templateId (OID)** | A dotted number that identifies a C-CDA section or entry type, for example `2.16.840.1.113883.10.20.22.2.6.1` for allergies | Filters find sections by OID |
| **nullFlavor** | CDA attribute saying a value is missing and why (for example `UNK` unknown) | Treated as no value, so the field is omitted |
| **LOINC** | A code system for clinical observations and document types | `http://loinc.org` appears in output |
| **NPI** | US National Provider Identifier | `http://hl7.org/fhir/sid/us-npi` on Practitioner |
| **Value set / code mapping** | A table that translates source codes to FHIR codes (for example `F` to `female`) | `templates/ccda/ValueSet/ValueSet.json` |
| **Narrative** | Human-readable text of a section | Optional XSLT conversion, `render_narrative` |
| **PHI** | Protected health information. Any health data that identifies a person | Real C-CDA files usually contain it. Only synthetic data is used in tests |
| **Synthetic data** | Invented data with no link to real people | `data/synthetic/` |
| **Profile (US Core)** | Extra rules layered on a FHIR resource | Some output carries `meta.profile` for US Core. No profile validation is done |

## Conversion engine

| Term | Meaning | In this repo |
|---|---|---|
| **Renderer** | Class that parses input, runs a template and returns FHIR | `CcdaRenderer`, `Stu3FhirRenderer`, base `BaseFhirRenderer` |
| **Template** | A Liquid text file that describes the FHIR JSON to produce | `fhir_converter/templates/` (about 1,090 files) |
| **Liquid** | A template language with `{{ }}` for values and `{% %}` for logic | Runs on python-liquid |
| **python-liquid** | The Python library that runs Liquid templates | Pinned to 1.12.1 |
| **Filter** | A function applied in a template with a pipe character, for example `{{ x \| to_array }}` | `filters.py` |
| **Tag** | A block or command in a template, for example `include` | `tags.py` adds `evaluate` and `mergeDiff` |
| **`include`** | Liquid tag that renders another template in the same scope | Section templates include entry and resource templates |
| **`evaluate`** | Project tag: render a template and store the result in a variable | Used for id generation |
| **`mergeDiff`** | Project tag: overlay changed fields onto an existing object | Used in STU3 templates such as `Patient.liquid` |
| **`batch_render`** | Filter that renders a template for every item in a list | Used for each entry in a section |
| **Environment** | The python-liquid object that holds loaders, filters and tags | `make_environment()` |
| **Loader** | Finds template text by name | `TemplateSystemLoader`, `PackageLoader` |
| **`_` prefix convention** | Templates in sub-folders are stored as `_Name.liquid` and referenced as `Folder/Name` | `loaders.py` |
| **Global** | A value available to every template | `code_mapping`, `render_narrative`, `narrative_xslt` |
| **`msg`** | The name of the parsed input inside a template | `{{ msg.ClinicalDocument... }}` |
| **`_originalData`** | Extra key holding the source document as text | Added by `CcdaRenderer` |
| **Section / Entry / Resource / DataType templates** | Layers of C-CDA templates: a section finds data, an entry handles one item, a resource builds the FHIR object, a data type builds small parts such as an address | `templates/ccda/Section`, `Entry`, `Resource`, `DataType` |
| **`parse_fhir`** | Post-processing step: parse template output, drop empty values, merge duplicate bundle entries | `hl7.py` |
| **Deterministic id** | An id computed from the data, so the same input always gives the same id | `generate_uuid` (SHA-256 based) |
| **Precision (dates)** | How much of a date is known: year, month, day or time | `Hl7DtmPrecision`, `FhirDtmPrecision` |
| **DTM** | HL7 date-time value such as `20240115103000-0500` | `hl7_to_fhir_dtm()` |

## Libraries and formats

| Term | Meaning |
|---|---|
| **lxml** | Python XML library. Parses the CDA and runs XSLT |
| **XSLT** | A language that transforms XML. Used for the narrative (`cda_narrative.xsl`) |
| **JSON5** | A relaxed JSON syntax. Read and written with **pyjson5** |
| **frozendict** | A dictionary that cannot be changed. Holds template globals |
| **psutil** | Library used to print memory and time in the CLI summary |
| **zlib** | A compression format. The `gzip` filter actually uses it |
| **base64** | A way to write binary data as text. Used for the attachment data |

## Running and testing

| Term | Meaning |
|---|---|
| **CLI** | Command line interface: `python -m fhir_converter` or `fhir_converter_cli` |
| **`--continue_on_error`** | Flag to keep going after a failed file. Does not change the final success banner |
| **`--flatten_to_dir`** | Flag to write all outputs into one folder |
| **`RenderingError`** | Project exception for any conversion failure. The original error is in `__cause__` |
| **venv** | A Python virtual environment, here `.venv` |
| **Poetry** | Python dependency tool used by the project's Makefile and CI |
| **Pin** | Exact version requirement such as `lxml==5.3.0` |
| **Wheel** | Prebuilt Python package. Missing wheels force a slow source build that can fail |
| **pytest** | Python test runner. 475 tests in `tests/` |
| **Fixture (test data)** | A fixed input file used by tests, here the `TD-...` files |
| **Test ID** | Label such as `TC-DT-02` (case) or `TD-CCDA-05` (data) |
| **Regression baseline** | A recorded set of results used to spot unintended changes later |
| **Observation (OBS)** | A behavior noticed in testing that may or may not be a defect |
| **Exploratory run** | Trying the system to learn its behavior, not a formal test |
| **Mermaid** | Text-based diagram format used in `docs/diagrams/*.mmd` |
