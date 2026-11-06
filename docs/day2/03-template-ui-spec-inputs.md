# Spec Inputs: Template Management UI (First Release)

Status: **DRAFT INPUTS for `/speckit-specify`. Not a specification.** No `spec.md`, `plan.md` or `tasks.md` was created for this feature. No application code, template, test, dependency or configuration file was changed, and no command was run.

Audience: the developers who own `python-fhir-converter` (v0.3.0), who review this before `/speckit-specify` is run.

Date: 2026-11-05.

Sources: `docs/day2/01-template-ui-discovery.md`, `docs/day2/02-template-ui-decisions.md`, `CLAUDE.md`, `.specify/memory/constitution.md` (v1.0.0, an unratified draft). Facts keep the discovery tags (Verified, Carried, Assumption). This document is an engineering proposal. It makes no claim that output is FHIR-valid and no compliance claim, and it is not a substitute for developer review.

## Approval status

The 17 scope items you listed come from the proposed decisions in `02-template-ui-decisions.md`. **None has been formally approved by the developers.** Every item below is tagged:

| Tag | Meaning |
|---|---|
| **[APPROVAL]** | Needs explicit developer approval before it is final |
| **[PROPOSED]** | My proposal where the sources are silent. Needs a decision, not derived from the repository |
| (no tag) | Follows directly from the repository, `CLAUDE.md`, the constitution, or the discovery report |

A term used throughout: **logical name** means the template name as written in an include (`Section/Goal`), which maps to a file on disk with a `_` prefix on the last segment (`Section/_Goal.liquid`). Root templates and `.json` names have no prefix (`loaders.py`, Verified).

## A new finding that affects relationship requirements

While preparing this document I re-read three templates already seen during the Goals work. Template-to-template references are written in **three forms**, not one:

| Form | Example | Evidence |
|---|---|---|
| `include` tag | `{% include 'Section/Problem' -%}` in `CCD.liquid` | Verified |
| `batch_render` filter | `... \| batch_render: 'Entry/Problem/entry', 'entry'` in `Section/_Problem.liquid` and `Section/_Goal.liquid` | Verified |
| `evaluate` tag | `{% evaluate patientId using 'Utils/GenerateId' ... %}` in `CCD.liquid` | Verified |

The discovery report (B.3, K9) described include detection only. A scan for `include` alone would find root-to-section links but miss every section-to-entry link, and so would wrongly report most entry templates as unused. Which forms count as "detectable" is therefore an open decision (U-02) and shapes FR-040 to FR-042 and FR-060.

---

## 1. Scope of the first release

**In scope**
1. A read-only local browser UI. **[APPROVAL]**
2. A Python standard-library HTTP server. **[APPROVAL]**
3. UI code under `fhir_converter/ui/`, isolated from the existing modules. **[APPROVAL]**
4. Launch with `python -m fhir_converter.ui`. **[APPROVAL]**
5. Packaged C-CDA and STU3 templates.
6. An optional user template directory. **[APPROVAL]** (showing user template contents)
7. Listing by format and category, search by name, search within source, view of source and metadata.
8. Includes and used-by relationships where detectable, override identification, and best-effort highlighting of unreferenced C-CDA sections.

**Out of scope:** see section 7.

---

## 2. Functional requirements

Format for each requirement: description, acceptance criteria (AC), positive scenarios (P), negative scenarios (N). All tests use synthetic data only (constitution III).

Acceptance criteria that depend on the live package contents are written against a **fixture catalog** built in the test (a temporary folder of invented templates), not against the real 1,083 files, so they do not break when a template is added. A smaller set of checks against the real package confirms the mapping at scale.

### 2.1 Launch and configuration

**TUI-FR-001 Launch.** The UI starts with `python -m fhir_converter.ui` and serves a local page. **[APPROVAL]**
- AC1: running the command starts a server and prints (or otherwise makes known) the local address, within 10 seconds on a developer machine. **[PROPOSED]** (the 10 seconds is mine)
- AC2: no change to `pyproject.toml` scripts, `fhir_converter/__main__.py` or any existing module is needed to start it.
- AC3: importing `fhir_converter` does not import `fhir_converter.ui` (checked by inspecting loaded modules after `import fhir_converter`).
- P: start with no arguments, packaged templates only. Start with a user directory.
- N: `python -m fhir_converter` still behaves exactly as before (see TUI-FR-091).

**TUI-FR-002 Loopback only.** The server accepts connections on the loopback interface only, never on all interfaces. No authentication exists in the project, so network exposure is not acceptable by default (discovery B.7).
- AC1: the listening address is `127.0.0.1` (and optionally `::1`, **[PROPOSED]**), verified by a test that inspects the bound address.
- AC2: there is no option that binds to a non-loopback address in the first release. **[APPROVAL]** (confirm no such option)
- P: browser on the same machine connects.
- N: a connection attempt from a non-loopback address is not accepted (verified where the test environment allows, otherwise by inspecting the bound address).

**TUI-FR-003 Optional user template directory.** The user can supply one directory of custom templates at startup, with the same meaning as `--template-dir` (the user directory is searched before packaged templates). **[APPROVAL]**
- AC1: with a valid directory, user templates appear in the catalog with origin "user" (TUI-FR-012).
- AC2: without the option, the catalog contains only packaged templates and no user-directory control implies otherwise.
- AC3: the name of the startup option is not fixed by discovery (see U-06).
- P: directory with two valid `.liquid` files is listed.
- N: option given with a missing path, a file instead of a directory, or an unreadable directory: see TUI-FR-004.

**TUI-FR-004 Startup validation.** A bad configuration produces a clear startup error and no partial server (discovery B.6).
- AC1: for a missing, non-directory or unreadable user directory, the process exits with a non-zero code before serving, and prints a fixed message that does not include exception text.
- AC2: no listening socket remains open after that failure.
- P: valid directory starts normally.
- N: each of the three bad-directory cases above.

### 2.2 Catalog: listing

**TUI-FR-010 List templates by format.** The UI lists templates for a chosen format, C-CDA or STU3. The two formats are separate lists.
- AC1: for a fixture package with N C-CDA and M STU3 `.liquid` templates, the C-CDA list has N entries and the STU3 list has M.
- AC2: against the real package, the count for each format equals the number of `.liquid` files under `templates/ccda` and `templates/stu3` found on disk at test time (819 and 264 when discovery was written, Verified).
- AC3: non-template files (`metadata.json`, `ValueSet.json`, `cda_narrative.xsl`) are not counted as templates. Whether `ValueSet.json` should be browsable is U-08.
- P: switch from C-CDA to STU3 and see a different list.
- N: an unknown format value is rejected with the generic error (TUI-FR-070).

**TUI-FR-011 Category.** Each template is shown with a category derived from its location, and the list can be filtered by category.
- AC1: C-CDA categories cover at least: root (top-level files), `Section`, `Entry`, `Resource`, `Reference`, `DataType`, `Extension`, `Utils`, `ValueSet`, and "other" for anything unmatched. The discovery list (B.3) omitted `Reference`, `Extension` and `Utils`, which exist on disk (Verified), so the final taxonomy is U-07. **[APPROVAL]**
- AC2: every listed template has exactly one category, and the per-category counts sum to the format total.
- AC3: filtering by a category returns only templates of that category.
- P: filter `Section` returns only section templates.
- N: an unknown category is rejected with the generic error, or returns an empty list, whichever the spec chooses (U-07).
- Note: how STU3 templates are categorized is not established. The discovery report records one template per resource type at the top level plus sub-folders for nested pieces (Carried). The STU3 taxonomy is U-07.

**TUI-FR-012 Origin.** Each template shows whether it comes from the package or from the user directory.
- AC1: every entry has an origin value of "package" or "user".
- AC2: no template is shown with both origins unless it is the override case (TUI-FR-050).
- P: fixture user template shows "user". Packaged shows "package".
- N: with no user directory configured, no entry shows "user".

**TUI-FR-013 Logical names.** Templates are shown and addressed by logical name, mapped from the file on disk using the same rule as the loader.
- AC1: `Section/_Goal.liquid` is shown as `Section/Goal`. A root file such as `CCD.liquid` is shown as `CCD`. A `.json` name is shown without a prefix.
- AC2: for every template in the fixture catalog, the logical name resolves to that file through the loader's public behavior (a test compares the catalog's mapping with `env.get_template` or `read_text`). The loader's private `_resolve_template_name` is not imported. **[APPROVAL]** (confirm re-implementing the rule rather than exposing it, since exposing it changes the public surface)
- AC3: list order is deterministic (sorted by logical name) across runs.
- P: nested path `Entry/Problem/entry_act_entryRelationship`.
- N: a file in a sub-folder without the `_` prefix is handled consistently with the loader (U-09).

### 2.3 Search

**TUI-FR-020 Search by name.** The user can search templates by name.
- AC1: a search returns every template whose logical name matches the query, and none that do not.
- AC2: matching rule is case-insensitive substring on the logical name. **[PROPOSED]** (the rule is not in the discovery)
- AC3: an empty query returns the full list for the selected format. **[PROPOSED]**
- AC4: results can be combined with the format and category filters.
- P: query `goal` finds `Section/Goal`, `Entry/Goal/entry`, `Resource/Goal`, `Reference/Goal/Subject` in a fixture that has them.
- N: a query with no matches shows an empty result and a plain "no results" message, not an error. A query of excessive length is rejected with the generic error (limit in U-10).

**TUI-FR-021 Search within source.** The user can search inside template source text.
- AC1: a query returns the templates whose source contains the text, and none that do not.
- AC2: each result identifies the template and, where practical, shows where the text occurs. **[PROPOSED]**
- AC3: results are capped at a stated maximum and the UI says when results were cut off (limit in U-10).
- AC4: only `.liquid` and `.json` files are searched (discovery B.5), and files above the size cap are skipped with a visible note (cap in U-10).
- AC5: search never alters any file or the template cache.
- P: query for a known `get_ccda_section_by_template_id` string in the fixture.
- N: a file that cannot be read is skipped without failing the search and without exposing exception text. A binary or non-UTF-8 file does not crash the search.

### 2.4 View

**TUI-FR-030 View source.** The user can open a template and read its source, read-only.
- AC1: the displayed text equals the file contents byte for byte (decoded as UTF-8).
- AC2: the source is shown as inert text. Any Liquid or HTML in it is not interpreted or executed by the page (a test with a source containing `<script>` and `{% ... %}` verifies nothing is executed or rendered as markup).
- AC3: no control on the page changes the file.
- P: view a packaged template and a user template.
- N: viewing a name that is not in the catalog returns the generic error. A file over the size cap is shown as "too large to display" with its size.

**TUI-FR-031 View metadata.** The template view shows its metadata.
- AC1: shows at least: format, logical name, category, origin, file size, and the on-disk file name relative to its root (not an absolute host path, see TUI-SEC-008).
- AC2: for a C-CDA section template, shows the template id (OID) it looks up when one can be read from the source (discovery F8). That detection is best-effort and labelled (TUI-FR-042). **[APPROVAL]**
- P: `Section/Goal` shows `2.16.840.1.113883.10.20.22.2.60` in the real package (Verified in the Goals work).
- N: a section template with no readable id shows "not detected", not an empty or invented value.

### 2.5 Relationships

**TUI-FR-040 Includes.** For a template, the UI lists the other templates it references, where detectable.
- AC1: for the fixture catalog, the list equals the set of template names referenced by the template in the forms the spec adopts (U-02).
- AC2: each referenced name is shown with whether it exists in the catalog. A reference to a name that does not exist is shown as "not found", not hidden.
- AC3: at minimum, literal quoted names in `include` tags are detected (the form discovery assumed). `batch_render` and `evaluate` forms are required if U-02 adopts them. **[APPROVAL]**
- P: `CCD` lists `Section/Problem`, `Section/Medication` and the rest of its includes in the real package.
- N: a reference built from a variable (a dynamic name) is not guessed. It is reported as "dynamic reference, not detected" (TUI-FR-042).

**TUI-FR-041 Used-by.** For a template, the UI lists the templates that reference it, where detectable. For C-CDA it also shows the root templates that reach it.
- AC1: for the fixture catalog, used-by equals the inverse of the includes relation (A lists B as included implies B lists A as used-by).
- AC2: "root templates that reach it" is computed over the references that are detectable, and states whether it follows references transitively. Transitive or direct is U-03. **[APPROVAL]**
- AC3: a template with no detected references shows an explicit "none detected".
- P: `Section/Problem` is used by every root template that includes it.
- N: a cycle in references does not hang the UI. Computation ends and each template appears once.

**TUI-FR-042 Detectability labelling.** Every relationship view states that it is derived from a text scan and is best-effort, and lists what was not detected.
- AC1: the page shows a fixed note that relationships come from scanning template text for specific forms and that other forms are not detected.
- AC2: a dynamic or unrecognized reference is shown as "not detected" for that template.
- AC3: the UI never presents an absence of a relationship as proof that none exists.
- P: a template with a variable-name include shows the "not detected" notice.
- N: a fixture where a template is only referenced in a form the scan does not support is not asserted to be unreferenced without the best-effort label (see TUI-FR-060).

### 2.6 Overrides

**TUI-FR-050 Identify overrides.** The UI identifies packaged templates that a user template overrides.
- AC1: when a user template has the same logical name as a packaged template of the selected format, both entries are shown and linked, the user one marked "overrides packaged", the packaged one marked "overridden by user template". **[APPROVAL]**
- AC2: both sources can be viewed.
- AC3: a user template with no packaged counterpart is shown as "user only", not as an override.
- AC4: a filter shows only overrides, and the count of overrides is displayed.
- AC5: matching is by logical name using the same mapping as TUI-FR-013.
- P: fixture user directory containing `CCD.liquid` overrides packaged `CCD`.
- N: a user file with a similar but different name is not flagged. A user file in a different format selection is not compared against the other format.
- Note: `--template-dir` carries no format marker (discovery, decisions Q6). The proposed handling is that the user selects the format whose templates are compared. **[APPROVAL]** U-05.

### 2.7 Unreferenced sections

**TUI-FR-060 Highlight potentially unreferenced C-CDA sections.** The UI highlights C-CDA `Section` templates that no root template references, using best-effort analysis.
- AC1: in the fixture catalog, a section template referenced by no root template is highlighted. A section referenced by at least one root template is not.
- AC2: the wording says "potentially unreferenced" and states the analysis limits (TUI-FR-042). It does not say "unused" or "dead".
- AC3: root templates are the C-CDA top-level templates (including `Header`, which is include-only, Carried). The exact root list is determined from the catalog, not hard-coded. **[APPROVAL]**
- AC4: the count of highlighted sections is shown and the list can be filtered to them.
- AC5: no such highlight is shown for STU3 (no root and section structure, Carried).
- AC6: the result depends only on template contents, so it changes if a root template changes. In the real package it does not assume Goals, Health Concerns or Instructions are flagged. At discovery time those three were (Verified), and `001-wire-goals-section` may change that.
- P: fixture with an unwired `Section/Orphan`.
- N: a section referenced only through a form the scan does not detect is not claimed to be unreferenced without the best-effort label. A section referenced only by another section (not a root) is handled per U-03 and clearly described.

### 2.8 Errors

**TUI-FR-070 Generic errors.** Every error shown to the user uses a short fixed message and an error code, never exception text. **[APPROVAL]**
- AC1: for each failure case (unknown format, unknown category, unknown template name, invalid or traversal-style name, unreadable file, file too large, query too long, user directory problem), the response contains only the fixed message and code.
- AC2: a test that forces an internal exception with a distinctive marker string confirms the marker never appears in the page, response body or headers.
- AC3: the response never contains a stack trace or an absolute host path.
- AC4: unknown routes and unsupported methods get a generic message too.
- P: request for a non-existent template returns the fixed message.
- N: injected file-read failure and injected parse failure both stay generic.

### 2.9 Read-only and exclusions

**TUI-FR-080 Read-only.** The UI offers no way to create, edit, upload, rename or delete a template, and performs no writes.
- AC1: no page control and no server route accepts content for storage. Only read requests (GET and HEAD) are accepted. **[PROPOSED]** (the method list is mine)
- AC2: a snapshot (file names, sizes, hashes) of the packaged template tree and the user directory is identical before and after a test session that exercises every feature.
- AC3: requests with POST, PUT, PATCH or DELETE receive the generic error and change nothing.
- P: full browse and search session leaves the trees unchanged.
- N: each non-read method rejected.

**TUI-FR-081 No rendering.** The UI does not render a template, run a conversion, or load a Liquid environment for rendering.
- AC1: no route accepts input data or calls a renderer. A test confirms the UI package does not call the render functions during a session (for example by checking the renderer entry points are not invoked).
- AC2: no converted output is shown or stored.
- P: any browsing session.
- N: a request that tries to pass a document or ask for a render is rejected with the generic error.
- Note: whether a "does the template load" check is included is undecided (U-04) and is not part of this requirement.

**TUI-FR-082 No patient-data input.** The UI has no file upload, paste area or input for clinical documents.
- AC1: no form field or route accepts document content.
- AC2: the only user inputs are format, category, a name query, a source query, a template name chosen from the catalog, and the startup directory option.
- P and N: covered by TUI-FR-080 and TUI-FR-081 scenarios.

### 2.10 Constraints expressed as requirements

**TUI-FR-090 No new dependencies.**
- AC1: `pyproject.toml` dependency sections are unchanged. No import in `fhir_converter/ui/` comes from outside the Python standard library and the existing declared runtime dependencies.
- AC2: the served pages load no resource from another host (no CDN, font service or analytics). A test scans the served assets for external URLs, and the page renders with outbound network blocked. **[PROPOSED]**
- AC3: no frontend framework, bundler or package manager step is used.

**TUI-FR-091 No change to existing behavior.** **[APPROVAL]**
- AC1: converted output for the existing synthetic fixtures (`TD-CCDA-01` to `12`, excluding `08` which fails by design) and the two `tests/data/ccda` fixtures is identical before and after the feature is added (JSON diff of zero differences).
- AC2: the public API (`render_fhir_string`, `render_fhir`, `render_to_fhir`, `make_environment`, `render_to_dir`, `render_files_to_dir`) keeps its names, signatures and behavior. A test or review compares signatures.
- AC3: the existing CLI (`python -m fhir_converter`) keeps its arguments, output and exit codes. The CLI cases TC-CLI-05 and TC-CLI-11, and the other TC-CLI cases, give the same results as before. (`__main__.py` has no unit tests, so this is a manual check against `docs/test-cases.md`.)
- AC4: the pre-existing pytest suite passes with the same count as the recorded baseline, plus the new tests.
- AC5: no existing template, `ValueSet.json`, id-generation input or date rule is changed.

**TUI-FR-092 Isolation of new code.**
- AC1: all new application code lives under `fhir_converter/ui/`, and no existing module in `fhir_converter/` is edited to support it. **[APPROVAL]** (if packaging needs a `pyproject.toml` or ignore-file change, that is a configuration change needing its own approval, U-01)
- AC2: filesystem access for the UI is confined to one module, so it can be tested without a running server (discovery B.3).

---

## 3. Non-functional requirements

| ID | Requirement | Acceptance criterion |
|---|---|---|
| TUI-NFR-001 | **Performance (proposed targets).** **[APPROVAL]** | Against the real package, the first catalog list for a format is returned in under 2 seconds, and opening one template in under 1 second, on a normal developer machine. These numbers are mine and have no repository basis. Measure and record the results. |
| TUI-NFR-002 | **Responsiveness under search.** | A source search over the full real package ends in a bounded time (limit in U-10) and does not block other requests indefinitely. |
| TUI-NFR-003 | **Accessibility (design aim, not a conformance claim).** **[APPROVAL]** | Every function is operable by keyboard. Focus is visible. Pages use semantic headings and labelled controls. Text meets a stated contrast ratio chosen in the spec. No claim of conformance to any standard is made. |
| TUI-NFR-004 | **Browser support.** **[APPROVAL]** | Works in current versions of Chrome, Edge, Firefox and Safari. Legacy browsers are unsupported. |
| TUI-NFR-005 | **Determinism.** | Listing and search results are stable and sorted for the same input. |
| TUI-NFR-006 | **Staleness note.** | The page states that the catalog reflects templates at server start and that a restart is needed to see changes, consistent with the template cache having `auto_reload` off (Carried). Whether the catalog refreshes without restart is U-11. |
| TUI-NFR-007 | **Code standards.** | New code is fully type-annotated and passes mypy with the project settings. flake8 passes with max line length 120, ignore `E203,E721`, max complexity 10. Docstrings use `Args:`, `Returns:`, `Raises:`. No `logging`, and no new output channel without agreement (`CLAUDE.md` sections 4 and 5). |
| TUI-NFR-008 | **Testability.** | The catalog logic is unit tested without starting a server, and the server layer is tested separately. New tests follow the style of `tests/test_loaders.py`. |
| TUI-NFR-009 | **Documentation.** | README or docs page for the UI, a `CHANGES.md` entry (new feature), and `docs/test-cases.md` statuses set only after cases have been run. |
| TUI-NFR-010 | **Resource limits.** | Maximum served file size, maximum search result count, maximum query length and a request timeout are defined and enforced (values in U-10). |
| TUI-NFR-011 | **Portability.** | Works on Windows, Linux and macOS with consistent path handling (CI runs all three). |
| TUI-NFR-012 | **Honest wording.** | UI text and docs make no claim that templates or output are FHIR-valid or conformant, and no HIPAA, SOC 2, HITRUST or ISO claim (`CLAUDE.md` section 8). |

---

## 4. Security and privacy acceptance criteria

| ID | Criterion | Test approach |
|---|---|---|
| TUI-SEC-001 | Server binds to loopback only (TUI-FR-002). | Inspect bound address |
| TUI-SEC-002 | **Cross-site protection.** Requests with an unexpected `Host` or `Origin` header are rejected, or require a per-run token. Which mechanism is used is not decided (U-12). **[APPROVAL]** | Send requests with a foreign `Host`, a foreign `Origin`, and no token (if tokens are chosen). Expect rejection |
| TUI-SEC-003 | **No path built from request input.** Template access goes through an allowlist built from catalog enumeration. A request value never becomes a file path. | Confirm by code review and by traversal tests below |
| TUI-SEC-004 | **Traversal rejected.** Names containing `..`, absolute paths, drive letters, backslashes or NUL are rejected with the generic error (discovery B.5). | Table of malicious names, including URL-encoded and double-encoded forms **[PROPOSED]** for the encodings, expect rejection and no file read |
| TUI-SEC-005 | **User directory containment.** The user directory is resolved once at startup and must exist and be a directory. A symlink that leaves it is refused. | Fixture with a symlink to an outside file. Skipped where the platform cannot create symlinks, and the skip is reported |
| TUI-SEC-006 | **Extension allowlist and size cap.** Only `.liquid` and `.json` are readable. Larger files are not returned (cap in U-10). | Request a `.txt` file, an oversized fixture file |
| TUI-SEC-007 | **No exception text in responses** (TUI-FR-070). | Marker-string injection test |
| TUI-SEC-008 | **No absolute host paths in responses.** Paths shown are relative to the template root. **[PROPOSED]** | Search all responses in a session for the fixture's absolute temp path |
| TUI-SEC-009 | **Read-only enforced** (TUI-FR-080). | Before and after tree hash, rejected methods |
| TUI-SEC-010 | **Templates are code, never executed.** Template source is displayed as text and never rendered, imported or evaluated by the UI. | Source containing script tags and Liquid tags is displayed inert (TUI-FR-030 AC2), and no renderer is called (TUI-FR-081) |
| TUI-SEC-011 | **Output encoding.** All template-derived text placed in a page is escaped so a template cannot inject markup or script into the UI. | Fixture template with HTML, script and attribute-breaking characters in its name and body |
| TUI-SEC-012 | **No patient data handled.** The UI accepts no documents and shows no converted output (TUI-FR-081, TUI-FR-082). All test data is synthetic, and nothing from `data/sample` is used (constitution III). | Review of fixtures and routes |
| TUI-SEC-013 | **No persistence of user activity.** The UI writes no files, caches to disk or history. Search queries are not stored. | Tree snapshot before and after, review |
| TUI-SEC-014 | **Request logging.** The standard-library server logs each request to the console by default. Whether that is acceptable, or must be silenced because `CLAUDE.md` bars new output channels, is undecided (U-13). **[APPROVAL]** | Observe output during a session |
| TUI-SEC-015 | **No secrets, identifiers or addresses** appear in fixtures, docs or code (`CLAUDE.md` section 8). | Review and search |
| TUI-SEC-016 | **No compliance claims.** Docs and UI text avoid compliance statements. This document and the feature are engineering descriptions, not an attestation of any control. | Text review |
| TUI-SEC-017 | **Pre-existing exception-content issue is not worsened.** The UI does not call code paths known to place partial output in exception text. That issue is tracked separately (decisions Q10). **[APPROVAL]** to open the separate item | Review |

---

## 5. Compatibility requirements (Python 3.8 to 3.12)

| ID | Requirement | Acceptance criterion |
|---|---|---|
| TUI-COMP-001 | Everything under `fhir_converter/ui/`, including its `__main__`, is compatible with Python 3.8 through 3.12. The existing CLI module uses 3.11 syntax (`StrEnum`) and is not a model for this (`CLAUDE.md` section 4). | The pytest, flake8 and mypy jobs pass on the full CI matrix |
| TUI-COMP-002 | No syntax or library features newer than 3.8 are used in `fhir_converter/ui/`. Examples to avoid: built-in generic types such as `list[str]` in annotations evaluated at runtime, the `X \| Y` union syntax, `match` statements, `str.removeprefix` and `removesuffix`, `functools.cache`, parenthesized context managers. Use `typing` equivalents. | A 3.8 run of the full test suite plus the import of every UI module |
| TUI-COMP-003 | Only the standard library and already declared runtime dependencies are imported. | Import review and a test that imports each UI module in a clean environment |
| TUI-COMP-004 | The standard-library server module is used in a way that works on all five Python versions. The exact module and class are an implementation choice for the plan, not decided here. | CI matrix run, and a startup-and-request test on each version |
| TUI-COMP-005 | Behavior is the same on Windows, Linux and macOS, including path separators and case sensitivity. Logical names always use forward slashes. | CI on all three. A test with Windows-style separators in requested names |
| TUI-COMP-006 | `python -m fhir_converter.ui` works from an installed package and from a source checkout. That static assets are included in the built package is not verified (U-01). | A build-and-install check once approved, which is a configuration-affecting step |
| TUI-COMP-007 | Adding the UI does not change import time or behavior of `import fhir_converter` for library users (TUI-FR-001 AC3). | Module inspection test |

---

## 6. Proposed test scenarios

All data synthetic. "Fixture catalog" means a temporary tree of invented `.liquid` files built inside the test (for example a small C-CDA-like tree with a `CCD`, `Section/Alpha`, `Section/Orphan`, `Entry/Alpha/entry`, `Resource/Alpha`) plus a small user directory built the same way. Existing `data/synthetic/templates/Mini.liquid` can serve as one user template.

| # | Scenario | Covers |
|---|---|---|
| T01 | Start with packaged templates only, request the C-CDA list and the STU3 list, compare counts with files on disk | FR-001, FR-010 |
| T02 | Category filter returns only that category, and category counts sum to the total | FR-011 |
| T03 | Every template has an origin. With no user directory, none is "user" | FR-012 |
| T04 | Logical-name mapping for root, nested and `.json` names matches loader behavior, without using the private method | FR-013 |
| T05 | Name search: match, no match, empty query, case difference, overlong query | FR-020 |
| T06 | Source search: match, no match, result cap reached and reported, unreadable file skipped, non-UTF-8 file skipped | FR-021 |
| T07 | View source equals file bytes. A template containing script and Liquid tags is shown inert and escaped | FR-030, SEC-010, SEC-011 |
| T08 | Metadata fields present. Section OID shown when detectable, "not detected" otherwise | FR-031 |
| T09 | Includes and used-by agree (inverse relation). Missing target shown as "not found". Dynamic reference labelled "not detected". Cyclic references terminate | FR-040, FR-041, FR-042 |
| T10 | A relationship written with `batch_render` and one with `evaluate` are detected or reported as not detected, as the spec decides in U-02 | FR-040 |
| T11 | User directory overrides a packaged template. Both sources viewable. Override count and filter work. A user-only template is not an override | FR-050 |
| T12 | Fixture `Section/Orphan` is highlighted. A section referenced by a root is not. STU3 shows no such highlight. Wording says "potentially" | FR-060 |
| T13 | Each error case returns the fixed message and code. A marker string in an injected exception never appears | FR-070, SEC-007 |
| T14 | Non-read methods and unknown routes are rejected. Tree snapshot unchanged after a full session | FR-080, SEC-009 |
| T15 | No route accepts input data. Renderer entry points are not called. No upload control exists | FR-081, FR-082 |
| T16 | Traversal names (plain, encoded, backslash, drive letter, NUL, absolute) rejected, no file read | SEC-003, SEC-004 |
| T17 | Symlink escaping the user directory refused (where supported) | SEC-005 |
| T18 | Foreign `Host` or `Origin` rejected | SEC-002 |
| T19 | Bound address is loopback. No option exists to change it | FR-002, SEC-001 |
| T20 | Bad user directory (missing, file, unreadable) fails at startup with a non-zero exit and fixed message, with no listener left open | FR-004 |
| T21 | Served assets contain no external URLs. Page works with outbound network blocked | FR-090 |
| T22 | Conversion output before and after is identical for the named fixtures. Public API signatures unchanged. Existing suite count unchanged | FR-091 |
| T23 | `import fhir_converter` does not load the UI package | FR-001, COMP-007 |
| T24 | Full CI matrix (3.8 to 3.12, three platforms) runs pytest, flake8 and mypy on the new code | COMP-001 to COMP-005 |
| T25 | Keyboard-only walk through list, search, view, relationships and overrides | NFR-003 |
| T26 | Timing of list and view against the real package, recorded | NFR-001 |
| T27 | Real-package spot checks: `Section/Goal` maps to `Section/_Goal.liquid` and shows its OID. `CCD` shows its section references. Section-to-entry links appear if `batch_render` is adopted in U-02 | FR-013, FR-031, FR-040 |
| T28 | Manual: existing CLI cases (`TC-CLI-05`, `TC-CLI-11` and the rest) rerun and compared with recorded results | FR-091 AC3 |

Not planned: any test that renders a document through the UI, because rendering is out of scope. No FHIR validator is run, so no test asserts FHIR validity.

---

## 7. Out of scope (first release)

- Creating, editing, uploading, duplicating, renaming or deleting templates, packaged or user.
- Rendering a template, running a conversion, previewing output, or accepting any clinical document.
- Authentication, user accounts, multi-user access, remote or network access.
- Persistence, history, favourites or saved searches.
- Editing or managing `ValueSet.json` code mappings. Viewing them is undecided (U-08).
- A frontend framework, bundler, web framework, ASGI server or any new dependency.
- A new console script, a change to `pyproject.toml` scripts, or any change to the existing CLI.
- Fixing the unwired Goals, Health Concerns or Instructions sections (separate work, `001-wire-goals-section` awaits approval).
- Fixing the pre-existing exception-content behavior (separate item).
- Validating templates or output against FHIR, or any conformance statement.
- Compiling or syntax-checking templates, unless U-04 is decided in favour and verified.
- Hot reload of templates while the server runs, unless U-11 decides otherwise.
- Exposing or changing `TemplateSystemLoader._resolve_template_name`, or any public API change.

---

## 8. Assumptions

| # | Assumption | Basis |
|---|---|---|
| A1 | The developers will approve the read-only scope. Until they do, nothing here is final. | `02-template-ui-decisions.md` is a proposal |
| A2 | Users are developers and integrators, so technical names and raw Liquid source are acceptable to show. | Decisions Q2, not user-researched |
| A3 | All references of interest are literal quoted names. A scan can find them with plain text matching, and everything else is reported as not detected. | Discovery K9 (an Assumption), widened by the three-forms finding above |
| A4 | The catalog is built at server start and held in memory, consistent with the template cache having `auto_reload` off. | Discovery D.3 (Carried). Choice itself is not in discovery |
| A5 | A user directory applies to whichever format is selected, because `--template-dir` records no format. | Decisions Q6 |
| A6 | Packaged templates read by the UI are the same files the loader would serve. | Not confirmed for installed (non-source) packages |
| A7 | The standard-library server is adequate for one local user with a few concurrent requests. | Not tested |
| A8 | Template source is not patient data, so showing it is a lower privacy risk than showing output. | Discovery D.4 |
| A9 | The package build includes static files placed under `fhir_converter/ui/`. | Not verified. The shipped `.liquid` files suggest it works |
| A10 | Python 3.8 compatibility rules apply to `fhir_converter/ui/` because it is inside the package. | `CLAUDE.md` section 4, decisions Q4 |
| A11 | Counts of 819 and 264 `.liquid` files were correct when discovery ran and may change. Tests must not hard-code them. | Discovery (Verified at the time) |
| A12 | The proposed performance, accessibility and browser targets are placeholders. | Decisions Q11 |

---

## 9. Unresolved decisions

| ID | Decision needed | Why it matters | Needs approval |
|---|---|---|---|
| U-01 | Packaging: does the build include static assets under `fhir_converter/ui/`, and is any `pyproject.toml` or ignore-file change needed? | A configuration change needs approval and affects what ships | Yes |
| U-02 | Which reference forms count as detectable: `include` only, or also `batch_render` and `evaluate`? | Without `batch_render`, section-to-entry links and many used-by results are missing, and unreferenced-section logic is limited to root-to-section links | Yes |
| U-03 | Is "root templates that reach a template" direct only or transitive? And is a section referenced only by another section "referenced"? | Changes FR-041 and FR-060 results | Yes |
| U-04 | Include an optional "does this template compile" check in release 1? Needs verification that python-liquid parses on load and what that proves. | Could imply more than it proves | Yes |
| U-05 | How is a user directory tied to a format (user selects the format at comparison time, or a startup option, or sub-folders)? | `--template-dir` has no format marker | Yes |
| U-06 | The name and form of the startup option for the user directory, and for the port and address | Interface naming is not in discovery | Yes |
| U-07 | Final category taxonomy for C-CDA (including `Reference`, `Extension`, `Utils`) and for STU3, and behavior for an unknown category | The discovery list was incomplete and STU3 structure was not re-read | Yes |
| U-08 | Is `ValueSet.json` (and `metadata.json`, `cda_narrative.xsl`) browsable as read-only files? | Affects what counts as a "template" in counts | Yes |
| U-09 | Treatment of a user-directory file in a sub-folder that lacks the `_` prefix | The loader rewrites the last segment, so such a file would not resolve by its logical name | Yes |
| U-10 | Concrete limits: maximum file size shown, maximum search results, maximum query length, search time bound, request timeout | Needed for measurable criteria | Yes |
| U-11 | Does the catalog refresh while the server runs, or only at startup? | Interacts with the template cache and the staleness note | Yes |
| U-12 | Cross-site protection mechanism: header checks only, a per-run token, or both | Discovery lists both as options | Yes |
| U-13 | Is default per-request console logging by the standard-library server acceptable, or must it be silenced? | `CLAUDE.md` bars new output channels without agreement | Yes |
| U-14 | Port selection: fixed default, user-chosen, or automatic free port, and how the address is shown | Affects startup behavior and tests | Yes |
| U-15 | Should the page itself be a single HTML page with plain JavaScript, or several pages? Plain JavaScript was proposed, and no framework | Confirms the one file type the repo does not have today | Yes |
| U-16 | Ratification of the constitution (v1.0.0 draft) before the spec cites it as final | Cited as a draft until then | Yes |
| U-17 | Baseline run of pytest (and approval to install flake8 and mypy if absent) | Required before and after any implementation; no baseline exists for this feature | Yes |
| U-18 | Interaction with `001-wire-goals-section`: should the UI work, or its tests, assume any particular wired state? Proposed answer is no | Avoids coupling two features that await separate approval | Yes |

---

## 10. Traceability to the 17 scope items

| Scope item | Requirements |
|---|---|
| 1 Read-only local browser UI | FR-080, FR-081, FR-082, FR-002, TUI-SEC-009 |
| 2 Standard-library HTTP server | FR-090, TUI-COMP-004 |
| 3 Code under `fhir_converter/ui/` | FR-092, TUI-COMP-001 |
| 4 Launch with `python -m fhir_converter.ui` | FR-001 |
| 5 Packaged C-CDA and STU3 | FR-010, FR-011 |
| 6 Optional user directory | FR-003, FR-004, FR-012 |
| 7 List by format and category | FR-010, FR-011 |
| 8 Search by name | FR-020 |
| 9 Search within source | FR-021 |
| 10 View source and metadata | FR-030, FR-031 |
| 11 Includes and used-by | FR-040, FR-041, FR-042 |
| 12 Overrides | FR-050 |
| 13 Unreferenced C-CDA sections | FR-060 |
| 14 Generic errors | FR-070, SEC-007 |
| 15 No editing, deletion, rendering or preview | FR-080, FR-081, FR-082 |
| 16 No new dependencies | FR-090, COMP-003 |
| 17 No change to conversion, public API or CLI | FR-091, FR-092, COMP-007 |

## Next step

Wait for developer review of this document, especially section 9. After the decisions are confirmed or changed, `/speckit-specify` can be run with these inputs. Nothing further is created until then.
