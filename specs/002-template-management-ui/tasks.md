---

description: "Task list for the Template Management UI feature"
---

# Tasks: Template Management UI

**Input**: Design documents from `specs/002-template-management-ui/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/ui-contract.md](contracts/ui-contract.md), [quickstart.md](quickstart.md)

**Status**: TASK LIST ONLY. Nothing has been implemented. No application code, template, test or configuration file was changed while generating this list, and no command was run. The spec and plan are drafts. Nothing in them is approved, and the constitution is an unratified v1.0.0 draft.

**Tests**: The spec defines 43 test scenarios (T01 to T43) and the plan defines unit, server, security and regression testing, so test tasks are included alongside the implementation. Scenario IDs below such as "scenario T09" mean the spec's test scenarios, not the task IDs in this file.

**Organization**: Phases follow the order you asked for. User story labels come from the spec: **US1** find and read templates (P1), **US2** understand relationships (P1), **US3** overrides (P2), **US4** potentially unreferenced sections (P3), **US5** safe local use (P1). Phases 1, 2, 9 and 10 are shared or cross-cutting and carry no story label.

## Format: `- [ ] T### [P] [US#] Description with file path`

- **[P]**: can run in parallel (different files, no dependency on an incomplete task).
- **[US#]**: user story from the spec.
- **⛔ GATE-n**: approval-gated. **Do not perform the task until that gate is cleared by a developer.** See the gate table. Spec references (FR, SEC, NFR, COMP, SC and scenario IDs) follow each description.

## Approval gates

Every task that creates or edits a file is also blocked by **GATE-0**, so nothing starts before the developers approve implementation. The gates below are the additional ones. Clearing a gate means a developer records the decision in the spec's Decision Register or in this file. This list does not clear any of them.

| Gate | What needs approval | Spec decision |
|---|---|---|
| GATE-0 | Starting implementation at all (spec and plan reviewed, open **[APPROVAL]** items for each task decided) | FR-012 style gate, all D-items |
| GATE-1 | Running the baseline `pytest`, CLI captures and conversions. Installing flake8, mypy or Poetry | D-27 |
| GATE-2 | Running a package build and a clean virtual environment install | D-22 |
| GATE-3 | Exposing an explicit port option | D-13 |
| GATE-4 | The name and form of the user directory option | D-12 |
| GATE-5 | Suppressing per-request console logging and printing the single startup message | D-20 |
| GATE-6 | Token transport and the token appearing in the startup message | D-11, SEC-019 |
| GATE-7 | Concrete limits (file size, results, query length, time bounds, timeout, tag-text length) | D-16 |
| GATE-8 | Category taxonomy and handling of user files without the `_` prefix | D-08, D-09 |
| GATE-9 | Scanner details: double-quoted literals, comment and raw blocks, whitespace variants | D-04 |
| GATE-10 | Ruling on constitution principle I wording, and ratification | N-01, D-26 |
| GATE-11 | Running temporary investigation scripts or prototypes (I-2, I-4) | D-19 |
| GATE-12 | Installing or using other Python versions locally | I-3 |
| GATE-13 | Performance, startup, accessibility and browser targets | D-24 |
| GATE-14 | Single HTML page with plain JavaScript | D-25 |
| GATE-15 | Roots definition, effective graph, read timing, re-implemented name mapping | D-06, D-15, D-17, D-14 |
| GATE-16 | Extra response headers (CSP, nosniff, no-store, no-referrer) | N-04 |
| GATE-17 | Section id (OID) detection | FR-031 AC2, **[APPROVAL]** |
| GATE-18 | Any `pyproject.toml` or ignore-file change | FR-094 AC3 |

## Path conventions

New code: `fhir_converter/ui/`. New tests: `tests/test_ui_*.py` and one helper `tests/ui_fixtures.py`. Existing modules, templates and existing tests are never edited. Test commands use PowerShell: `.\.venv\Scripts\python.exe -m pytest tests\<file> -q`. The local `.venv` is Python 3.14.10, outside 3.8 to 3.12, so CI is the compatibility authority.

---

## Phase 1: Repository baseline and verification

**Purpose**: Measure the starting point and settle the investigations before code is written. Results go to a new `specs/002-template-management-ui/verification-log.md`. Record real results only, and list anything not run.

- [ ] T001 ⛔ GATE-0 ⛔ GATE-1 Run the baseline suite `.\.venv\Scripts\python.exe -m pytest -q` and record the passed count (expected 475, unverified) in `specs/002-template-management-ui/verification-log.md` (FR-091 AC4, SC-010)
- [ ] T002 ⛔ GATE-0 ⛔ GATE-1 Check whether flake8, mypy and Poetry are installed in `.venv` and record the result. Do not install them without GATE-1 approval (NFR-007, D-27)
- [ ] T003 ⛔ GATE-0 ⛔ GATE-1 Convert `data/synthetic/ccda/TD-CCDA-01` to `TD-CCDA-12` (excluding `TD-CCDA-08`, which fails by design), `tests/data/ccda/CCD.ccda` and `tests/data/ccda/History_and_Physical.ccda` with `--template-name CCD --source-format CCDA` into `$env:TEMP\ui-before` (never `data/out`), and record file hashes only in `specs/002-template-management-ui/verification-log.md` (FR-091 AC1, plan 9.5)
- [ ] T004 ⛔ GATE-0 ⛔ GATE-1 Capture `.\.venv\Scripts\python.exe -m fhir_converter --help` output to `$env:TEMP\ui-help-before.txt` for later comparison (FR-091 AC3)
- [ ] T005 [P] Record the starting inventory in `specs/002-template-management-ui/verification-log.md`: count of `.liquid` files under `fhir_converter/templates/ccda` and `.../stu3`, and confirm `fhir_converter/ui/` does not yet exist (FR-010 AC2, A12). Counts are for comparison only and are never hard-coded in tests
- [ ] T006 [P] Investigation I-6: read the C-CDA section templates under `fhir_converter/templates/ccda/Section/` and list the literal call forms that carry the section template id (OID). Add the findings to `specs/002-template-management-ui/research.md` (FR-031 AC2, plan 6.3 step 6)
- [ ] T007 [P] Investigation I-2 (read part): read `BaseLoader.get_source`, `BaseLoader.load`, `ChoiceLoader.get_source` in python-liquid 1.12.1 and `TemplateSystemLoader.get_source` in `fhir_converter/loaders.py` end to end. Record whether `get_source` and `read_text` avoid parsing, in `specs/002-template-management-ui/research.md` (D-19, FR-083). Preliminary reading only found `load` parsing after `get_source`
- [ ] T008 ⛔ GATE-0 ⛔ GATE-11 Investigation I-2 (run part): in a temporary script outside the repository, build `make_environment` over a temporary folder with a template holding a deliberate Liquid syntax error, call `read_text` (expect text, no error) and `get_template` (expect an error), then record the outcome and close D-19 in `specs/002-template-management-ui/research.md` (FR-083 AC2)
- [ ] T009 ⛔ GATE-0 ⛔ GATE-11 Investigation I-4: prototype the standard-library server in a temporary folder (port 0 binding, HEAD handling, clean `shutdown`, silencing `log_message` and `log_error`, `ThreadingHTTPServer` on the available Python versions) and record the results in `specs/002-template-management-ui/research.md` (FR-005, FR-007, COMP-004, TUI-COMP-008)
- [ ] T010 ⛔ GATE-0 ⛔ GATE-12 Investigation I-3: list which Python 3.8 to 3.12 interpreters exist locally and record them. Do not install any without GATE-12 approval. Record that the `.venv` is 3.14.10 and CI is the compatibility authority (COMP-001, N-06)
- [ ] T011 [P] Investigation I-1 (read part): read the full `pyproject.toml` and `.gitignore` for packaging inclusion rules, and confirm whether `MANIFEST.in` exists. Record in `specs/002-template-management-ui/research.md` (FR-094 evidence, R-7)
- [ ] T012 ⛔ GATE-0 ⛔ GATE-2 Investigation I-1 (early build): in a temporary copy of the repository outside the working tree, add placeholder files under `fhir_converter/ui/static/`, build the sdist and wheel into a temporary folder, list both archives, install the wheel into a clean temporary virtual environment and confirm the placeholders are packaged. Record the result in `specs/002-template-management-ui/verification-log.md`. If a file is missing, only write a proposal (FR-094 AC1, AC3, AC4, COMP-006, scenario T41)
- [ ] T013 ⛔ GATE-0 Obtain and record developer rulings in `specs/002-template-management-ui/spec.md` (Decision Register) for the items that Phases 2 to 8 depend on: D-04, D-06, D-08, D-09, D-10, D-11, D-12, D-13, D-14, D-15, D-16, D-17, D-20, D-25, N-01, N-04. Any item left open keeps its provisional behavior from plan 6.2 and its gate on the tasks below

**Checkpoint**: baseline numbers, investigation results and rulings are recorded. Gated items are either cleared or explicitly deferred.

---

## Phase 2: UI package and catalog

**Purpose**: Foundation shared by every story. The catalog is the only module that touches the filesystem (FR-092 AC2). Blocks Phases 3 to 8.

- [ ] T014 ⛔ GATE-0 Create `fhir_converter/ui/__init__.py` with a docstring only and no import of the server, so `import fhir_converter` never loads the UI (FR-001 AC3, FR-092 AC1, COMP-007)
- [ ] T015 [P] ⛔ GATE-0 Create `fhir_converter/ui/errors.py` with the fixed error codes and fixed messages from `contracts/ui-contract.md` (`BAD_REQUEST`, `NOT_FOUND`, `UNAVAILABLE`, `FORBIDDEN`, `METHOD_NOT_ALLOWED`), one mapping from internal failure kinds to them, and the fixed notes the page shows (best-effort, restart, user directory not tied to a format). It never carries exception text, stack traces, absolute paths or the rejected input value (FR-070 AC1, AC3, SEC-007, SEC-008, FR-042)
- [ ] T016 [P] ⛔ GATE-0 ⛔ GATE-7 Create `fhir_converter/ui/security.py` containing only the named limit constants for now: maximum displayed file size, maximum search results, maximum query length, search time bound, request timeout, displayed tag-text length. Mark every value as a provisional placeholder pointing at D-16 (TUI-NFR-010, plan 6.2)
- [ ] T017 [P] ⛔ GATE-0 Create `fhir_converter/ui/scanner.py` containing only the frozen dataclasses `Reference` (fields from data-model.md: `kind` literal, dynamic or unsupported; `form` include, batch_render, evaluate or other; `target` for literal references only; `line` 1-based; `text` length-capped) and `ScanResult` (`references` plus `literal_count`, `dynamic_count`, `unsupported_count`). Python 3.8 typing only (data-model.md, FR-040, FR-045)
- [ ] T018 ⛔ GATE-0 Create `fhir_converter/ui/catalog.py` with the frozen dataclass `TemplateEntry` with fields `logical_name`, `origin` ("package" or "user"), `format` (CCDA, STU3 or none), `category`, `size`, `relative_path` (never absolute), `absolute_path` (held internally only, never returned), `section_id` (or none), `scan` (data-model.md, FR-012, FR-031 AC1, SEC-008). Depends on T017
- [ ] T019 ⛔ GATE-0 ⛔ GATE-15 Implement the logical-name mapping in `fhir_converter/ui/catalog.py`: forward slashes, a name with more than one path part has `_` removed from its last segment and then `.liquid` removed, a single-part file keeps its name. Re-implement the rule and do not import the loader's private `_resolve_template_name` (FR-013 AC1, AC2, AC4, D-14)
- [ ] T020 [P] ⛔ GATE-0 Create the shared test helper `tests/ui_fixtures.py` that builds the fixture catalog trees in `tmp_path`, using the invented fixture templates listed in plan section 9 (`CCD`, `Header`, `Section/Alpha`, `Entry/Alpha/entry`, `Resource/Alpha`, `Section/Beta`, `Section/OnlyViaSection`, `Section/Orphan`, `Section/Dyn`, `Utils/CycleA`, `Utils/CycleB`, `Utils/SelfRef`, an inert-content template, a syntax-error template, `.json` `.xsl` `.txt` files, and the user tree with `CCD.liquid`, `Section/UserOnly` and `Custom.liquid`). Synthetic data only (constitution III, SEC-012)
- [ ] T021 ⛔ GATE-0 Implement enumeration of the packaged `ccda` and `stu3` template folders in `fhir_converter/ui/catalog.py`, located from the `fhir_converter` package directory on disk, `.liquid` files only, sorted by logical name. Note the unzipped-install assumption (N-05) in the module docstring (FR-010 AC1 to AC3, FR-014 AC1, NFR-005)
- [ ] T022 ⛔ GATE-0 ⛔ GATE-8 Implement category derivation in `fhir_converter/ui/catalog.py`: C-CDA top-level files are "root", known folders `Section`, `Entry`, `Resource`, `Reference`, `DataType`, `Extension`, `Utils`, `ValueSet` use the folder name, anything else is "other"; STU3 is "root" or the first folder name. Provisional until D-08 is decided, and a category with no `.liquid` files must not error (FR-011 AC1 to AC4)
- [ ] T023 ⛔ GATE-0 ⛔ GATE-8 Implement user directory enumeration in `fhir_converter/ui/catalog.py`: `.liquid` only (a `.json` file is ignored), resolve the root once, refuse any file whose resolved path leaves the root (symlink escape), skip and count user files in a sub-folder without the `_` prefix without listing their names (provisional, D-09, N-02) (FR-003 AC1, FR-014 AC3, SEC-005, SEC-006)
- [ ] T024 ⛔ GATE-0 Implement user directory startup validation in `fhir_converter/ui/catalog.py`: must exist, be a directory and be readable, otherwise return a fixed failure kind with no exception text (FR-004 AC1)
- [ ] T025 ⛔ GATE-0 ⛔ GATE-7 Implement `read_source` in `fhir_converter/ui/catalog.py`: read a catalogued file's text from disk at request time as UTF-8, re-check containment, enforce the size cap, and return a fixed failure kind for an unreadable, removed, oversized or non-UTF-8 file. Take the path from the catalog entry only, never from a request value (FR-006 AC3, AC4, FR-030 AC1, SEC-003, SEC-006, D-17)
- [ ] T026 ⛔ GATE-0 Implement `build_catalog(user_dir)` in `fhir_converter/ui/catalog.py` returning an immutable `Catalog` with per-format effective entries (user wins), packaged entries, user entries, a recorded build time, and the unmapped-file count. Both formats are built at startup and there is no refresh method (FR-006 AC1, AC2, FR-051 AC5, NFR-006). Depends on T019 to T025
- [ ] T027 [P] ⛔ GATE-0 Create `tests/test_ui_compat.py`: parse every module under `fhir_converter/ui/` with `ast.parse(..., feature_version=(3, 8))` and fail on the avoid list (built-in generics evaluated at runtime, the `X | Y` union syntax, `match`, `str.removeprefix`, `str.removesuffix`, `functools.cache`, parenthesized multi-item `with`) (COMP-001, COMP-002, I-3)
- [ ] T028 [P] ⛔ GATE-0 Test logical-name mapping in `tests/test_ui_catalog.py`: root, nested and `_`-prefixed names, forward slashes only, and a check for every fixture template that the logical name resolves to the same file through `make_environment` and `read_text` without using the private method (FR-013 AC1 to AC4, scenario T04)
- [ ] T029 [P] ⛔ GATE-0 Test enumeration in `tests/test_ui_catalog.py`: counts per format equal the `.liquid` files on disk, `.json`, `.xsl` and `.txt` files are not listed or counted, deterministic order, one category per template and category counts sum to the total, an empty category does not error (FR-010, FR-011, FR-014 AC1 to AC3, scenarios T01, T02, T25)
- [ ] T030 [P] ⛔ GATE-0 Test the user directory in `tests/test_ui_catalog.py`: missing path, a file instead of a directory, an unreadable directory, a `.json` file ignored, a symlink leaving the root refused (skip and report where the platform cannot create symlinks), unprefixed sub-folder files counted (FR-003, FR-004, SEC-005, scenarios T26, T33)
- [ ] T031 [P] ⛔ GATE-0 Test `read_source` and the startup snapshot in `tests/test_ui_catalog.py`: a file added after build is absent, a file removed after build is still listed and then gives the fixed failure kind, the oversized and non-UTF-8 cases, and `Catalog` has no refresh method (FR-006 AC1, AC4, scenario T36)

**Checkpoint**: the catalog builds, maps names, enforces scope and reads files safely, and the 3.8 syntax check runs. Stories can start.

---

## Phase 3: Template listing and search (US1)

**Goal**: A developer lists templates by format and category and finds them by name or by text in the source.

**Independent Test**: against the fixture catalog, list, category filter, name search and source search return exactly the expected templates and write nothing.

- [ ] T032 [P] [US1] ⛔ GATE-0 Implement the list function in `fhir_converter/ui/search.py`: for a format, filter by category, origin and the override flag, return logical name, category, origin and size sorted by logical name, and reject an unknown format or category with the fixed `BAD_REQUEST` kind (FR-010, FR-011 AC3, FR-012, NFR-005)
- [ ] T033 [P] [US1] ⛔ GATE-0 ⛔ GATE-7 Implement name search in `fhir_converter/ui/search.py`: case-insensitive substring on the logical name, empty query returns the full filtered list, combines with format, category and origin filters, query longer than the limit is rejected (FR-020 AC1 to AC4)
- [ ] T034 [US1] ⛔ GATE-0 ⛔ GATE-7 Implement source search in `fhir_converter/ui/search.py` using `catalog.read_source`: stop at the result cap or time bound and report the cut-off, skip oversized, unreadable and non-UTF-8 files and report how many were skipped, never change a file or the catalog (FR-021 AC1 to AC6, TUI-NFR-002). Depends on T025
- [ ] T035 [P] [US1] ⛔ GATE-0 Test listing and filters in `tests/test_ui_search.py`: per-format lists, category and origin filters, unknown format and category rejected (FR-010, FR-011, FR-012, scenarios T01, T02, T03)
- [ ] T036 [P] [US1] ⛔ GATE-0 Test name search in `tests/test_ui_search.py`: match, no match shows an empty result, empty query, case difference, overlong query rejected (FR-020, scenario T05)
- [ ] T037 [P] [US1] ⛔ GATE-0 Test source search in `tests/test_ui_search.py`: match and no match, result cap reached and reported, oversized file skipped and counted, unreadable and non-UTF-8 files skipped without exposing exception text (FR-021, scenario T06)

**Checkpoint**: US1 listing and search work and are tested without a server.

---

## Phase 4: Source and metadata viewer (US1)

**Goal**: A developer opens a template and reads its source as inert text with metadata.

**Independent Test**: open fixture templates and compare the displayed text with the file, including an inert-content template and an oversized one.

- [ ] T038 [P] [US1] ⛔ GATE-0 Implement metadata assembly in `fhir_converter/ui/catalog.py`: format, logical name, category, origin, size and the path relative to its root, never an absolute path (FR-031 AC1, SEC-008)
- [ ] T039 [US1] ⛔ GATE-0 ⛔ GATE-17 Implement section id detection `detect_section_id` in `fhir_converter/ui/scanner.py` using the call forms recorded by T006, reporting "not detected" when no literal is found and never an invented value (FR-031 AC2, **[APPROVAL]**)
- [ ] T040 [US1] ⛔ GATE-0 Implement the viewer payload in `fhir_converter/ui/catalog.py`: the source text, or "too large to display" with the size, and the fixed failure kind for a name not in the catalog, a removed file or a non-UTF-8 file (FR-030 AC1, AC3). Depends on T025
- [ ] T041 [P] [US1] ⛔ GATE-0 Test metadata and section id in `tests/test_ui_catalog.py`: fields present, no absolute path in any value, "not detected" when no id (FR-031, scenario T08)
- [ ] T042 [P] [US1] ⛔ GATE-0 Test the viewer in `tests/test_ui_catalog.py`: displayed text equals the file, too-large case, a name not in the catalog, an inert-content fixture holding script, HTML and Liquid tags is returned as plain text and untouched (FR-030 AC1 to AC3, scenario T07)
- [ ] T043 [P] [US1] ⛔ GATE-0 Add real-package spot checks to `tests/test_ui_catalog.py`: `Section/Goal` maps to `Section/_Goal.liquid` and shows its template id, and per-format counts follow the files on disk. Assert nothing about which sections are wired or highlighted (FR-013, FR-031, FR-093 AC3, scenario T27)

**Checkpoint**: US1 data layer complete. With the server and page (Phases 7 and 8) US1 is demonstrable.

---

## Phase 5: Reference scanner and relationship graph (US2)

**Goal**: A developer sees what a template references and what references it, directly and transitively, with dynamic and unsupported references shown separately.

**Independent Test**: on the fixture catalog, the three reference forms are found with the right form and line, direct and transitive lists match the expected sets, the four states are distinct, and no dynamic or unsupported reference is dropped.

- [ ] T044 [US2] ⛔ GATE-0 ⛔ GATE-9 Implement comment-block blanking in `fhir_converter/ui/scanner.py`: replace the text between `{% comment %}` and `{% endcomment %}` with newlines so line numbers stay correct, and treat `{% raw %}` blocks as proposed in D-04 (FR-040 AC6)
- [ ] T045 [P] [US2] ⛔ GATE-0 ⛔ GATE-9 Implement `include` detection in `fhir_converter/ui/scanner.py`: the quoted name is the reference, including whitespace-control variants and tags with parameters after the name (FR-040 AC1)
- [ ] T046 [P] [US2] ⛔ GATE-0 ⛔ GATE-9 Implement `evaluate <var> using <name>` detection in `fhir_converter/ui/scanner.py`: the name after `using` is the reference (FR-040 AC3)
- [ ] T047 [P] [US2] ⛔ GATE-0 ⛔ GATE-9 Implement `batch_render:` detection in `fhir_converter/ui/scanner.py`: the first argument is the reference and the second argument is a variable name and is never recorded as a reference (FR-040 AC2)
- [ ] T048 [US2] ⛔ GATE-0 ⛔ GATE-9 Implement classification in `fhir_converter/ui/scanner.py`: single-quoted literal is literal, double-quoted literal only if D-04 adopts it, anything else dynamic, and a `render` tag, an unreadable construct or an unterminated reference tag is unsupported, never silently dropped (FR-045 AC1, AC2, AC4). Depends on T044 to T047
- [ ] T049 [US2] ⛔ GATE-0 ⛔ GATE-7 Implement line numbers and capped tag text in `fhir_converter/ui/scanner.py`, and the public `scan_template(text)` returning a `ScanResult` with counts. The module must not import python-liquid and must not call any parse, compile or render function (FR-040 AC5, AC7, FR-045 AC3, AC5, FR-083)
- [ ] T050 [P] [US2] ⛔ GATE-0 Test the scanner forms in `tests/test_ui_scanner.py`: one reference of each form found with the right form and line, second `batch_render` argument not reported, comment-block reference not reported, a name in a plain string not reported (FR-040, scenario T09)
- [ ] T051 [P] [US2] ⛔ GATE-0 Test dynamic and unsupported handling in `tests/test_ui_scanner.py`: `include some_variable`, a `render` tag and an unterminated reference tag each listed with form and line, none dropped, no invented target (FR-045, scenario T14)
- [ ] T052 [P] [US2] ⛔ GATE-0 Test scanner isolation in `tests/test_ui_scanner.py`: `fhir_converter/ui/scanner.py` has no python-liquid import, and scanning calls no compile or render function (FR-040 AC7, FR-083, scenario T23)
- [ ] T053 [US2] ⛔ GATE-0 ⛔ GATE-15 Implement the effective graph per format in `fhir_converter/ui/graph.py`: adjacency and reverse adjacency from effective sources, each literal target marked found, not found, or outside catalog (a non-`.liquid` target), deduplicated per target with the forms used, sorted by logical name (FR-041 AC1 to AC3, AC5, FR-014 AC4, D-15). Depends on T026, T049
- [ ] T054 [US2] ⛔ GATE-0 Implement transitive relationships in `fhir_converter/ui/graph.py`: breadth-first search with a parent map giving a shortest chain per entry, both directions, transitive equals reachable minus direct minus the template itself, a visited set so cycles and self references terminate (FR-043 AC1 to AC5)
- [ ] T055 [US2] ⛔ GATE-0 Implement the four-state relationship view in `fhir_converter/ui/graph.py` with separate fields `direct`, `transitive`, `dynamic_or_unsupported` and `no_references_detected` (true only with zero literal and zero dynamic or unsupported entries), plus on the referenced-by side the catalog-wide dynamic count and an `incomplete` flag (FR-044 AC1 to AC4)
- [ ] T056 [P] [US2] ⛔ GATE-0 Test direct and inverse relationships in `tests/test_ui_graph.py`: used-by is the exact inverse, found, not found and outside-catalog states (a `.json` target), sorted, deduplicated (FR-041, FR-014 AC4, scenarios T09, T10)
- [ ] T057 [P] [US2] ⛔ GATE-0 Test transitive relationships in `tests/test_ui_graph.py`: `CCD` has direct `Section/Alpha` and transitive `Entry/Alpha/entry` and `Resource/Alpha` with chains, direct targets are not repeated, a two-template cycle and a self reference terminate, a template is never reachable from itself (FR-043, scenarios T11, T12)
- [ ] T058 [P] [US2] ⛔ GATE-0 Test the four states in `tests/test_ui_graph.py`: distinct fields, a dynamic-only template shows dynamic and not "no references detected", the referenced-by side shows the catalog-wide count and incompleteness note when the count is above zero (FR-044, FR-045 AC5, scenarios T13, T15)
- [ ] T059 [US2] ⛔ GATE-0 Investigation I-5: compare the scanner's literal totals over the packaged templates with the recorded grep counts (619 `include`, 188 `batch_render`, 24 `evaluate`), explain any difference, and confirm nothing literal is mislabelled dynamic. Record in `specs/002-template-management-ui/verification-log.md`. Record only, do not assert the counts in a test (plan I-5, A3, A12). Depends on T049

**Checkpoint**: US2 complete and testable without a server.

---

## Phase 6: Overrides and section analysis (US3, US4)

**Goal**: A developer sees which user templates override packaged ones, and which C-CDA sections no root template appears to reach, with best-effort wording.

**Independent Test**: on the fixture catalog, override flags and counts are correct in each format view, an orphan section is highlighted while reached sections are not, and every highlight carries the best-effort note.

- [ ] T060 [P] [US3] ⛔ GATE-0 Implement override detection in `fhir_converter/ui/catalog.py`: for the selected format a user template with the same logical name as a packaged one is "overrides packaged", the packaged one is "overridden by user template", a user template with no packaged counterpart is "user only", with an override count and filter, using the mapping from T019 (FR-050 AC1 to AC5)
- [ ] T061 [US3] ⛔ GATE-0 Implement the user-selected format comparison in `fhir_converter/ui/catalog.py` and `fhir_converter/ui/graph.py`: the same user directory content appears in both format views, overrides, relationships, root usage and section flags are computed for the selected format only, nothing is inferred from the user directory's contents (FR-051 AC1 to AC5)
- [ ] T062 [US3] ⛔ GATE-0 ⛔ GATE-15 Implement the effective-source rule in `fhir_converter/ui/graph.py`: an overridden template's relationships come from the user source and the packaged version's own references are returned in a separate `packaged_version` block (FR-041 AC4, D-15)
- [ ] T063 [P] [US3] ⛔ GATE-0 Test overrides in `tests/test_ui_catalog.py` and `tests/test_ui_graph.py`: a user `CCD.liquid` is "overrides packaged" in the C-CDA view and "user only" in the STU3 view, both sources viewable, the override filter and count, a similarly named user file not flagged, effective-graph relationships from the user source with `packaged_version` shown (FR-050, FR-051, scenarios T11, T16, T17)
- [ ] T064 [US4] ⛔ GATE-0 ⛔ GATE-15 Implement root derivation in `fhir_converter/ui/graph.py` for C-CDA: top-level effective templates including `Header` and any top-level user template, derived from the catalog and never a fixed list (FR-060 AC4, D-06)
- [ ] T065 [US4] ⛔ GATE-0 Implement root reach in `fhir_converter/ui/graph.py`: multi-source breadth-first search from all roots, and per template the roots that reach it split into directly and transitively (FR-043 AC6)
- [ ] T066 [US4] ⛔ GATE-0 Implement section flags in `fhir_converter/ui/graph.py`: a `Section/` template no root reaches directly or transitively is "Potentially unreferenced (best-effort)", with "analysis may be incomplete" and the count when any root or reached template has dynamic or unsupported references, no flag for STU3, a count and a filter (FR-060 AC1 to AC6, FR-042 AC1, AC2)
- [ ] T067 [US4] ⛔ GATE-0 Make `fhir_converter/ui/graph.py` free of any named section: no reference to Goals, Health Concerns or Instructions or any other real section name (FR-060 AC7, FR-093 AC1, AC2)
- [ ] T068 [P] [US4] ⛔ GATE-0 Test section analysis in `tests/test_ui_graph.py`: `Section/Orphan` is highlighted, `Section/Beta` and `Section/OnlyViaSection` are not, a user root `Custom` that references `Section/Orphan` makes it not highlighted, a section reached only through a dynamic include is highlighted with the incompleteness note, no highlight in the STU3 view (FR-060, scenarios T18, T19)
- [ ] T069 [P] [US4] ⛔ GATE-0 Test Goals independence in `tests/test_ui_graph.py`: the analysis tests give identical results on a fixture tree with and without a wired `Section/Goal` and no test names a real section as highlighted (FR-093, SC-013, scenario T34)

**Checkpoint**: US3 and US4 complete and testable without a server.

---

## Phase 7: HTTP server and security (US5)

**Goal**: The data layer is served read-only, on the local machine only, with checks that other pages and processes cannot use it, fixed errors, and no console noise.

**Independent Test**: start the server on port 0 in a test thread and confirm the bound address, the header and token checks, rejected methods and routes, the generic errors, and unchanged files after a full session.

- [ ] T070 [US5] ⛔ GATE-0 ⛔ GATE-15 Implement Host validation in `fhir_converter/ui/security.py`: allowed values are the bound loopback address with the chosen port and, if D-10 is approved, `localhost` with that port, anything else is rejected with the generic `FORBIDDEN` kind (SEC-002, D-10)
- [ ] T071 [P] [US5] ⛔ GATE-0 Implement Origin validation in `fhir_converter/ui/security.py`: when an `Origin` header is present it must equal the server's own origin (SEC-002)
- [ ] T072 [US5] ⛔ GATE-0 ⛔ GATE-6 Implement the per-run token in `fhir_converter/ui/security.py`: created at startup from the standard-library `secrets` module, compared in constant time, never logged, never written to a file, different on every start (SEC-018, D-11)
- [ ] T073 [P] [US5] ⛔ GATE-0 Implement request value validation in `fhir_converter/ui/security.py`: format, category and origin checked against fixed sets, names accepted only by dictionary lookup in the catalog, and rejection of `..`, absolute paths, drive letters, backslashes, NUL and URL-encoded and double-encoded forms of these (SEC-003, SEC-004)
- [ ] T074 [P] [US5] ⛔ GATE-0 Implement the extension and size policy in `fhir_converter/ui/security.py`: only `.liquid` is served, files above the cap are not returned (SEC-006, FR-014 AC2)
- [ ] T075 [P] [US5] ⛔ GATE-0 Test the pure security functions in `tests/test_ui_security.py`: allowed and foreign Host, wrong port, foreign Origin, absent Origin, token create and compare, the traversal table (plain, `%2e%2e`, double-encoded, absolute, drive letter, backslash, NUL, overlong), extension policy (SEC-002, SEC-004, SEC-006, SEC-018, scenarios T16, T24, T30)
- [ ] T076 [US5] ⛔ GATE-0 Create `fhir_converter/ui/server.py` with the handler skeleton: GET and HEAD only, any other method gets the generic `METHOD_NOT_ALLOWED` response and changes nothing, unknown routes get `NOT_FOUND`, no route reads a request body (FR-080 AC1, AC3, FR-082, SEC-009)
- [ ] T077 [US5] ⛔ GATE-0 ⛔ GATE-5 Override `log_message` and `log_error` in `fhir_converter/ui/server.py` so no request line, URL, query string, template name, path or token is written to the console (FR-007 AC1, AC3, SEC-014, D-20)
- [ ] T078 [US5] ⛔ GATE-0 Implement the API routes in `fhir_converter/ui/server.py` per `contracts/ui-contract.md`: `/api/meta`, `/api/templates`, `/api/template`, `/api/source`, `/api/search/source`, `/api/sections/unreferenced`, names and filters as query values only, JSON responses, and no route that renders, compiles, accepts documents, writes or refreshes (FR-006 AC2, FR-080 to FR-083, SEC-003)
- [ ] T079 [US5] ⛔ GATE-0 ⛔ GATE-14 Implement the static routes in `fhir_converter/ui/server.py`: `/`, `/static/app.js` and `/static/style.css` from an explicit three-file allowlist read from `fhir_converter/ui/static/`, anything else under `/static/` is `NOT_FOUND`, no directory listing (plan 6.8, D-25)
- [ ] T080 [US5] ⛔ GATE-0 ⛔ GATE-16 Add the response headers in `fhir_converter/ui/server.py`: same-origin `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `Cache-Control: no-store`, `Referrer-Policy: no-referrer`, and no CORS headers. These are **[PROPOSED]** and not in the spec (N-04, SEC-011, SEC-013)
- [ ] T081 [US5] ⛔ GATE-0 Enforce the checks in `fhir_converter/ui/server.py`: Host and Origin on every request, the token header on every `/api/` request, each failure returning only the fixed message and code, and every internal exception mapped to a fixed error with no exception text, stack trace or absolute path (FR-070 AC1 to AC4, SEC-002, SEC-007, SEC-018). Depends on T070 to T074
- [ ] T082 [US5] ⛔ GATE-0 ⛔ GATE-7 Implement `create_server(config)` in `fhir_converter/ui/server.py` returning the server and the token, bound to `127.0.0.1` with port 0 by default, with the request timeout applied, and no option to bind another address (FR-002 AC1, AC2, FR-005 AC1, SEC-001)
- [ ] T083 [US5] ⛔ GATE-0 ⛔ GATE-4 Create `fhir_converter/ui/__main__.py`: parse the optional user directory using the option name decided by D-12, validate it with T024 before opening any socket (non-zero exit, fixed message, no listener left), build the catalog, start the server, and leave `fhir_converter/__main__.py` untouched (FR-001 AC2, FR-003, FR-004 AC1, AC2)
- [ ] T084 [US5] ⛔ GATE-0 ⛔ GATE-5 ⛔ GATE-6 Print the single startup message from `fhir_converter/ui/__main__.py` with the address, port and token-bearing address, and nothing else during a session apart from fatal startup errors (FR-001 AC1, FR-005 AC1, FR-007 AC2, SEC-014, SEC-019)
- [ ] T085 [US5] ⛔ GATE-0 ⛔ GATE-3 Add an explicit port option to `fhir_converter/ui/__main__.py` **only if D-13 is approved**: a busy port gives a non-zero exit and a fixed message and no listener remains. Without approval, do not expose any port option (FR-005 AC3, D-13)
- [ ] T086 [P] [US5] ⛔ GATE-0 Test routes in `tests/test_ui_server.py` using `create_server` on port 0 in a thread and `http.client`: JSON shapes match the contract, the three static files are served, other static paths are not found, a refresh route does not exist, the build time and restart note are in `/api/meta` (FR-006 AC2, AC5, scenarios T07, T22, T36)
- [ ] T087 [P] [US5] ⛔ GATE-0 Test methods and read-only behavior in `tests/test_ui_server.py`: POST, PUT, PATCH and DELETE are rejected with the generic error, no route accepts input data, and a hash of the packaged template tree and the user directory is identical before and after a full session (FR-080 AC2, AC3, FR-082, SEC-009, SEC-013, scenarios T14, T22)
- [ ] T088 [P] [US5] ⛔ GATE-0 Test errors in `tests/test_ui_server.py`: each failure case returns only the fixed message and code, and an injected internal exception holding a distinctive marker string never appears in the body, headers, captured output or any response, nor does the fixture's absolute temporary path (FR-070 AC1 to AC4, SEC-007, SEC-008, scenarios T13, T21)
- [ ] T089 [P] [US5] ⛔ GATE-0 Test request protection in `tests/test_ui_server.py`: foreign Host, wrong port, foreign Origin, missing token, wrong token and the correct token each behave as required, and the bound address is `127.0.0.1` with no option to change it (FR-002, SEC-001, SEC-002, SEC-018, scenarios T19, T29, T30, T31)
- [ ] T090 [P] [US5] ⛔ GATE-0 Test no rendering and no compile in `tests/test_ui_server.py`: instrument the renderer methods and `get_template` and confirm zero calls during a full session, a syntax-error fixture is listed and viewable as text and not flagged invalid, and no UI text claims a template is valid or compiles (FR-081, FR-083 AC1 to AC3, scenario T23)
- [ ] T091 [P] [US5] ⛔ GATE-0 Test the process behavior in `tests/test_ui_process.py` using `subprocess` and `python -m fhir_converter.ui`: the startup message and timing, two simultaneous instances get different ports and both answer, a bad user directory (missing, a file, unreadable) exits non-zero with the fixed message and leaves no listener, captured standard output and error across a full session equal the startup message only, and `import fhir_converter` does not load `fhir_converter.ui` (FR-001, FR-004, FR-005, FR-007, scenarios T01, T32, T33, T35, T39)

**Checkpoint**: the read-only server runs on loopback with all checks and fixed errors. US5 testable end to end.

---

## Phase 8: UI integration

**Goal**: The plain HTML, CSS and JavaScript page puts every story on screen, with text-only output and no external resources.

**Independent Test**: start the UI against the fixture catalog and a fixture user directory and walk through the checks in `quickstart.md`.

- [ ] T092 [US1] ⛔ GATE-0 ⛔ GATE-14 Create `fhir_converter/ui/static/index.html`: a format control (default C-CDA), category, origin and override filters, a name search box, a source search box, a list area, a template view area, relationship area, an unreferenced-section view, and a notes area for the fixed notes. No external URLs, fonts or scripts (FR-010, FR-051 AC1, FR-090 AC2, D-25)
- [ ] T093 [US1] ⛔ GATE-0 ⛔ GATE-14 ⛔ GATE-6 Create `fhir_converter/ui/static/app.js` core: read the token from the address fragment, send it in the request header on every API call, load the list and the metadata, insert all text with `textContent`, never `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `eval` or `document.write` (SEC-002, SEC-011, SEC-018, FR-090)
- [ ] T094 [US1] ⛔ GATE-0 Extend `fhir_converter/ui/static/app.js` with the template view: metadata and source shown in a `pre` element as inert text, with the "too large to display" case and the fixed error message display (FR-030 AC2, FR-031, FR-070)
- [ ] T095 [US2] ⛔ GATE-0 Extend `fhir_converter/ui/static/app.js` with the relationship panel: separate headings and wording for direct references, transitively reachable templates with their chains, "No references detected", and dynamic or unsupported references with form and line, plus the referenced-by side and the catalog-wide dynamic note, and the fixed best-effort note on every relationship view (FR-042 AC1, AC2, FR-044 AC1 to AC5)
- [ ] T096 [US3] ⛔ GATE-0 Extend `fhir_converter/ui/static/app.js` with the override view: "overrides packaged", "overridden by user template" and "user only" labels, the override filter and count, both sources viewable, and the statement that user directory templates are not tied to a format (FR-050, FR-051 AC2)
- [ ] T097 [US4] ⛔ GATE-0 Extend `fhir_converter/ui/static/app.js` with the unreferenced-sections view for C-CDA only, using only "Potentially unreferenced (best-effort)" wording and showing the incompleteness note and count (FR-042 AC2, FR-060 AC2, AC3, AC5)
- [ ] T098 [US5] ⛔ GATE-0 Extend `fhir_converter/ui/static/app.js` with the build time and the "restart to refresh" note from `/api/meta`, and no refresh control (FR-006 AC2, NFR-006)
- [ ] T099 [P] [US1] ⛔ GATE-0 Create `fhir_converter/ui/static/style.css`: visible keyboard focus, semantic structure support, the four relationship states distinguishable by text and not by color alone, no external fonts or images (TUI-NFR-003, FR-090 AC2)
- [ ] T100 [P] [US5] ⛔ GATE-0 Test the static files in `tests/test_ui_security.py`: `fhir_converter/ui/static/` contains no `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `eval`, `document.write` or external `http`/`https` URL, and the page text uses only the allowed relationship and section wording (SEC-011, FR-090 AC2, scenario T37)
- [ ] T101 [P] [US2] ⛔ GATE-0 Test the wording rules in `tests/test_ui_security.py`: scan the static files and the server's fixed notes for the forbidden definitive words (`unused`, `dead`, `orphan`, `orphaned`, `unreferenced` without "potentially", `no references`) and for any validity or compliance claim (FR-042 AC2, AC3, TUI-NFR-012, D-18, scenario T20)
- [ ] T102 [US5] ⛔ GATE-0 Test output encoding in `tests/test_ui_security.py`: a fixture template whose name and body contain HTML, script and attribute-breaking characters is returned as data and the page script inserts it only as text (SEC-011, scenario T07)
- [ ] T103 ⛔ GATE-0 Manual check, recorded in `specs/002-template-management-ui/verification-log.md`: keyboard-only walk through list, search, view, relationships, overrides and highlights, and a look in the browsers listed by NFR-004 (NFR-003, NFR-004, scenario T42). Record real results, and note any browser not tried

**Checkpoint**: every story is reachable from the page, text-only and local.

---

## Phase 9: Tests and regression verification

**Purpose**: Prove that nothing existing changed and that compatibility holds. This phase has no story label because it spans all stories.

- [ ] T104 ⛔ GATE-0 ⛔ GATE-1 Create `tests/test_ui_regression.py` recording the `inspect.signature` strings of `render_fhir_string`, `render_fhir`, `render_to_fhir`, `make_environment`, `render_to_dir` and `render_files_to_dir`, and asserting they match (FR-091 AC2). Record the expected strings from the unchanged code before any UI file exists
- [ ] T105 ⛔ GATE-0 ⛔ GATE-1 Re-convert the same named fixtures as T003 into `$env:TEMP\ui-after` and diff each file with `$env:TEMP\ui-before`. Expect zero differences and record the result (FR-091 AC1, SC-010, scenario T38)
- [ ] T106 ⛔ GATE-0 ⛔ GATE-1 Capture `python -m fhir_converter --help` again and compare with `$env:TEMP\ui-help-before.txt`. Rerun the TC-CLI cases in `docs/test-cases.md` by hand and compare with the recorded results, including TC-CLI-05 and TC-CLI-11 (FR-091 AC3, scenario T28)
- [ ] T107 ⛔ GATE-0 ⛔ GATE-1 Run the full suite `.\.venv\Scripts\python.exe -m pytest -q` and compare the passed count with T001: the baseline plus the new tests, none failing (FR-091 AC4, SC-010, scenario T38)
- [ ] T108 ⛔ GATE-0 ⛔ GATE-1 Run mypy with the project settings and flake8 (max line length 120, ignore `E203,E721`, complexity 10) over `fhir_converter/ui/` and the new tests, if installed and approved, otherwise record "not run, CI covers it" (NFR-007, COMP-001)
- [ ] T109 ⛔ GATE-0 Compatibility check for Python 3.8 to 3.12: run `tests/test_ui_compat.py` (AST feature-version 3.8 and avoid-list) locally, then confirm the full CI matrix (3.8 to 3.12 on Ubuntu, Windows and macOS) runs pytest, flake8 and mypy over the new code, including the server startup tests and the port-0 test on every version and platform. Local runs cannot prove 3.8 because `.venv` is 3.14.10. Record what was and was not run (COMP-001 to COMP-005, COMP-008, scenario T40)
- [ ] T110 ⛔ GATE-0 Goals independence run: run the UI test suite against a temporary copy of the templates with and without a wired `Section/Goal` and confirm identical results and no named-section assertion. This does not require the Goals feature to be merged (FR-093 AC3, AC4, SC-013, scenario T34)
- [ ] T111 ⛔ GATE-0 Run the security suite in full and check the coverage matrix: every SEC-001 to SEC-019 criterion is covered by a test or an explicit recorded review. Record gaps (SEC-001 to SEC-019, scenarios T16 to T21, T24 to T26, T29 to T31, T37)
- [ ] T112 ⛔ GATE-0 ⛔ GATE-13 Measure and record, without asserting: list and open timings, startup build time, transitive-view time and source-search time against the real package (NFR-001, NFR-002, NFR-013, SC-001, scenario T43). Report whether the proposed targets were met and leave the decision to D-24
- [ ] T113 ⛔ GATE-0 Traceability check in `specs/002-template-management-ui/verification-log.md`: confirm each of the 43 scenarios T01 to T43 maps to a test, a measurement or a recorded manual check, and each functional requirement has at least one scenario or review check (SC-014)

**Checkpoint**: regression, compatibility and security results are recorded, with anything not run listed.

---

## Phase 10: Documentation and packaging verification

**Purpose**: Document the feature and prove the static files ship.

- [ ] T114 [P] ⛔ GATE-0 Write `docs/template-ui.md`: what the UI is for, how to start it, the user directory option, that it is read-only and local only, the catalog is built at startup and a restart refreshes it, the three reference forms and what the scan does not detect, that relationship and unreferenced-section results are best-effort, and that nothing is rendered or validated. No FHIR conformance or compliance claim (NFR-009, NFR-012, NFR-014, FR-042)
- [ ] T115 [P] ⛔ GATE-0 Add a short link to `docs/template-ui.md` in the Templates section of `README.md` (NFR-009)
- [ ] T116 [P] ⛔ GATE-0 Add a `CHANGES.md` entry for the new read-only template UI, noting that no conversion output, public API or CLI behavior changed (NFR-009, FR-091)
- [ ] T117 ⛔ GATE-0 Add the new cases to `docs/test-cases.md` and set a status only for cases that were actually run, leaving the rest unset (NFR-009, `CLAUDE.md` section 6)
- [ ] T118 ⛔ GATE-0 ⛔ GATE-2 Packaging verification on the real assets (I-1 step 4): build the sdist and wheel into a temporary folder, list both archives, confirm `fhir_converter/ui/static/index.html`, `app.js` and `style.css` and every module under `fhir_converter/ui/` are present in each, install the wheel into a clean temporary virtual environment and run `python -m fhir_converter.ui` to confirm it starts and serves the three static files. Record the result (FR-094 AC1, AC4, COMP-006, scenario T41)
- [ ] T119 ⛔ GATE-0 ⛔ GATE-2 ⛔ GATE-18 If T118 finds any file missing, write the smallest `pyproject.toml` or ignore-file change as a proposal in `specs/002-template-management-ui/verification-log.md` and stop. Do not edit configuration without GATE-18 approval (FR-094 AC2, AC3)
- [ ] T120 ⛔ GATE-0 ⛔ GATE-10 Constitution check: re-run the Constitution Check table in `specs/002-template-management-ui/plan.md` against the finished work, record the developers' ruling on N-01 (new `ui` package versus principle I) and note that the constitution is still an unratified draft unless D-26 was decided (plan Constitution Check, D-26)
- [ ] T121 ⛔ GATE-0 Run every step of `specs/002-template-management-ui/quickstart.md`, record real results, list anything not run, and update the Decision Register in `specs/002-template-management-ui/spec.md` and the checklist `specs/002-template-management-ui/checklists/requirements.md` to reflect decisions made. No FHIR-valid or conformance statement (SC-001 to SC-014)

---

## Dependencies and execution order

### Phase dependencies

- **Phase 1** has no code dependency. T001 to T004 and T012 need their gates. T006, T007, T011, T005 are read-only and can start once GATE-0 allows work. T013 must finish before any gated Phase 2 to 8 task that depends on a ruling.
- **Phase 2** depends on Phase 1 rulings and blocks Phases 3 to 8. Within it: T014 first, then T015, T016, T017 in parallel, then T018, T019, T020, then T021 to T025, then T026, with T027 to T031 once their targets exist.
- **Phase 3** (US1) depends on Phase 2. **Phase 4** (US1) depends on Phase 2, and T039 depends on T006.
- **Phase 5** (US2) depends on Phase 2, and T053 depends on T049.
- **Phase 6** (US3, US4) depends on Phases 2 and 5 (graph).
- **Phase 7** (US5) depends on Phases 2 to 6 for the data it serves. The pure security functions T070 to T075 depend only on Phase 2 and can start earlier.
- **Phase 8** depends on Phase 7 routes (T078, T079, T081, T082).
- **Phase 9** depends on all earlier phases. T104 must record signatures from unchanged code, so do it before any other edit if possible.
- **Phase 10** depends on Phase 8 for the real assets, and T118 needs the finished static files.

### User story dependencies

- **US1** (P1): Phases 2 to 4, then the server and page (Phases 7 and 8) to be demonstrable. Independent of US2 to US4.
- **US2** (P1): Phase 5. Independent of US1 at the data layer. The page integrates both.
- **US3** (P2): Phase 6 part one. Needs the catalog and the graph from Phase 5 for the effective-source rule (T062).
- **US4** (P3): Phase 6 part two. Needs Phase 5.
- **US5** (P1): Phase 7. Cross-cuts all stories.

### Within a phase

Tests accompany the code they cover. Write each test before or with the code and confirm the new tests fail first where practical (the suite style is `unittest.TestCase` under pytest, matching `tests/test_loaders.py`).

### Parallel opportunities

- **Phase 1**: T005, T006, T007, T011 together.
- **Phase 2**: T015, T016, T017 together, then T020, T027 together, and T028 to T031 together once the code exists.
- **Phase 3**: T032 and T033 together, then T035 to T037 together.
- **Phase 4**: T038, then T041 to T043 together.
- **Phase 5**: T045, T046, T047 together, T050 to T052 together, T056 to T058 together.
- **Phase 6**: T060 and the T068, T069 tests when their code exists.
- **Phase 7**: T071, T073, T074 together, then T086 to T091 together.
- **Phase 8**: T099, T100, T101 together.
- **Phase 10**: T114, T115, T116 together.

### Parallel example (Phase 5 scanner forms)

```text
Task: "Implement include detection in fhir_converter/ui/scanner.py"            (T045)
Task: "Implement evaluate ... using detection in fhir_converter/ui/scanner.py"  (T046)
Task: "Implement batch_render: detection in fhir_converter/ui/scanner.py"       (T047)
```
These touch the same file, so they are marked [P] only where the work is in separate functions. If the team prefers one author per file, run them in order instead.

---

## Implementation strategy

### MVP first (User Story 1)

1. Clear GATE-0 and run Phase 1 (baseline and investigations).
2. Complete Phase 2 (catalog).
3. Complete Phases 3 and 4 (listing, search, viewer).
4. Add the minimum of Phase 7 needed to serve it (T070 to T073, T076 to T082, T083) and the page parts T092 to T094, T099.
5. **Stop and validate**: run the US1 independent test from `quickstart.md`, and the Phase 9 regression checks that already apply (T104 to T107).
6. Review with the developers, then continue.

### Incremental delivery

1. Add US2 (Phase 5, T095), validate.
2. Add US3 and US4 (Phase 6, T096, T097), validate.
3. Complete US5 hardening (rest of Phase 7, T098, T100 to T102), validate.
4. Finish Phases 9 and 10.
5. Each increment must leave conversion output, the public API and the CLI unchanged.

### Rules while implementing

- Do not edit any existing module, template or test. Do not touch `docs/docstrings/`, `site/`, `htmlcov/` or `.venv`.
- Do not add a dependency or change a pin. Do not change `pyproject.toml` without GATE-18.
- Do not perform any gated task before its gate is cleared, and do not resolve an open decision by choosing silently.
- Use synthetic data only and never `data/sample`. Write conversion output to a temporary folder.
- Report real results and list anything not run.

---

## Notes

- Task IDs run T001 to T121.
- "Scenario Tnn" refers to the spec's 43 test scenarios and "FR/SEC/NFR/COMP/SC" to spec requirement and success criterion IDs.
- Names of error codes, field names and module layout are the plan's provisional choices and may change in review without changing the spec.
