# Discovery: UI for Managing Templates

Status: discovery only. No application code, template, test, dependency or configuration file was changed. This is the only file created. No feature specification has been written.

Audience: the developers who own `python-fhir-converter` (v0.3.0), who will review these findings before a spec is started.

Date: 2026-11-05.

## How to read this document

| Tag | Meaning |
|---|---|
| **Verified** | Read from a file in this repository, or counted with a command, during this discovery. |
| **Carried** | Taken from `docs/day2/00-discovery.md` (section A and B), which already covers part of this topic. Not re-read line by line today. |
| **Assumption** | Not confirmed in this repository. Needs a check. |
| **Not run** | Not executed. The pytest suite, flake8 and mypy were not run. |

This is an engineering analysis. It makes no claim that any output is FHIR-valid and no compliance claim. It is not a substitute for review by the developers.

Sources read: `CLAUDE.md`, `.specify/memory/constitution.md` (v1.0.0, a draft awaiting ratification), `docs/day2/00-discovery.md`, `pyproject.toml`, `fhir_converter/loaders.py`, `fhir_converter/__main__.py` (lines 80 to 100), `tests/test_loaders.py`, `README.md` (Templates section), `docs/test-cases.md`, and directory listings.

Overlap note: `00-discovery.md` section B already proposed a read-only template browser. This report widens the inspection to the eight questions asked and keeps the UI proposal as one option, not a decision.

---

## 1. Current architecture

### 1.1 Shape of the project

| Item | Finding | Evidence |
|---|---|---|
| Type | Python library plus a command line tool. No server, database, API or UI | **Verified** `pyproject.toml` (classifier "Environment :: Console", one script entry), no web files or web packages (see 5) |
| Pipeline | Parse input to a dict (`msg`), render a Liquid template, `parse_fhir` prunes and merges, write JSON | **Carried** `renderers.py`, `CLAUDE.md` section 1 |
| Where mapping lives | In templates, not Python. `CLAUDE.md` says to prefer template changes | **Verified** `CLAUDE.md` section 1 and 3, constitution principle I |
| Entry points | `fhir_converter_cli` script and `python -m fhir_converter`, both ending in `fhir_converter.__main__:entrypoint` | **Verified** `pyproject.toml` `[tool.poetry.scripts]` |
| Public API | `render_fhir_string`, `render_fhir`, `render_to_fhir`, `make_environment`, `render_to_dir`, `render_files_to_dir` must stay stable | **Verified** `CLAUDE.md` section 3, constitution II |

### 1.2 How templates are organized and stored

| Item | Finding | Evidence |
|---|---|---|
| Location | Inside the package: `fhir_converter/templates/ccda/` and `fhir_converter/templates/stu3/` | **Verified** directory listing |
| Count | 1,083 `.liquid` files: 819 under `ccda`, 264 under `stu3` | **Verified** `find ... -name '*.liquid' | wc -l` |
| C-CDA layout | Root templates at the top: `CCD`, `ConsultationNote`, `DischargeSummary`, `HistoryandPhysical`, `OperativeNote`, `ProcedureNote`, `ProgressNote`, `ReferralNote`, `TransferSummary`, plus `Header`. Sub-folders: `Section`, `Entry`, `Resource`, `Reference`, `DataType`, `Extension`, `Utils`, `ValueSet`. Also `cda_narrative.xsl` | **Verified** directory listing |
| STU3 layout | One template per resource type at the top, plus sub-folders for nested pieces | **Carried** (not re-listed today) |
| Naming rule | Files in a sub-folder carry a `_` prefix on disk (`Section/_Problem.liquid`), but are referred to without it (`Section/Problem`). Root templates and `.json` names have no prefix | **Verified** `loaders.py` `_resolve_template_name` |
| Metadata | `templates/ccda/metadata.json` is `{"type": "ccda"}` and `templates/stu3/metadata.json` is `{"type": "fhir"}` | **Verified** file contents. Only reader found is a test (**Carried**) |
| Code translations | `ValueSet/ValueSet.json`, read through the loader and exposed as the `code_mapping` global | **Carried** |
| Not a database | Templates are plain files shipped with the package. No versioning, ownership or status field exists | **Verified** by absence |

### 1.3 How templates are discovered, loaded and rendered

| Step | What happens | Evidence |
|---|---|---|
| Default loading | `PackageLoader` for `ccda` and for `stu3`, wrapped by `TemplateSystemLoader` (or the caching variant when the cache size is above 0) | **Verified** `loaders.py` (`make_template_system_loader`), **Carried** `renderers.py:59-67` |
| Lookup | The caller passes a template name string. `TemplateSystemLoader` rewrites the last path segment, then python-liquid asks each loader in order | **Verified** `loaders.py` |
| Discovery | There is **no listing or search function**. The loader can only fetch a name it is given. A missing name raises `TemplateNotFound`, surfaced as `RenderingError` | **Verified** `loaders.py` (no list method), `docs/test-cases.md` TC-CLI-11 |
| Reading source | `loaders.read_text(env, filename)` returns a template's source through the loader | **Verified** `loaders.py` |
| Caching | Defaults are `auto_reload` off and a 300-template cache, so an edited template is not seen until restart | **Carried** `renderers.py:315`. Parameter defaults visible in `loaders.py` (`cache_size: int = 300`) |
| Includes | Templates call each other with `{% include 'Section/Problem' %}`. Includes resolve at render time, so a file can load and still fail later | **Carried** |

### 1.4 How a template is selected during C-CDA to FHIR conversion

- The user chooses a **root template by name** on the command line: `--template-name CCD` (required), together with `--source-format CCDA`. The same name is a string argument to `render_fhir_string` and friends. **Verified** `docs/test-cases.md` TC-CLI-05 and TC-CLI-11, **Carried** `__main__.py:131-198`.
- The root template decides which sections run by including them (`CCD.liquid` includes `Section/Problem`, `Section/Medication` and so on). **Verified** earlier in this project by reading `CCD.liquid`.
- Nothing in the application chooses a template by document type. The user has to know which root name to pass. **Assumption** drawn from the absence of any selection logic in `renderers.py`. Confirm.
- Not every section template is reachable from a root template. The Goal, Health Concern and Instruction sections exist but no root includes them. **Carried** `00-discovery.md` C.2.

### 1.5 Custom and user-defined templates

- `--template-dir PATH` builds an environment with the user folder first and the packaged loader second, so user files override packaged ones of the same name. **Verified** `__main__.py` `get_user_defined_environment` (lines 86 to 94).
- That function has no unit tests and lives in the CLI module. **Carried** `CLAUDE.md` section 6.
- Examples of custom templates: `data/synthetic/templates/Mini.liquid` (**Verified** referenced by TC-CLI-05), and `data/templates/ccda/*.liquid` (**Carried**).
- Library users can pass their own `Environment` to a renderer, and may register extra filters and tags. **Carried**.

---

## 2. Existing template management capabilities

| Capability | Available today? | How | Evidence |
|---|---|---|---|
| View a template's source | Only by opening the file on disk, or from code via `read_text` | No command or screen | **Verified** `loaders.py`, no CLI argument for it |
| List templates | No | No listing in code or CLI | **Verified** CLI arguments in `docs/test-cases.md`, `loaders.py` |
| Search templates | No | None | **Verified** by absence |
| Inspect structure (what includes what, which root uses a section) | No | A reader must search text by hand | **Verified** by absence |
| Check that a name resolves | Indirectly | Run a conversion and see `TemplateNotFound` (TC-CLI-11) | **Verified** |
| Check that a template compiles | Not as a feature | `env.get_template(name)` would load it | **Assumption**: python-liquid parses on load but a missing include only fails at render |
| Create or edit a template | By hand in a text editor, then `--template-dir` | No tool | **Verified** |
| Delete, version, roll back | No | Only file system and git | **Verified** by absence |
| Override a packaged template | Yes, by name, through `--template-dir` | One directory, user wins | **Verified** `__main__.py:86-94` |
| Warn about an override | No | A user file silently shadows a packaged one | **Carried** `00-discovery.md` D.3 |
| Validate output of a template | No | The project runs no FHIR validator | **Verified** `CLAUDE.md` section 8 |
| UI of any kind | No | See section 5 | **Verified** |

---

## 3. Identified gaps

| # | Gap | Effect |
|---|---|---|
| G1 | No way to list or search the 1,083 templates | A user must know exact names and browse folders by hand |
| G2 | No view of relationships (root, section, entry, resource, datatype) | Impact of a change cannot be seen. Unwired sections such as Goal went unnoticed |
| G3 | No way to see which templates are used by which root template | Cannot tell what a root template actually converts |
| G4 | No check of a name or a custom template before running a conversion | Errors appear only at render time, sometimes with data in the message |
| G5 | Overrides from `--template-dir` are invisible | A same-named user file silently changes output |
| G6 | No safe edit workflow | Editing is by hand with no preview, and the cache hides changes until restart |
| G7 | No tests for the catalog-style behavior or for `get_user_defined_environment` | Any new logic has no existing test pattern to extend, except `tests/test_loaders.py` |
| G8 | Existing error text can carry source data (a decode failure printed partial output in the earlier probe) | A UI that shows errors would amplify the leak |
| G9 | Docs describe templates only in the README Templates section and `docs/architecture.md`. No page lists or explains the template catalog | New users rely on reading the tree |

---

## 4. Relevant tests and documentation

| Item | What it covers | Evidence |
|---|---|---|
| `tests/test_loaders.py` | `TemplateSystemLoader` name resolution (with and without `_`, with args and context), caching variants, `TemplateNotFound`, `read_text` (file not found, read text) | **Verified** test names listed (about 17 tests) |
| `tests/test_renderers.py` | `make_environment` defaults, auto reload, cache size, additional loaders, plus conversion tests on `tests/data/ccda` | **Carried** (names seen earlier in this project) |
| `docs/test-cases.md` | TC-CLI-05 (custom template through `--template-dir`) and TC-CLI-11 (unknown template name) are the only template management cases | **Verified** |
| `README.md` Templates section | States templates load from the package by default and that `TemplateSystemLoader` handles the naming convention | **Verified** |
| `docs/architecture.md`, `docs/repo-analysis.md`, `docs/api-guide.md` | Describe loading and extension points. No management feature to describe | **Carried** |
| `mkdocs.yml` and `docs/` site | The project publishes docs with MkDocs. Docstring pages are generated (`docs/docstrings/`, do not edit) | **Verified** `mkdocs.yml` present, **Carried** `CLAUDE.md` section 8.1 |
| Gap | No test or doc for listing, searching, inspecting or editing templates, because the capability does not exist | **Verified** by absence |

---

## 5. Existing frameworks, dependencies and entry points that could support a UI

| Item | Finding | Evidence |
|---|---|---|
| Web or UI frameworks present | **None.** No `package.json`, `.html`, `.js` or `.tsx` outside `.venv`, `site` and `htmlcov`. No flask, fastapi, django, starlette, uvicorn, streamlit, gradio, jinja2, tornado, aiohttp or bottle in the virtual environment | **Verified** `find` and a `site-packages` listing run today |
| Runtime dependencies | `python-liquid 1.12.1`, `xmltodict`, `pyjson5`, `frozendict`, `importlib_resources`, `psutil`, `isodate`, `lxml`, `typing-extensions` (all pinned) | **Verified** `pyproject.toml` |
| Dev dependencies | pytest, pytest-mock, pytest-cov, type stubs, flake8, mypy, smokeshow, lazydocs, mkdocs plugins, watchdog | **Verified** `pyproject.toml` |
| Rule on dependencies | Do not add or change pins without asking | **Verified** `CLAUDE.md` section 7, constitution V |
| Python version | `^3.8.1`, CI 3.8 to 3.12. Library modules must stay 3.8 compatible. `__main__.py` already uses 3.11 syntax | **Verified** `CLAUDE.md` section 4 |
| Entry points | `fhir_converter_cli` and `python -m fhir_converter` (CLI only). A new entry would mean a `pyproject.toml` change, which needs approval | **Verified** |
| Reusable pieces | `make_environment`, the default loaders, `read_text`, `RenderingError`, and `TemplateSystemLoader._resolve_template_name` (private) | **Carried** `00-discovery.md` A.4, `_resolve_template_name` **Verified** |
| Standard library options | A UI could be built from the standard library (`http.server`, `html`, `json`) with no new dependency. `tkinter` is also standard but may be missing on some Python builds | **Assumption**: availability on the CI matrix is not checked. `docs` MkDocs site is not an application UI |
| Frontend framework | Not needed to meet any requirement found. None is assumed | Per your instruction |

---

## 6. Potential UI approaches based on the existing application

These are options, not a decision. None adds a dependency.

| Option | Description | Fit | Concerns |
|---|---|---|---|
| A. Extend the CLI | Add listing, search and inspect sub-commands to the existing CLI | Smallest. Matches the current product (a console tool) | Changes the CLI surface and `pyproject.toml` scripts (approval). The CLI has no unit tests. Not a visual UI |
| B. Local read-only browser using the standard library | A small local page served from `http.server`, static HTML and plain JavaScript, backed by a catalog module over the existing loaders | Matches "UI layer" without a framework. Reuses `make_environment` and `read_text` | New module and endpoints. Needs loopback binding and request checks. Hand-written routing |
| C. Generated static report | A script that writes a static HTML catalog of templates and relationships | No server, no network exposure, safest for privacy | Not interactive and goes stale. Not "management" |
| D. Full management UI (create, edit, delete) | Edit templates in a browser | Meets the broadest reading of "managing" | Not proven safe here. A write path changes output for every consumer, and the template cache needs invalidation. See risks. Do not choose this without an explicit decision |
| E. Web framework plus ASGI server | Better routing and validation | Familiar | Adds runtime dependencies to a library that has none, needs approval and a 3.8 to 3.12 check. Not recommended now |

Suggested direction for review: a phased path of **C or B first (read-only)**, then a separate decision on editing. This extends `00-discovery.md` section B, which recommended B. A catalog module (list, classify, read, resolve names, include graph) is common to A, B and C, so it can be built and tested once whichever surface is chosen. **Assumption**: that reuse holds, since no code exists yet.

---

## 7. Functional requirements to consider

Not yet requirements. Items to decide on in the spec.

| # | Candidate requirement | Phase hint |
|---|---|---|
| F1 | List all templates for a chosen format (C-CDA or STU3), with kind (root, section, entry, resource, datatype, valueset, other) and origin (package or user directory) | 1 |
| F2 | Search templates by name and by text within source | 1 |
| F3 | View a template's source, size and resolved file name (the logical name without `_`) | 1 |
| F4 | Show what a template includes, and which templates include it (used-by), including which root templates reach a section | 1 |
| F5 | Flag sections that no root template includes (the Goal case) | 1 |
| F6 | Check that a name resolves and that the template compiles, with a safe error message | 1 |
| F7 | List templates from a configured user directory and mark those that override a packaged template | 1 |
| F8 | Show the C-CDA section template id (OID) a section template handles | 1 (**Assumption**: a text match on `get_..._by_template_id` calls is enough) |
| F9 | Render preview with a user-supplied input file | 2, needs approval (privacy) |
| F10 | Create, edit, duplicate, delete user templates | Not recommended without an explicit decision |
| F11 | Export or download a template | Optional, read-only |
| F12 | Show code-mapping tables from `ValueSet.json` | Optional, read-only |

---

## 8. Non-functional requirements to consider

| Area | Requirement to consider |
|---|---|
| Compatibility | No change to the public API, CLI options, exit codes, ids or conversion output (constitution II). Library code stays Python 3.8 compatible |
| Dependencies | None added. No frontend framework assumed |
| Architecture | Follow template-first design. New Python code, if any, lives in a separate catalog module with typed code and `Args:/Returns:/Raises:` docstrings, flake8 limits (120 columns, complexity 10) and no new logging channel |
| Security | Loopback binding only. No authentication exists, so no network exposure. Same-origin or token check against cross-site requests. Names come only from an allowlist built from enumeration, never from a request path. Read-only by default |
| Privacy | Use synthetic data only (`data/synthetic`). Never show `str(exception)` for render errors. Do not write converter output to a persistent or shared location. Output embeds the whole source document |
| Compliance | No HIPAA, SOC 2, HITRUST or ISO wording in UI text or docs |
| Reliability | A clear startup error if a configured directory is missing. No partial server |
| Performance | Listing 1,083 files should be quick. **Assumption**: no target is stated, so one must be set. Cache the catalog and note that the template cache has `auto_reload` off |
| Portability | Works on Windows, Linux and macOS (CI runs all three). Handle Windows paths, backslashes and drive letters |
| Accessibility | Keyboard navigable and readable without a mouse if there is a visual UI. **Assumption**: no standard is chosen yet |
| Testability | Catalog logic unit tested with the existing `tests/test_loaders.py` style. CLI changes checked manually against `docs/test-cases.md` (TC-CLI-*). Full suite before and after, with real results reported |
| Documentation | README or docs page, `CHANGES.md` entry, and `docs/test-cases.md` updated only after cases are run |
| Maintainability | Small diffs. No mass edits of the 1,083 templates. No edits to generated or ignored paths |

---

## 9. Security and compatibility considerations for managing templates

| Topic | Consideration | Status |
|---|---|---|
| Templates are code | Liquid templates can call custom filters and tags (`evaluate`, `mergeDiff`, `gzip` and others). A user template runs with the process's file and CPU access. Never render or install templates from an untrusted source | **Carried** `00-discovery.md` B.7. Exact python-liquid sandbox limits are **Assumption** |
| Path traversal | A UI must not build file paths from request values. Reject `..`, absolute paths, drive letters, backslashes and NUL. Refuse symlinks that leave the configured root | **Carried** B.5. Whether `FileExtensionLoader` refuses names outside its search path is **Assumption** and should not be relied on |
| Overrides change conversion | A user file with a packaged name silently changes output and, indirectly, which fields exist | **Verified** override order in `__main__.py:86-94` |
| Cache | An edit is invisible until restart when `auto_reload` is off. A write feature would need invalidation | **Carried** |
| Ids | Resource ids hash the entry JSON. A template edit that changes what is hashed changes every consumer's ids | **Verified** `CLAUDE.md` section 3 |
| Backward compatibility | Read-only features cannot change output. Editing features can | Follows from the above |
| Data exposure | Source text of packaged templates is not patient data. Rendered output is. Error text from decode failures has carried partial output | **Carried** B.6 |
| Name mapping | A UI must map `Section/_Goal.liquid` back to `Section/Goal` the same way the loader does | **Verified** `loaders.py` |
| Platform | CI runs Windows, so path handling must be explicit | **Carried** |
| Python | `__main__.py` uses 3.11 syntax while the library supports 3.8. Where new code lives decides which rule applies | **Verified** `CLAUDE.md` section 4 |

---

## 10. Risks

| # | Risk | Likelihood and impact | Mitigation to consider |
|---|---|---|---|
| K1 | Scope creep from "view" to "edit" | Likely, high. Editing changes output for all consumers | Phase the work. Decide on editing separately |
| K2 | A write feature without cache invalidation or safe storage | Medium, high | Keep read-only first. If editing is approved, write only to a configured user directory |
| K3 | Path traversal or arbitrary file read through a local server | Medium, high | Allowlist from enumeration, loopback only, request checks |
| K4 | Cross-site requests reach the local server | Medium, medium | Token or same-origin check, host and origin validation |
| K5 | Error messages leak source data in a UI | Likely, high | Generic message with an error code, never `str(exception)` for render errors |
| K6 | New dependency added by habit (framework, server) | Medium, medium | Standard library first. Approval for anything else |
| K7 | Placing UI code inside the shipped package forces 3.8 compatibility and ships it to every library user | Medium, medium | Decide placement (see open questions) |
| K8 | Overrides hide intended behavior | Medium, medium | Flag overrides in the listing |
| K9 | Include analysis by text search misses dynamic includes | Medium, low | Treat as best-effort and label it so. **Assumption**: all includes seen use single quotes |
| K10 | A "management" UI creates an expectation of validation, but the project runs no FHIR validator | Medium, medium | State plainly that output is not validated |
| K11 | Baseline not run, so regressions are not measurable yet | Certain today | Run the suite and record counts before any change |
| K12 | The constitution is an unratified draft | Low, medium | Treat it as guidance until ratified |
| K13 | `data/sample` and `data/sample/stu3` files show as modified in the working tree, and their provenance is unknown | Low, medium | Not touched by this work. Do not use or copy them into tests, docs or specs (`CLAUDE.md` section 8) |

---

## 11. Open questions for the developers

| # | Question |
|---|---|
| Q1 | What does "managing" mean for the first release: view and search only, or also create, edit and delete? |
| Q2 | Who are the users (developers of this library, integrators, analysts)? A visual UI may not be needed if the answer is developers only |
| Q3 | Which surface: extended CLI, local browser page, static report, or something else? (Section 6 options A to E) |
| Q4 | Where should the UI code live: inside `fhir_converter` (ships to all users, Python 3.8 rules) or in a separate top-level folder? |
| Q5 | Is a new console script or CLI argument acceptable, given it changes `pyproject.toml` or the CLI surface? |
| Q6 | Which templates are in scope: packaged, user directory, or both? Should the packaged ones be strictly read-only? |
| Q7 | Is a render preview with user-supplied input wanted at all, given that output embeds the source document and errors can carry data? If yes, what storage and error policy applies? |
| Q8 | Should overrides from `--template-dir` be shown and flagged? |
| Q9 | Should unwired sections (Goal, Health Concerns, Instructions) be flagged in the UI? This links to the `001-wire-goals-section` work, which is still awaiting approval |
| Q10 | Is the exception-content issue (decode errors carrying partial output) to be tracked as its own item before any UI shows errors? |
| Q11 | Are there performance, accessibility or browser-support expectations? |
| Q12 | May I run the full suite, flake8 and mypy for a fresh baseline? flake8, mypy and Poetry were not installed in the local environment when last checked |
| Q13 | Is the constitution ready to ratify, so the spec can cite it as final? |

---

## 12. Suggested next step

Review sections 3, 6 and 11. Once Q1 to Q5 are answered, a feature specification can be written for the agreed first phase. No spec has been created.
