# Day 2 Discovery: Template Management UI and a Missing C-CDA Segment

Status: discovery only. No application code, template, test, dependency or configuration file was changed. The only file created in the repository is this one.

Audience: the developers who own `python-fhir-converter` (v0.3.0).

## How to read this document

Every statement is tagged.

| Tag | Meaning |
|---|---|
| **Verified** | Read directly from a source file in this repository, or observed by running the existing converter on synthetic data. The path (and line, where useful) is given. |
| **Assumption** | Not confirmed in this repository. Needs a check before it is relied on. |
| **Not run** | Something this discovery did not execute. |

This is an engineering analysis, not a conformance or compliance statement. Nothing here claims that any output is FHIR-valid or US Core conformant. No validator is run by the project (CLAUDE.md section 8).

Not run in this discovery: the pytest suite (CLAUDE.md expects 475 passed), flake8, mypy. The only execution was a read-only probe with synthetic input and temporary templates kept outside the repository (see section C.5).

---

## A. Current architecture

### A.1 How templates are stored, loaded, selected and rendered

| Step | What happens | Evidence |
|---|---|---|
| Storage | Liquid templates ship inside the package: `fhir_converter/templates/ccda/` and `fhir_converter/templates/stu3/`. 1,083 `.liquid` files in total. | **Verified** (file count of `fhir_converter/templates`) |
| Default loaders | `PackageLoader(package="fhir_converter.templates", package_path="ccda")` and the same for `stu3`. | **Verified** `fhir_converter/renderers.py:59-67` |
| Name resolution | A name with a folder gets `_` prepended to the last segment unless it ends in `.json` or already starts with `_`. `Section/Problem` loads `Section/_Problem.liquid`. Root names (`CCD`) are not prefixed. | **Verified** `fhir_converter/loaders.py:73-80` |
| Environment | `make_environment(loader, auto_reload, cache_size, additional_loaders)` wraps the loaders in `TemplateSystemLoader` (a python-liquid `ChoiceLoader`), or `CachingTemplateSystemLoader` when `cache_size > 0`. Registers builtin filters and tags. | **Verified** `renderers.py:313-348`, `loaders.py:106-142` |
| Selection | The caller passes a template name string. There is no registry, listing or discovery function. A name that does not exist raises `TemplateNotFound`, wrapped as `RenderingError`. | **Verified** (no listing code in `loaders.py` or `renderers.py`; `docs/api-guide.md:188`) |
| C-CDA render | Parse XML to a dict (`msg`), add `_originalData`, render the template, then `parse_fhir` parses the text and merges duplicate bundle entries. | **Verified** `renderers.py:243-285`, `hl7.py:227-252` |
| STU3 render | Same shape, input parsed with `parse_json(..., ignore_empty_fields=False)`, no template globals. | **Verified** `renderers.py:300-310` |
| Code mappings | `ValueSet/ValueSet.json` is read through the loader and exposed to templates as the `code_mapping` global. | **Verified** `renderers.py:227-241` |
| Root templates (C-CDA) | `CCD`, `ConsultationNote`, `DischargeSummary`, `HistoryandPhysical`, `OperativeNote`, `ProcedureNote`, `ProgressNote`, `ReferralNote`, `TransferSummary`. `Header` is include-only. | **Verified** directory listing and include search |
| Root templates (STU3) | One `.liquid` per resource type at the top of `templates/stu3/` (110 files), plus subfolders for nested pieces. | **Verified** |
| `metadata.json` | `{"type": "ccda"}` exists in `templates/ccda`, and a `metadata.json` exists in `templates/stu3`. The only reader found is a test. | **Verified** `tests/test_loaders.py:139`. No use in package code (grep). The STU3 content was not read. |

### A.2 CLI and extension points

| Item | Detail | Evidence |
|---|---|---|
| CLI entry | `fhir_converter_cli` (Poetry script) or `python -m fhir_converter`. | **Verified** `pyproject.toml:67-68`, `__main__.py:212-217` |
| CLI arguments | `--from-file`, `--from-dir`, `--to-dir` (required), `--template-name` (required), `--source-format {STU3,CCDA}` (required), `--template-dir`, `--continue_on_error`, `--flatten_to_dir`. | **Verified** `__main__.py:131-198` |
| Custom templates | `--template-dir` builds `make_environment(loader=FileExtensionLoader(search_path=...), additional_loaders=[packaged loader])`. The user directory is searched first, the packaged templates second. | **Verified** `__main__.py:86-94` |
| Library API | `CcdaRenderer(env, template_globals)`, `Stu3FhirRenderer(env)`, methods `render_fhir_string`, `render_to_fhir`, `render_fhir`. | **Verified** `renderers.py` |
| Custom filters, tags | Consumers register extras on the environment passed to a renderer. Builtins are added unless a name is already taken. | **Verified** `renderers.py:76-110`, `tags.py`, `filters.py` |
| Custom template examples | `data/templates/ccda/LabsandVitals.liquid`, `Pampi.liquid`, `data/synthetic/templates/Mini.liquid`. | **Verified** directory listing |
| Known CLI behaviors | `--from-dir` reads only `.ccda` and `.xml`. `--continue_on_error` still prints `RENDER SUCCESS` and exits 0. | **Verified** `__main__.py:127-128`, CLAUDE.md section 5 |

### A.3 Is there a frontend, web server or API?

No. **Verified** by:
- `find` for `package.json`, `*.html`, `*.js`, `*.tsx` outside `.venv`, `site`, `htmlcov`: no results.
- No `flask`, `fastapi`, `django`, `starlette`, `uvicorn`, `streamlit`, `gradio`, `jinja2`, `tornado`, `aiohttp` or `bottle` in `.venv/Lib/site-packages`.
- `docs/repo-analysis.md` section 7: "No HTTP/REST service, queue consumer or server code found."
- `pyproject.toml` dependencies contain no web framework.

### A.4 Reusable components and dependencies

| Component | Reusable for a UI | Evidence |
|---|---|---|
| `renderers.make_environment`, `CcdaRenderer.defaults()`, `Stu3FhirRenderer.defaults()` | Yes. Build the same environment the CLI builds. | **Verified** |
| `loaders.TemplateSystemLoader._resolve_template_name` | Yes. Single source of truth for name to file mapping. It is a private method. | **Verified** `loaders.py:73` |
| `loaders.read_text(env, filename)` | Yes. Reads a template source through the loader. | **Verified** `loaders.py:145-162` |
| `exceptions.RenderingError`, `fail` | Yes. The UI must translate errors (see B.6). | **Verified** |
| `__main__.get_user_defined_environment` | Logic is reusable but lives in the CLI module. It has no unit tests. | **Verified** `__main__.py:86-94`, CLAUDE.md section 6 |
| Dependencies | `python-liquid 1.12.1`, `lxml 5.3.0`, `pyjson5 1.6.5`, `frozendict 2.3.10`, `psutil 5.9.7`, `isodate 0.6.1`, `xmltodict 0.13.0` (declared, not imported), `importlib_resources 6.1.1` (declared, not imported), `typing-extensions 4.12.2`. | **Verified** `pyproject.toml:39-49`, CLAUDE.md section 7 |
| Constraints | Library modules must stay Python 3.8 compatible. `__main__.py` already uses `StrEnum` (3.11+). Do not add dependencies without asking. | **Verified** CLAUDE.md sections 4 and 7 |

---

## B. Template Management UI

### B.1 Recommendation in one paragraph

Build a **local, read-only template browser** first. It lists the packaged templates and the templates in an optional user directory, shows a template's source and its include relationships, and checks that a name resolves. It does not edit, upload or delete anything. It is a thin layer over the existing loaders, with a small, separately testable catalog module that does all path handling. A render preview with user-supplied input is a separate, later phase because converter output embeds the whole source document (CLAUDE.md section 3 and 8).

### B.2 Capability scope

| Capability | Phase | Safe with existing architecture? | Notes |
|---|---|---|---|
| View available templates (package and user directory) | 1 | Yes | Needs a new filesystem listing. The loaders cannot list (A.1). |
| Select a template and view details (source, size, type, includes, which root templates use it) | 1 | Yes | Read only. |
| Check that a template name resolves and its source compiles | 1 | Yes, with a caveat | Use `env.get_template(name)`. **Assumption**: python-liquid parses on load, so this catches syntax errors but not a missing include, which resolves at render time. Confirm before promising it in the UI. |
| Show which sections a root template includes | 1 | Yes | Parse `{% include 'Section/...' %}` lines. **Assumption**: a regex over single-quoted includes is enough (all includes seen use single quotes). |
| Render preview with a user-supplied file | 2, needs approval | Yes technically | Privacy and error handling work required (B.6, D.4). |
| Manage custom templates (list a configured user directory) | 1 (read only) | Yes | Matches the only supported mechanism, `--template-dir`. |
| Create, edit, upload, delete custom templates | Not recommended | Not proven safe | Writing into a template directory changes conversion output and ids for every consumer, and the loader cache has `auto_reload` off by default (`make_environment`, `renderers.py:315`). The caching layer would also need invalidation. Do not build this without an explicit decision. |

### B.3 Architecture

```
Browser (static HTML + JS)
        |  HTTP, loopback only, JSON
        v
ui/server (thin request handling, input validation, error mapping)
        |
        v
ui/catalog (NEW: lists templates, validates names, reads sources, no writes)
        |
        v
fhir_converter.renderers / loaders (EXISTING: make_environment, read_text, defaults())
        |
        v
templates/ccda, templates/stu3 (package)   +   optional --template-dir (user)
```

Two ways to host the server. This is a decision for the developers, because the second adds dependencies.

| Option | Pros | Cons |
|---|---|---|
| 1. Python standard library (`http.server`) plus static HTML and vanilla JS | No new dependencies. No change to pins. Fits "do not add dependencies without asking". | Hand-written routing and no built-in schema validation. Fine for 4 or 5 read-only endpoints. |
| 2. A web framework with an ASGI server | Better validation, docs and testability for a larger API. | Adds runtime dependencies to a library that currently has none of this. Needs approval. Needs a compatibility check on the Python 3.8 to 3.12 CI matrix. |

Recommendation: **Option 1** for Phase 1, with the catalog written so a framework can replace the server later without touching it.

Proposed read-only endpoints (design only, none exist today):

| Endpoint | Returns |
|---|---|
| `GET /api/formats` | `CCDA`, `STU3` (mirrors `RenderSourceFormat`, `__main__.py:32-36`) |
| `GET /api/templates?format=CCDA` | Name, kind (root, section, entry, resource, datatype, valueset, other), origin (package or user), size |
| `GET /api/templates/{format}/{name}` | Source text, includes found, used-by list, resolved logical file name |
| `POST /api/templates/{format}/{name}/check` | Resolves and compiles the template, returns pass or a sanitized error |

### B.4 Reuse and new components

| Existing file | Reuse |
|---|---|
| `fhir_converter/renderers.py` | `make_environment`, `ccda_default_loader`, `stu3_default_loader`, `defaults()` |
| `fhir_converter/loaders.py` | Name resolution, `read_text` |
| `fhir_converter/exceptions.py` | `RenderingError` as the source for error mapping |
| `fhir_converter/templates/*/metadata.json` | Optional "type" label for a format |
| `tests/test_loaders.py` | Pattern for loader tests |

| New component (proposed) | Purpose |
|---|---|
| Catalog module (for example `fhir_converter/ui/catalog.py`) | Enumerate templates, classify them, extract includes, validate names. All filesystem access lives here. |
| Server module | HTTP handling, validation, error mapping, loopback binding. |
| Static assets (HTML, CSS, JS) | The page. |
| `tests/test_template_catalog.py` and server tests | See D.5. |
| CLI flag or console script | Optional. Changing `pyproject.toml` scripts needs approval. |
| `docs/` page and `CHANGES.md` entry | Documentation. |

Open placement question: inside the `fhir_converter` package (ships to every library user, must be Python 3.8 compatible) or a separate top-level folder (not shipped, simpler constraints). See section 4 of the closing items.

### B.5 Validating template paths and inputs

| Rule | Reason |
|---|---|
| Never build a file path from a request value. Build an allowlist from the catalog at startup (or per request) and accept only names found in it. | Removes path traversal by construction. |
| Accept `format` only from the `RenderSourceFormat` values. | Matches the CLI. |
| Reject names containing `..`, absolute paths, drive letters, backslashes or NUL. Windows is a supported platform (CI runs Windows). | Defense in depth. |
| Resolve the user template directory once at startup, require it to exist and be a directory, and refuse symlinks that leave it. | Limits reads to the configured folder. |
| Read only `.liquid` and `.json` files. Cap the size returned. | Avoids leaking unrelated files and huge responses. |
| **Assumption**: `FileExtensionLoader` (python-liquid) refuses names that resolve outside its search path. Confirm in python-liquid 1.12.1 before relying on it. The catalog allowlist should not depend on it. | Not verified in this discovery. |

### B.6 Error handling and user feedback

| Situation | Behavior |
|---|---|
| Unknown template or format | HTTP 404 or 400 with a short fixed message. |
| Template does not compile | Show the Liquid error text and line if available. **Assumption**: python-liquid error messages contain template text, not input data. Verify. |
| Render failure (Phase 2) | Do **not** echo the underlying exception. See below. |
| Directory missing or unreadable | A clear startup error, no partial server. |

Verified privacy hazard behind the render-failure rule: in the probe in C.5, a `RenderingError` caused by a `pyjson5` decode failure carried the **partially decoded output** in its message, including identifier values and display text. The CLI prints that traceback (`__main__.py:53-56`). CLAUDE.md section 5 says never put patient data into exception messages, so this is an existing gap in the project that a UI would amplify. A UI must show a generic message with an error code, and must not display `str(exception)` for render errors.

### B.7 Security considerations for file access

| Risk | Mitigation |
|---|---|
| Path traversal through template names | Allowlist built from enumeration (B.5). |
| Server reachable from other machines | Bind to `127.0.0.1` only. No authentication exists in this project, so network exposure is not acceptable by default. |
| Cross-site requests from a browser to a local server | Require a same-origin check or a per-run token on every request. Reject unexpected `Host` and `Origin` headers. |
| Reading arbitrary files through `--template-dir` | One configured root, read only, size cap, extension allowlist. |
| Rendering untrusted Liquid | Templates can call custom tags and filters (`evaluate`, `mergeDiff`, `gzip` and others in `tags.py` and `filters.py`). Treat a user template as code that runs with the process's file and CPU access. Do not render templates from an untrusted source. **Assumption** about exact sandbox limits of python-liquid. |
| Sensitive output | Converter output embeds the full source document. Never write it to a shared or persistent location by default. |
| Secrets | None are used by the project. Add none. |

---

## C. Missing C-CDA segment

### C.1 Method

I compared the section templates in `templates/ccda/Section/` (30 files, **verified**) with the include lists of every root template in `templates/ccda/*.liquid`, then searched the templates and tests for section OIDs. A "missing Patient field" (for example `maritalStatusCode` into `Patient.maritalStatus`, found in the previous task) is a different category and is excluded here: it lives inside `Resource/_Patient.liquid`, not in a C-CDA section.

### C.2 Main finding: three section templates exist but no root template includes them

**Verified** by searching `fhir_converter/templates/ccda/*.liquid` for `Section/Goal'`, `Section/HealthConcern'` and `Section/Instruction'`: no root template includes any of them. They can never run during a normal `CCD`, `ConsultationNote` or other root conversion.

Note on documentation: `docs/repo-analysis.md:111` says these sections are "present in other root templates". The search shows they are present in none. That line should be corrected after the developers agree.

### C.3 Candidates

| | Candidate 1: Goals Section | Candidate 2: Health Concerns Section | Candidate 3: Plan of Treatment Section |
|---|---|---|---|
| Template id | `2.16.840.1.113883.10.20.22.2.60` | `2.16.840.1.113883.10.20.22.2.58` | `2.16.840.1.113883.10.20.22.2.10` |
| Status | Templates exist, **not wired**, and **broken** (see C.5) | Templates exist, **not wired**, converts in a probe | **No template at all** |
| Source XML shape | `section/entry/observation` (moodCode `GOL`) with `id`, `code`, `statusCode`, `effectiveTime`, optional `value`, optional `entryRelationship/observation` for priority. | `section/entry/act` containing `entryRelationship/observation` (the concern). | Mixed entries. In `tests/data/ccda/History_and_Physical.ccda` from line 1276: `observation` (RQO, template `...4.44`), `act` (RQO, `...4.39`), `encounter` (INT, `...4.40`), `procedure` (RQO, `...4.41`). |
| Existing template files | `Section/_Goal.liquid`, `Entry/Goal/_entry.liquid`, `Resource/_Goal.liquid`, `Reference/Goal/_Subject.liquid` | `Section/_HealthConcern.liquid`, `Entry/HealthConcern/_entry.liquid`, `Entry/HealthConcern/_entry_act_entryRelationship.liquid`, `Resource/_Condition.liquid`, `Reference/Condition/_Subject.liquid` | None. `Resource/_ServiceRequest.liquid` exists (used by `Entry/ReasonforReferral`). A `Reference/CarePlan` folder exists but no `Resource/_CarePlan.liquid`. |
| Expected FHIR | `Goal` (subject to the Patient) | `Condition` | **Assumption**: a mix of `ServiceRequest`, `Procedure`, `Encounter` and possibly `CarePlan`, depending on entry type and mood. Needs a mapping decision. |
| Test coverage | None. No test in `tests/*.py` mentions goal, health concern or these OIDs, and no fixture in `tests/data` contains `...2.60`, `...2.58` or `...2.45`. | Same as Goals. | The only fixture that contains the section is exercised through `CCD`, which has no template for it, so the section is ignored. No assertion about it. |
| Missing work | Fix a template defect, wire into root template(s), fixture, tests, map status values. | Wire into root template(s), fixture, tests, decide category and status mapping. | Design a mapping per entry type, new `Entry/` and possibly `Resource/` templates, fixtures, tests. |
| Complexity | **Low to medium** | **Low to medium** | **High** |

Related, not a candidate on its own: the Instructions Section (`2.16.840.1.113883.10.20.22.2.45`) has the same unwired state and the same `{$` defect in `Entry/Instruction/_entry.liquid:3`. It would be a natural follow-up using the same fix.

### C.4 Candidate details (verified source)

**Goals.** `Section/_Goal.liquid` finds the section by OID and renders `Entry/Goal/entry`. `Resource/_Goal.liquid` emits `id`, `identifier`, `description` (from `code`), `lifecycleStatus` (the raw `statusCode.code`), `priority`, and one `target` with `measure`, `detailQuantity` and `dueDate`. `Reference/Goal/_Subject.liquid` emits the Patient reference.
Mapping questions seen in the source: `lifecycleStatus` is not mapped through `ValueSet.json` (no `ValueSet/Goal...` key exists), so a C-CDA status that is not a FHIR `GoalLifecycleStatus` code would pass through unchanged. **Assumption**: C-CDA allows `aborted`, which FHIR does not list. Verify against the C-CDA and FHIR value sets.

**Health Concerns.** `Entry/HealthConcern/_entry.liquid` only batches the act's `entryRelationship` items to `_entry_act_entryRelationship.liquid`, which creates a `Condition` from the nested `observation` and references the Patient. The act itself (status, author, time) is not mapped. **Assumption**: health concerns can legitimately differ from problem-list conditions and may need a category, and Problems and Health Concerns could produce duplicate-looking Conditions. A mapping decision is needed.

**Plan of Treatment.** No file references the OID. A real mapping needs per-entry-type rules. The template ids in the fixture (`4.39`, `4.40`, `4.41`, `4.44`) are taken from the file; their C-CDA names are **assumption**, not confirmed in this repository.

### C.5 Evidence from a read-only probe (synthetic data)

To check whether the unwired templates work, I rendered a synthetic document with a Goals section and a Health Concerns section through two temporary probe templates that included `Section/Goal` and `Section/HealthConcern`. Files were kept in the session scratchpad, outside the repository, and nothing in the repository was modified. The ids and names in the input are invented.

| Probe | Result |
|---|---|
| `Section/HealthConcern` | `RENDER SUCCESS`. Output was a `Condition` with `identifier`, `code` (SNOMED), `onsetDateTime` and `subject`. **Observed gap**: no `clinicalStatus` or `category` in the output. |
| `Section/Goal` | `RENDER FAILURE`: `pyjson5.Json5IllegalCharacter: Expected b'colon' near 1184, found U+0069`, wrapped as `RenderingError: Failed to render FHIR`. |

Cause of the Goal failure (**verified** from source and consistent with the probe): `Entry/Goal/_entry.liquid:3` is

```
{$ include 'Reference/Goal/Subject' ID: observationId, REF: fullPatientId -$}
```

It uses `{$ ... $}` instead of `{% ... %}`. Liquid does not treat it as a tag, so the text lands in the JSON output and the JSON5 parser rejects it. The same typo is at `Entry/Instruction/_entry.liquid:3`. A literal reading of the template alone is consistent with the failure. The probe is the confirmation. I did not try the corrected tag.

To reproduce: create a temporary template directory, put in it a root template that does `{% evaluate patientId using 'Utils/GenerateId' obj: msg.ClinicalDocument.recordTarget.patientRole -%}`, `{% assign fullPatientId = patientId | prepend: 'Patient/' -%}` and `{% include 'Section/Goal' -%}` inside the Bundle `entry` array, then run the CLI with `--template-dir` pointing at it.

### C.6 Recommendation: Goals Section

| Evidence | Why it favors Goals |
|---|---|
| Fully built chain (section, entry, resource, reference) and nothing reaches it | The smallest change that makes a whole C-CDA section produce output. |
| Reproducible failure with a one-character-class cause | A concrete, verifiable defect, not a vague gap. |
| Zero test coverage today | A new synthetic fixture and tests add real value and are easy to scope. |
| Distinct FHIR resource (`Goal`) with a bundled Patient reference | Lower duplication risk than Health Concerns, which can overlap with Problem Conditions. |
| Stays inside the existing architecture | Template changes only, per CLAUDE.md section 1: "Prefer a template change over Python code". |

Why not the others first: Health Concerns works already once wired, so it is a smaller win and has the duplication question. Plan of Treatment is the largest real gap but needs mapping design and approval; I recommend it as a later, separately scoped item.

Caveat: calling Goals "missing" is a simplification. It is **incomplete** (built but unreachable and broken), not absent.

---

## D. Technical risks

### D.1 Backward compatibility

| Risk | Detail |
|---|---|
| Adding `Section/Goal` to a root template adds `Goal` entries to bundles | Output shape changes for every consumer of that root template. CLAUDE.md section 9 requires asking before changing output shape. Documents with no Goals section are unaffected (**assumption**: `get_first_ccda_sections_by_template_id` returns an empty mapping when the section is absent, then `batch_render` renders nothing; confirm with a before and after diff). |
| Ids | Resource ids come from `generate_uuid` over the entry JSON. Do not change what is hashed (CLAUDE.md section 3). The Goal template follows the existing pattern. |
| Public API | `render_fhir_string`, `render_fhir`, `render_to_fhir`, `make_environment`, `render_to_dir`, `render_files_to_dir` must stay stable. A UI must call them, not change them. |
| Python 3.8 | New library modules must avoid 3.9+ only syntax. |

### D.2 FHIR mapping risks

| Risk | Detail |
|---|---|
| Goal `lifecycleStatus` is copied raw | Values outside the FHIR value set would pass through (C.4). |
| Goal `priority` can be emitted with an empty coding | Seen in the template. `parse_fhir` is documented to drop empty values, which was **not** confirmed for this exact case. |
| Health Concern Condition lacks `clinicalStatus` and `category` | Observed in the probe. |
| Health Concern and Problem overlap | Possible near-duplicate Conditions. |
| Plan of Treatment entry types | Different moods (RQO, INT) imply different FHIR resources and statuses. |
| Output is not validated | No FHIR validator runs in the project. Do not describe output as conformant. |

### D.3 Template-loading risks

| Risk | Detail |
|---|---|
| Loader cache | `make_environment` defaults `auto_reload=False`, `cache_size=300`. Edited templates are not seen until restart. Relevant to any UI that shows source. |
| Name prefixing | A UI that lists files must map `Section/_Goal.liquid` back to `Section/Goal` the same way `_resolve_template_name` does. |
| User directory shadows packaged templates | With `--template-dir`, the user folder wins. A user file named like a packaged one silently changes conversion. The UI should flag overrides. |
| Includes are late bound | A template can compile and still fail at render when an include is missing (the `{$` case shows the related failure mode for malformed tags). |
| `tags.py` | Uses internal python-liquid modules. Treat as fragile. `python-liquid` stays at 1.12.1. |
| `metadata.json` | Only read by a test. A UI should not depend on it without checking the STU3 copy. |

### D.4 Security and data privacy

| Risk | Detail |
|---|---|
| Exceptions carry data | Verified in C.5 and B.6. Existing behavior, also printed by the CLI. |
| Output embeds the source document | `_originalData` and `Resource/_DocumentReference.liquid`. Anything written under `data/out` is sensitive. |
| Test data | Use only synthetic data (`data/synthetic`). Do not use `data/sample`, whose provenance is unknown. |
| Probe hygiene | The probe used only invented data and temporary files outside the repository. |
| No authentication | A UI must stay loopback-only unless an access model is agreed. |
| No compliance claims | Do not add HIPAA, SOC 2, HITRUST or ISO language to UI text or docs. |

### D.5 Testing requirements

| Area | Requirement |
|---|---|
| Baseline | Run the full suite before and after any change and report actual results (CLAUDE.md section 6). Baseline not re-run in this discovery. |
| Goals | New synthetic fixture, for example `data/synthetic/ccda/TD-CCDA-13_goals.ccda`. Tests in `tests/test_renderers.py`: Goal present with subject reference, no Goal when the section is absent, existing `CCD` output otherwise unchanged. Diff JSON before and after on `TD-CCDA-01`. |
| Status mapping | A test for each C-CDA goal status that is expected to appear. |
| Catalog | Tests for listing, classification, name mapping (`_` prefix), traversal attempts (`..`, absolute path, backslash, drive letter), symlink escape, oversized file, missing directory. |
| Server | Tests for loopback binding, header checks, error mapping, and that no exception text reaches a response. |
| CLI | `__main__.py` has no unit tests. Any CLI change is checked manually with `docs/test-cases.md` (TC-CLI-*). |
| Docs | Update `docs/test-cases.md` statuses only after running them. Update `CHANGES.md` for user-visible changes. |

---

## Closing items

### 1. Recommended UI architecture

A local, read-only template browser. Python standard library HTTP server bound to `127.0.0.1`, static HTML and JS, and a new catalog module that enumerates templates and validates names against an allowlist. It reuses `make_environment`, the package loaders and `read_text`. Phase 1 covers listing, selection, details (source, includes, used-by), and a name and compile check. No editing, upload or deletion. Render preview with user input is deferred to a later phase with the privacy handling in B.6.

### 2. Recommended missing segment

**Goals Section** (`2.16.840.1.113883.10.20.22.2.60`, FHIR `Goal`). The chain of templates exists, nothing includes it, it has no tests, and it fails today because of the `{$ ... $}` typo at `Entry/Goal/_entry.liquid:3`. Health Concerns (`...2.58`) and Instructions (`...2.45`) are follow-ups with the same unwired state. Plan of Treatment (`...2.10`) is the larger real gap and needs its own design.

### 3. Proposed implementation scope

Two independent pieces, each its own change set.

| Piece | In scope | Out of scope |
|---|---|---|
| Goals | Fix the tag typo. Include `Section/Goal` in the agreed root template(s). Add a synthetic fixture. Add tests. Map or restrict `lifecycleStatus` if the developers choose. `CHANGES.md` entry. Correct `docs/repo-analysis.md:111`. | Health Concerns, Instructions, Plan of Treatment. Any change to ids, public API or dependencies. |
| UI Phase 1 | Catalog module, read-only endpoints, static page, loopback binding, tests, docs. | Template editing, upload, deletion. Render preview with real input. Authentication. New dependencies unless approved. |

Suggested order: Goals first (small, verifiable, immediate value), then the UI catalog.

### 4. Open questions requiring developer approval

| # | Question |
|---|---|
| 1 | Which root templates should include `Section/Goal`: `CCD` only, or also `ConsultationNote`, `ReferralNote`, `TransferSummary` and others? This changes output for those templates. |
| 2 | Is it acceptable that bundles from documents with a Goals section gain `Goal` entries (output shape change)? |
| 3 | How should goal status be handled: pass through, map through a new `ValueSet/...` key in `ValueSet.json`, or drop unknown values? |
| 4 | Should the same `{$` typo in `Entry/Instruction/_entry.liquid:3` be fixed in the same change, even though Instructions stay unwired? |
| 5 | Should Health Concerns and Plan of Treatment be queued as separate work items? |
| 6 | UI hosting: standard library server (no new dependencies) or a web framework (new dependencies, 3.8 to 3.12 check)? |
| 7 | UI placement: inside the `fhir_converter` package (ships to all users, must stay Python 3.8 compatible) or a separate top-level folder? |
| 8 | Is a render preview with user-supplied input wanted at all, given that the output embeds the source document and render errors can carry data? If yes, what is the storage and error policy? |
| 9 | May the UI show the content of `--template-dir` user templates, and should overrides of packaged templates be flagged? |
| 10 | Should the pre-existing exception-content issue (decode errors carrying partial output, printed by the CLI) be tracked as its own item? It is outside this discovery but it blocks a safe UI error path. |
| 11 | May I run the full suite, flake8 and mypy for a fresh baseline before implementation? CLAUDE.md notes flake8, mypy and Poetry were not installed locally when last checked. |
