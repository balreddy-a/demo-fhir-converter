# Implementation Plan: Template Management UI

**Branch**: none (spec directory `specs/002-template-management-ui`) | **Date**: 2026-11-05 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/002-template-management-ui/spec.md`

**Status**: PLAN ONLY. Nothing is implemented. No application code, template, test or configuration file was changed, and no test, build or installation command was run. Every decision still tagged **[APPROVAL]** or **TBD** in the spec stays open here. Where this plan has to assume something to make a design concrete, it says "provisional" and lists it in section 11 for a developer ruling. Approval-gated actions are listed in section 10 and are not executed.

**Sources read**: the spec, `.specify/memory/constitution.md` (v1.0.0, an **unratified draft**), `CLAUDE.md`, `docs/day2/00` to `04`, and the repository files cited below. The constitution is cited as draft guidance, not as a ratified document.

**Not claimed**: nothing in this plan says any template or converter output is FHIR-valid or conformant, and nothing makes a HIPAA, SOC 2, HITRUST or ISO claim. This is an engineering plan, not a substitute for developer review.

## Summary

Add a read-only, local-only browser interface for exploring the packaged C-CDA and STU3 Liquid templates and an optional user template directory. It lists, searches, shows source and metadata, shows direct and transitive template relationships (three literal reference forms), flags user overrides, and highlights C-CDA sections that no root template appears to reach, all labelled best-effort.

Technical approach, in one paragraph: a new package `fhir_converter/ui/` holds everything. A **catalog** reads `.liquid` files straight from disk once at startup. A **scanner** extracts literal references from template text with a small set of regular expressions, never parsing or compiling Liquid. A **graph** module builds the per-format effective graph and answers direct, transitive, root-reach and unreferenced-section questions. A **security** module holds Host, Origin, token and name checks. A **server** module exposes a small read-only JSON API and three static files through `http.server`, with per-request logging switched off. The page is plain HTML, CSS and JavaScript. No existing module is edited and no dependency is added.

## Technical Context

| Item | Value |
|---|---|
| **Language/Version** | Python, library modules compatible with 3.8 to 3.12 (`pyproject.toml` `^3.8.1`, CI matrix 3.8 to 3.12, `CLAUDE.md` section 4). Plain HTML, CSS and JavaScript for the page |
| **Primary Dependencies** | None new. Standard library only (`http.server`, `socketserver`, `pathlib`, `re`, `json`, `secrets`, `dataclasses`, `argparse`, `typing`). The existing runtime dependencies are not needed by the UI |
| **Storage** | None. Read-only access to template files. No files written, no cache on disk, no history |
| **Testing** | pytest 7.4.4 (existing). Standard-library `http.client` and `threading` for server tests. `subprocess` for startup, console and import-isolation tests |
| **Target Platform** | Developer workstation, Windows, Linux, macOS (CI runs all three). Browser: current Chrome, Edge, Firefox, Safari (NFR-004, **[APPROVAL]**) |
| **Project Type** | Library plus CLI, gaining an optional local web tool inside the package |
| **Performance Goals** | Proposed only: list under 2 s, open under 1 s, startup build under 5 s (NFR-001, NFR-013, **[APPROVAL]**, D-24 open) |
| **Constraints** | No change to conversion output, public API, CLI, templates, ids or the date rule. Loopback only. No path built from request input. No `logging`, no output beyond the startup message and fatal startup errors (FR-007, **[APPROVAL]**) |
| **Scale/Scope** | About 1,083 `.liquid` files (819 C-CDA, 264 STU3 at discovery, not to be hard-coded). References found: 619 `include`, 188 `batch_render`, 24 `evaluate` literals in the packaged files (Verified, narrow patterns) |

**Local environment facts (Verified)**: `.venv` runs Python **3.14.10**, outside the supported 3.8 to 3.12 range (`CLAUDE.md` section 7 calls this a known deviation). The `python` on PATH is a Microsoft Store shortcut and does not run. So local runs cannot prove 3.8 compatibility. CI is the authority, and investigation I-3 covers this.

### Unknowns carried from the spec (none silently resolved)

| Unknown | Plan treatment |
|---|---|
| D-08 STU3 category taxonomy | Provisional generic rule (first folder name, or "root"), see 6.2 |
| D-09 user file in a sub-folder without `_` prefix | Provisional: not mapped, counted, not in the graph, see 6.2 |
| D-12 user directory option name | Placeholder `<user-dir option>`. Working name `--template-dir` is proposed because it mirrors the CLI meaning, and is **not decided** |
| D-13 explicit port option | Code accepts a port value internally. The command-line flag is **not exposed** until approved |
| D-16 limits | Named constants in one module, values **provisional placeholders** marked in code and docs |
| D-19 does loader source-read compile | Plan avoids the question at runtime (direct disk reads). Verified by investigation I-2 |
| D-22 packaging | Investigation I-1 |
| D-24 targets | Measured and recorded, not asserted |
| D-26 constitution ratification | Cited as draft |
| D-27 baseline run and tool installs | Gated, section 10 |
| D-29 coverage treatment | Not decided. Default of including the new code is provisional |

## Constitution Check

*GATE: checked before Phase 0 research and again after Phase 1 design. The constitution is an unratified draft, so this checks against draft guidance.*

| Principle | Assessment | Result |
|---|---|---|
| I. Template-First Mapping | The UI changes nothing the converter emits, and no mapping logic is added. **Tension found**: the principle says "Python code changes are limited to the loaders, renderers, parsers, filters, tags and CLI". A new `fhir_converter/ui/` package is Python outside that list. It is purely additive and read-only, but the sentence read literally does not cover it | **Needs a developer ruling** (N-01). Either the sentence is amended to cover tooling, or the developers state it is intended for output-affecting code only. Not silently resolved |
| II. Output Stability | No change to ids, public API, date rule, CLI or output. Regression diff planned (section 9.5) | Pass, subject to verification |
| III. Synthetic Data and Privacy | No documents in, no output shown, synthetic fixtures only, no `data/sample` use, no exception text or absolute paths in responses | Pass |
| IV. Verify Before Claiming | Baseline, before/after diff, new tests, packaging and source-read investigations, and nothing described as FHIR-valid | Pass, subject to execution |
| V. Minimal, Scoped Change | One new package, new tests, docs. No existing module edited, no mass edit, no pin change. Configuration touched only if I-1 proves it necessary, with approval | Pass |
| Technical Constraints | Python 3.8 syntax rules, full type annotations, flake8 limits, `RenderingError` conventions (not needed, no render), no `logging`. The UI-specific bullet (loopback only, no paths from request input) is met by design | Pass. One item needs approval: console output (FR-007, D-20) |
| Governance | Plan is checked against the draft principles. Deviations are listed, not hidden | Pass |

**Gate result**: no unjustified violation. One open ruling (N-01) and one open approval (D-20, console output). Neither blocks writing the plan. Both block implementation.

**Post-design re-check**: the design in section 5 adds no dependency, edits no existing module, binds loopback only, reads no path from a request, and keeps the UI package out of `import fhir_converter`. N-01 and D-20 remain the only open items.

## Repository findings that shape the design (Verified unless marked)

| Finding | Evidence | Effect on the plan |
|---|---|---|
| Templates live at `fhir_converter/templates/ccda` and `.../stu3`. `fhir_converter/templates` has **no `__init__.py`**, so it is a namespace folder | Directory listing | Packaged directory is located from the `fhir_converter` package directory on disk. Assumes an unzipped install (N-05) |
| Default loaders are `PackageLoader(package="fhir_converter.templates", package_path="ccda" or "stu3")` | `renderers.py:59-67` | The UI does not need these loaders. They are used only in tests to cross-check name mapping (FR-013 AC2) |
| Name rule: only a name with more than one path part gets `_` prepended to its last segment, unless it ends `.json` or already starts `_` | `loaders.py` `_resolve_template_name` (private) | Re-implemented in the catalog (D-14) and tested against the loader's public behavior |
| User directory searched first, packaged second | `__main__.py:86-94` | Effective graph rule (FR-041, D-15) |
| `TemplateSystemLoader` extends python-liquid's `ChoiceLoader`. `read_text` calls `env.loader.get_source(...)` | `loaders.py` | Source-reading question for I-2 |
| **Preliminary, read-only inspection of the local python-liquid 1.12.1**: `BaseLoader.load` calls `get_source` and then `env.from_string` (the parse step), and `Environment.get_template` calls `loader.load`. So `get_source` alone returns text and does not parse | `liquid/builtin/loaders/base_loader.py:126-148`, `liquid/environment.py:342-366` | Suggests reading through `get_source` or direct file reads does not compile. **Not confirmed**: I-2 must confirm by test and also read the `ChoiceLoader` path. D-19 stays open |
| Template references appear in three forms and all packaged ones are single-quoted literals. No `render` tag, no dynamic names | Grep counts in doc 04 section 2 | Literal-only scanner covers the shipped files. Dynamic and unsupported cases need fixtures |
| `pyproject.toml` packages `fhir_converter` with `include`, no `MANIFEST.in`, `.gitignore` has no html, js or css rule (partial search) | `pyproject.toml`, `.gitignore` | Static files under `fhir_converter/ui/static/` are probably included. Unproven until I-1 |
| `__main__.py` uses `StrEnum` (3.11+) | `CLAUDE.md` section 4 | Not a model for UI code |
| Existing tests: `tests/test_loaders.py`, `test_renderers.py` and others, pytest, `unittest.TestCase` style with `tmp` data under `tests/data` | Directory listing | New tests follow the same style |
| No logging usage, CLI prints with its own helpers | `CLAUDE.md` section 5 | No `logging`. Startup message only |

## Architecture

### Layers

```text
Browser (plain HTML + CSS + JavaScript, static, no framework, no external hosts)
        |  HTTP on 127.0.0.1:<auto port>, GET/HEAD only, JSON, per-run token header
        v
fhir_converter/ui/server.py     routing, header checks, error mapping, static files, log suppression
        |                       uses
        v
fhir_converter/ui/security.py   Host/Origin check, token, name and limit validation (pure functions)
        |
        v
fhir_converter/ui/search.py     name search (pure), source search (bounded, via catalog.read_source)
fhir_converter/ui/graph.py      effective graph, direct, inverse, transitive, root reach, section flags (pure)
fhir_converter/ui/scanner.py    text-only reference scanner (pure)
        |
        v
fhir_converter/ui/catalog.py    the ONLY module that touches the filesystem
        |
        v
fhir_converter/templates/{ccda,stu3}  (package, read-only)   +   optional user directory (read-only)
```

Dependency direction is strictly downward. `scanner` and `graph` take strings and plain data and never import `pathlib`-based IO, so they are unit tested without a server or files (NFR-008, FR-092 AC2). `catalog` is the only module that opens files.

### Startup sequence (`python -m fhir_converter.ui`)

1. Parse arguments (user directory, and port only if D-13 is approved).
2. Validate the user directory: must exist, be a directory and be readable. On failure print a fixed message and exit non-zero before opening any socket (FR-004).
3. Build the catalog for both formats (packaged plus user), scan every template once, build the effective graph for each format, compute section flags and counts. Record the build time (FR-006, NFR-013).
4. Create the per-run token from `secrets` (SEC-018).
5. Bind to `127.0.0.1` with port 0 unless an approved explicit port is given, and read back the real port (FR-002, FR-005).
6. Print one startup message with the address, port and the token-bearing address (FR-001, FR-007, **[APPROVAL]** D-11, D-20).
7. Serve until interrupted. Per-request logging is off.

### Request flow

1. Handler receives a request. Methods other than GET and HEAD get the generic rejection (FR-080).
2. Host header must be an allowed value, else generic rejection (SEC-002).
3. If an Origin header is present it must equal the server's own origin, else rejection (SEC-002).
4. `/` and the three static files are served after the Host check. Data routes under `/api/` also need the token header, compared in constant time (SEC-018).
5. Query values are validated against fixed allowlists (format, category, origin) and the template name against the catalog dictionary. A request value is never joined into a path (SEC-003).
6. The route calls pure functions on the in-memory catalog and graph, or reads one catalogued file (`catalog.read_source`).
7. The response is JSON with security headers. Any exception becomes a fixed message and code, never exception text (FR-070).

### Components

| Module (all under `fhir_converter/ui/`) | Responsibility | Spec coverage |
|---|---|---|
| `__init__.py` | Package marker and docstring only. Imports nothing from the server so importing the package stays cheap | FR-001 AC3, COMP-007 |
| `__main__.py` | Argument parsing, startup validation, build, bind, startup message, serve loop, exit codes | FR-001, FR-003, FR-004, FR-005, FR-007 |
| `catalog.py` | Enumerate `.liquid` files, map logical names, categories, origin, overrides, size, relative path, section id (OID). Holds the immutable `Catalog`. Reads source text for a catalogued file. Symlink containment. The only filesystem module | FR-006, FR-010 to FR-014, FR-030, FR-031, FR-050, FR-051, FR-092, SEC-005, SEC-006 |
| `scanner.py` | Strip comment blocks keeping line numbers, find references in the three forms, classify literal, dynamic and unsupported, record form and line | FR-040, FR-045, NFR-014 |
| `graph.py` | Build the effective graph per format, direct and inverse edges, target states, transitive sets with a shortest chain, root reach, section flags, dynamic counts | FR-041, FR-043, FR-044, FR-060 |
| `search.py` | Name search, source search with caps, skips and cut-off reporting | FR-020, FR-021 |
| `security.py` | Allowed Host values, Origin check, token create and compare, name and traversal validation, limit constants, extension allowlist | SEC-002 to SEC-006, SEC-018 |
| `errors.py` | Fixed error codes and messages, one mapping from internal failure kinds to them | FR-070, SEC-007 |
| `server.py` | Handler, routes, static file allowlist, security headers, method rejection, logging off, server factory usable by tests | FR-002, FR-007, FR-080 to FR-082, SEC-001, SEC-009 to SEC-011, SEC-014 |
| `static/index.html`, `static/app.js`, `static/style.css` | The page. No framework, no external URLs, text inserted with `textContent`, no `innerHTML`, `eval` or `document.write` | FR-030, FR-042, FR-044, FR-090, NFR-003, SEC-011 |

## Design details

### 6.1 Catalog

- **Roots**: packaged `ccda` and `stu3` directories under the `fhir_converter` package directory, and the optional user directory. Walk each for `*.liquid` only (FR-014). Sort by logical name for determinism (NFR-005).
- **Logical name**: relative path with forward slashes. If the path has more than one part, the last part must start with `_`, which is removed, then the `.liquid` suffix is removed. A one-part file keeps its name. This copies the loader's rule (D-14), and a test checks it against `read_text` through `make_environment` for every fixture template.
- **Origin and format**: packaged entries belong to one format. User entries have no format and are projected into both format views (FR-051, A7).
- **Overrides**: in a format view, a user entry whose logical name equals a packaged entry's is an override. The effective source is the user one. Both entries stay viewable (FR-050).
- **Metadata**: size, relative path (never absolute, SEC-008), category, origin, section id when found (see 6.5).
- **Symlinks and containment**: each enumerated user file is resolved and must lie inside the resolved user root. Others are skipped and counted. Re-checked at read time. Tests skip where symlinks cannot be created and report the skip (SEC-005).
- **Startup snapshot**: lists, metadata and graph are built once. Source text is read on request (FR-006, D-17).

### 6.2 Provisional behavior pending decisions

| Topic | Provisional behavior until the developers decide |
|---|---|
| Category (D-08) | C-CDA: top-level files are "root"; first folder name is the category when it is one of the listed folders; anything else is "other". STU3: "root" for top-level, otherwise the first folder name. An unknown category in a request returns the generic error (FR-011) |
| User file without `_` in a sub-folder (D-09) | Not mapped to a logical name. Excluded from list and graph. Counted and the count is shown in the page metadata. The names are not listed |
| Limits (D-16) | One `limits` block holds: maximum displayed file size, maximum search results, maximum query length, search time bound, request timeout, displayed tag-text length. Each is a named constant with a provisional placeholder value and a comment pointing at D-16. They are not final and tests refer to the constants, not literals |

### 6.3 Scanner

Text-only. It does not import python-liquid and never calls parse or compile (FR-083). `tags.py` and python-liquid internals are treated as fragile (`CLAUDE.md`), so a tokenizer from the library is not used. A library tokenizer would also require parsing.

Algorithm, as small functions to respect flake8 complexity 10:
1. **Blank comment blocks**: replace the text of `{% comment %}` to `{% endcomment %}` with newline-preserving blanks, so line numbers stay correct. Proposed the same for `{% raw %}` blocks (D-04 open).
2. **Find constructs**: locate `{% ... %}` tags and `{{ ... }}` outputs. For each, test three forms:
   - tag `include`, argument is the template reference.
   - tag `evaluate <var> using <arg>`, the argument after `using` is the reference.
   - filter `batch_render:` followed by its first argument. The second argument is a variable name and is not recorded.
3. **Classify the argument**: single-quoted literal is a literal reference. A double-quoted literal is literal only if D-04 adopts it, otherwise dynamic. Anything else (identifier, expression, concatenation) is dynamic.
4. **Unsupported**: a `render` tag, or a `{%` or `{{` opened for a reference keyword that never closes, or text that matches a reference keyword but cannot be read.
5. **Record**: kind (literal, dynamic, unsupported), form, target when literal, line number, and a length-capped tag text for display (limit D-16).
6. **Section id**: for C-CDA section templates, the literal in a `get_ccda_section_by_template_id` or `get_first_ccda_sections_by_template_id` call, else "not detected" (FR-031 AC2, **[APPROVAL]**). Provisional approach, needs checking against the real files in the investigation.

Known limit, stated in the UI and docs (FR-042, NFR-014): a text scan, not a parser. A reference built any other way is missed or reported dynamic.

### 6.4 Graph

- **Nodes** per format: effective logical names (user wins). Edges from effective sources. A literal target is `found` if it is in the effective catalog, `not found` if absent, `outside catalog` if it names a non-`.liquid` file (FR-014 AC4).
- **Direct**: adjacency and reverse adjacency, deduplicated per target with the set of forms.
- **Transitive**: breadth-first search from the template, keeping a parent map so each entry has a shortest chain. Transitive = reachable minus direct minus self. Reverse direction the same on reverse edges. A visited set makes cycles and self references terminate (FR-043 AC4). Computed on request from precomputed adjacency, since the catalog has about 800 nodes (NFR-002).
- **Roots (C-CDA)**: top-level effective templates, including `Header` and top-level user templates (D-06, **[APPROVAL]**), derived from the catalog and never a fixed list.
- **Root reach**: multi-source breadth-first search from all roots. A section is "potentially unreferenced" if it is not reached. Each flag carries the count of dynamic and unsupported references found in roots and in reached templates (FR-060 AC3).
- **Four states** (FR-044): the API returns separate fields, `direct`, `transitive`, `dynamic_or_unsupported`, and a boolean `no_references_detected` that is true only when there are zero literals and zero dynamic or unsupported entries. The "referenced by" side returns the catalog-wide dynamic count and an `incomplete` flag instead of per-target attribution (FR-044 AC4).
- **Overridden templates**: the effective source drives the graph. The packaged version's own references are returned in a separate `packaged_version` block (FR-041 AC4, D-15).
- **No special names**: nothing refers to Goals, Health Concerns or Instructions or any named section (FR-093).

### 6.5 Search

- **Name**: casefold substring on the logical name, combined with format, category and origin filters. Empty query returns the full filtered list (FR-020, **[PROPOSED]**).
- **Source**: loop over catalogued files for the format, read each through `catalog.read_source` with the size cap, skip unreadable, non-UTF-8 or oversize files and count them, stop at the result cap or time bound and say so (FR-021). Reads current disk text (D-17).

### 6.6 Security controls

| Control | Where | Spec |
|---|---|---|
| Bind `127.0.0.1` only, no option to change | `__main__.py`, `server.py` | FR-002, SEC-001 |
| Host allowlist: `127.0.0.1:<port>` and `localhost:<port>` (D-10) | `security.py` | SEC-002 |
| Origin must equal own origin when present | `security.py` | SEC-002 |
| Per-run token from `secrets`, constant-time compare, required on every `/api/` request, not logged, not stored | `security.py`, `server.py` | SEC-018 |
| Proposed token transport: the startup address carries the token in the URL **fragment**, so it is not sent in the request line and not in `Referer`. The page script reads it and sends it in a custom request header on each API call. A custom header also blocks simple cross-site requests. Static files need only the Host check | `server.py`, `app.js` | D-11, **[APPROVAL]** |
| Names validated only by dictionary lookup in the catalog. No request value reaches a path | `security.py`, `catalog.py` | SEC-003 |
| Traversal rejections: `..`, absolute, drive letter, backslash, NUL, and URL-decoded and double-decoded forms | `security.py` | SEC-004 |
| `.liquid` only, size cap, symlink containment | `catalog.py` | SEC-005, SEC-006 |
| GET and HEAD only, others get the generic rejection, no route accepts a body | `server.py` | FR-080, SEC-009 |
| No render or compile call, no document input | whole package | FR-081, FR-082, FR-083, SEC-010, SEC-012 |
| Escaping: JSON responses only, the page uses `textContent`. A test scans the static files for `innerHTML`, `outerHTML`, `eval`, `document.write`, `insertAdjacentHTML` and any `http` URL | `static/`, tests | SEC-011, FR-090 AC2 |
| Defense-in-depth headers, **[PROPOSED]**: `Content-Security-Policy` limited to same origin, `X-Content-Type-Options: nosniff`, `Cache-Control: no-store`, `Referrer-Policy: no-referrer` | `server.py` | SEC-011, SEC-013. Not in the spec, listed in N-04 |
| Console: `log_message` and `log_error` overridden to do nothing. The only output is the startup message and fatal startup errors | `server.py`, `__main__.py` | FR-007, SEC-014, **[APPROVAL]** D-20 |
| Fixed error messages and codes, no stack trace, no absolute path | `errors.py`, `server.py` | FR-070, SEC-007, SEC-008 |
| No files written, no disk cache, no history | whole package | SEC-013 |

### 6.7 Server

- Uses `ThreadingHTTPServer` with a `BaseHTTPRequestHandler` subclass. The standard-library choices are available on all of 3.8 to 3.12. Whether the exact behavior (HEAD, port 0 binding, shutdown, log suppression) is the same on each version and platform is checked by tests and investigation I-4.
- A `create_server(config)` factory returns the server and the token, so tests can start it on port 0 in a thread, send requests with `http.client`, and call `shutdown()`.
- The catalog and graph are immutable after startup, so handler threads need no locks.
- Request timeout is applied to the handler (limit D-16).

### 6.8 Page

One HTML page, one script, one stylesheet, served from an explicit allowlist of three files read from `fhir_converter/ui/static/` (D-25, **[APPROVAL]**). Views: list with format, category, origin and override filters; name and source search; template view with metadata, source and the four relationship states; unreferenced-section view; a fixed best-effort note; build time and restart note. Text only, keyboard operable, visible focus, state shown by words not color alone (NFR-003). No external fonts, scripts or images (FR-090).

## HTTP and command-line interface

Defined in [contracts/ui-contract.md](contracts/ui-contract.md). Summary: `GET` and `HEAD` only. Static: `/`, `/static/app.js`, `/static/style.css`. API under `/api/` with the token header: `meta`, `templates` (list and filters), `template` (metadata and relationships), `source`, `search/source`, `sections/unreferenced`. Names are passed as query values, never as path segments.

## Files expected to change

**New (all in the working tree, none exist yet)**

| Path | Purpose |
|---|---|
| `fhir_converter/ui/__init__.py`, `__main__.py`, `catalog.py`, `scanner.py`, `graph.py`, `search.py`, `security.py`, `errors.py`, `server.py` | UI code |
| `fhir_converter/ui/static/index.html`, `app.js`, `style.css` | The page |
| `tests/test_ui_catalog.py`, `test_ui_scanner.py`, `test_ui_graph.py`, `test_ui_search.py`, `test_ui_security.py`, `test_ui_server.py`, `test_ui_process.py`, `test_ui_regression.py` | Tests |
| `tests/data/ui/` (small invented `.liquid` fixture trees) | Synthetic fixtures. Trees may instead be built in `tmp_path` inside tests, to avoid a large static fixture. Undecided, either is allowed |
| A docs page, for example `docs/template-ui.md`, and a README link | NFR-009 |
| `CHANGES.md` | New feature entry |
| `docs/test-cases.md` | New cases, statuses only after they are run |

**Existing files not changed**: every file in `fhir_converter/` outside `ui/`, every template, `ValueSet.json`, every existing test, `__main__.py`, `renderers.py`, `loaders.py`, `pyproject.toml` scripts and dependencies.

**Conditionally changed, approval required first**: `pyproject.toml` (include or exclude rules) or ignore files, only if investigation I-1 shows the static files do not ship (FR-094 AC3). `mkdocs.yml` only if the docs page needs a navigation entry (not checked).

**Never edited**: `docs/docstrings/`, `site/`, `htmlcov/`, `.venv` (`CLAUDE.md` section 8.1).

## Verification strategy

### 9.1 Investigation steps (explicit)

| ID | Investigation | Method | Gate |
|---|---|---|---|
| **I-1** | **Poetry packaging verification** (FR-094, D-22, COMP-006). Do the built sdist and wheel contain `fhir_converter/ui/static/*`? | (1) Read the full `pyproject.toml` and `.gitignore`, and check whether Poetry is installed. (2) In a **temporary copy of the repository outside the working tree**, add placeholder static files under `fhir_converter/ui/static/`, build the sdist and wheel into a temporary folder, and list both archives. (3) Install the wheel into a clean temporary virtual environment and run `python -m fhir_converter.ui` to confirm it starts and serves the three files. (4) Repeat on the real files once they exist. (5) If any file is missing, write the smallest configuration change as a proposal for approval and do not edit. Preliminary evidence only: `packages` include, no `MANIFEST.in`, no html, js or css ignore rule found by partial search | **Approval needed** to install Poetry or build tooling and to run a build (D-22, D-27). Not run |
| **I-2** | **python-liquid source-reading verification** (D-19, FR-083). Does reading template text through the loader avoid compiling it, and is the runtime plan (direct disk reads) safe? | (1) Read `BaseLoader.get_source`, `BaseLoader.load`, `ChoiceLoader.get_source` and `TemplateSystemLoader.get_source` in python-liquid 1.12.1 and the project's `loaders.py` end to end. Preliminary read already shows `load` calls `get_source` then `from_string`. (2) In a temporary script outside the repository, build `make_environment`, point it at a temporary folder holding a template with a deliberate Liquid syntax error, call `read_text` and confirm it returns the text with no error, then call `get_template` and confirm it raises. (3) Record the result in `research.md`. (4) Decide the instrumented test: assert `get_template` and the render methods have zero calls in a UI session | Reading code needs no approval. Running the temporary script is a command, so it waits for the same go-ahead as the baseline (D-27). Not run |
| **I-3** | **Python 3.8 to 3.12 coverage**. The local `.venv` is 3.14.10 | List which interpreters exist locally. If none of 3.8 to 3.12, rely on the CI matrix and keep UI code to a conservative 3.8 subset, enforced by a test that parses every UI module with the `ast` module using `feature_version=(3, 8)` and by review of the avoid list (COMP-002) | Local installs of other Pythons need approval. CI needs no local action |
| **I-4** | **Standard-library server behavior**: port 0 binding, HEAD handling, shutdown, log suppression, Windows and macOS differences, `ThreadingHTTPServer` on 3.8 | Prototype in a temporary folder, then confirm by the server tests on the CI matrix | Running the prototype waits for the go-ahead |
| **I-5** | **Scanner calibration**: scanner counts against the packaged templates | After implementation, compare scanner totals with the grep counts (619 `include`, 188 `batch_render`, 24 `evaluate` single-quoted literals) and explain any difference. Also confirm there is no construct the scanner mislabels as dynamic | Part of the test run |
| **I-6** | **Section id detection**: which literal forms in section templates carry the OID | Read the C-CDA section templates and list the call forms before fixing the pattern (FR-031 AC2, **[APPROVAL]**) | Read only |

### 9.2 Unit tests (NFR-008)

Pure modules, no server. Fixture catalog trees built in `tmp_path`.

| File | Covers | Spec scenarios |
|---|---|---|
| `test_ui_catalog.py` | `.liquid`-only scope, logical-name mapping against the loader's public behavior, categories, origin, overrides, effective source, metadata, relative paths, symlink containment, user-directory problems, read-on-request, size cap | T01 to T04, T08, T11, T16, T25, T26, T36 |
| `test_ui_scanner.py` | Three forms, second `batch_render` argument ignored, comment blocks ignored, dynamic and unsupported cases, unterminated tag, line numbers, double-quote handling per D-04, no parse or compile call | T09, T14 |
| `test_ui_graph.py` | Direct and inverse, target states, transitive with chains, cycles, self reference, four states, root reach, section flags, dynamic count and incompleteness, effective graph and packaged version, STU3 has no flag, independence from section names | T10 to T13, T15, T17, T18, T19, T27, T34 |
| `test_ui_search.py` | Name search rules, source search caps, skips, cut-off note, non-UTF-8 files | T05, T06 |

### 9.3 Server tests

`test_ui_server.py` starts `create_server` on port 0 in a thread and uses `http.client`. Covers: routes and JSON shapes, static allowlist, GET and HEAD only, rejected methods and unknown routes, generic errors including a marker-string injection that must not appear in body, headers or captured output, source view inert and escaped, no route accepts input data, refresh route absent, build time and restart note present (T07, T13 error side, T21 to T23, T29, T36).

`test_ui_process.py` uses `subprocess` to run `python -m fhir_converter.ui`: startup message content and timing, two simultaneous instances get different ports, bad user directory exits non-zero with the fixed message and leaves no listener, captured standard output and error equal the startup message only across a session, `import fhir_converter` does not load the UI package (T01, T32, T33, T35, T39).

### 9.4 Security tests

`test_ui_security.py` plus server-level cases (T16, T18 to T21, T24 to T26, T29 to T31, T37):

| Case | Method |
|---|---|
| Bound address is `127.0.0.1` and no option changes it | Inspect socket, check arguments |
| Host and Origin | Allowed, foreign Host, wrong port, foreign Origin, absent Origin |
| Token | Missing, wrong, right. Differs between runs. Absent from files and from console beyond the startup message |
| Traversal | Table of names: `..`, `%2e%2e`, double-encoded, absolute, drive letter, backslash, NUL, long names |
| Extension and size | `.json`, `.txt`, oversized `.liquid` refused |
| Symlink escape | Skipped where the platform lacks symlinks, skip reported |
| Output encoding | Fixture template with HTML and script text in name and body is inert. Static files scanned for forbidden APIs and external URLs |
| Read-only | File tree hash before and after a full session. POST, PUT, PATCH, DELETE rejected |
| Privacy | Search all responses for the temp absolute path and the marker string. No document input route. No renderer or compile call recorded |
| Wording | A scan of all UI text for the forbidden definitive words (FR-042 AC3, D-18) |

### 9.5 Regression tests (FR-091)

| Check | Method | Notes |
|---|---|---|
| Conversion output unchanged | Before any edit, convert `TD-CCDA-01` to `12` except `08`, `tests/data/ccda/CCD.ccda` and `History_and_Physical.ccda` into a temporary folder. Repeat after, then diff. Expect zero differences | Temporary folder only. `data/out` is not used. Not run |
| Public API unchanged | `test_ui_regression.py` records `inspect.signature` strings for `render_fhir_string`, `render_fhir`, `render_to_fhir`, `make_environment`, `render_to_dir`, `render_files_to_dir` | New test |
| CLI unchanged | Capture `python -m fhir_converter --help` before and after and compare. Rerun TC-CLI cases manually against `docs/test-cases.md` | `__main__.py` has no unit tests |
| Existing suite | `.\.venv\Scripts\python.exe -m pytest -q`, record count before and after (expected 475, unverified) | Needs the go-ahead for the baseline (D-27) |
| Goals independence | Run the UI tests on a tree with and without a wired Goals section, and confirm no named-section assertion exists (FR-093, T34) | Does not require the Goals feature to be merged. A temporary copy of the templates can simulate both states |
| Import isolation | Subprocess check that `fhir_converter` import does not load `fhir_converter.ui` | In `test_ui_process.py` |

### 9.6 Quality gates (NFR-007)

mypy with the project settings and flake8 (120 columns, ignore `E203,E721`, complexity 10) over `fhir_converter/ui/`. Both are run by CI. They were not installed locally when last checked, so local runs need approval to install (D-27). The `ast` feature-version test in I-3 gives an extra local 3.8 syntax check.

### 9.7 Measurements to record, not assert

List and open timings, startup build time, transitive-view time and source-search time against the real package (NFR-001, NFR-002, NFR-013, T43). The numbers in the spec are proposed targets, so the result is recorded and reported, and the developers decide whether it meets D-24.

### 9.8 Requirement coverage

| Spec requirements | Components | Primary tests |
|---|---|---|
| FR-001 to FR-007 | `__main__`, `server`, `security` | `test_ui_process`, `test_ui_server` |
| FR-010 to FR-014 | `catalog` | `test_ui_catalog` |
| FR-020, FR-021 | `search`, `catalog` | `test_ui_search` |
| FR-030, FR-031 | `catalog`, `scanner`, `server`, `app.js` | `test_ui_catalog`, `test_ui_server` |
| FR-040, FR-045 | `scanner` | `test_ui_scanner` |
| FR-041 to FR-044 | `graph` | `test_ui_graph` |
| FR-050, FR-051 | `catalog`, `graph` | `test_ui_catalog`, `test_ui_graph` |
| FR-060 | `graph` | `test_ui_graph` |
| FR-070 | `errors`, `server` | `test_ui_server`, `test_ui_security` |
| FR-080 to FR-083 | `server`, whole package | `test_ui_server`, `test_ui_security` |
| FR-090 to FR-094 | whole package, I-1 | `test_ui_regression`, `test_ui_process`, I-1 |
| NFR-001 to NFR-014 | all | measurements, mypy, flake8, review |
| SEC-001 to SEC-019 | `security`, `server`, `catalog`, `app.js` | `test_ui_security`, `test_ui_server` |
| COMP-001 to COMP-008 | all | CI matrix, I-3, I-4, I-1 |

All 43 scenarios (T01 to T43) are covered by these files and by the investigation and measurement items. T28 (existing CLI cases) and T41 (packaging build) are manual or approval-gated, and T25, T42 are partly manual (keyboard walk).

## Approval-gated actions (not executed)

| Action | Gate |
|---|---|
| Starting any implementation | FR-012 style gate for this feature: developers approve the spec and the open **[APPROVAL]** decisions first |
| Baseline `pytest` run | D-27 |
| Installing flake8, mypy, Poetry or build tooling | D-22, D-27 |
| Running a package build and a clean-venv install check (I-1) | D-22, D-27 |
| Running the temporary scripts for I-2 and I-4 | Same go-ahead as the baseline |
| Installing other Python versions for I-3 | Approval |
| Any `pyproject.toml` or ignore-file edit | FR-094 AC3, only if I-1 shows it is needed |
| Exposing an explicit port flag | D-13 |
| Choosing the user directory option name | D-12 |
| Suppressing per-request logging and printing one startup message | D-20 |
| Token transport choice | D-11 |
| Edits to the constitution or its ratification | D-26, N-01 |

## Open decisions kept visible

Every spec decision D-01 to D-29 remains as the spec states it. The plan's provisional treatment of each is in the Technical Context table and in section 6.2. In addition, planning surfaced items that are not in the spec's register:

| ID | Item | Why it matters | Needs approval |
|---|---|---|---|
| N-01 | Constitution principle I limits Python changes to loaders, renderers, parsers, filters, tags and CLI. A new `fhir_converter/ui/` package is not in that list | A literal reading forbids it. The constitution is an unratified draft, so it can be clarified before ratification | Yes |
| N-02 | User files in a sub-folder without `_` are skipped and counted, not shown (provisional, D-09) | Silent skipping could confuse users | Yes |
| N-03 | Limit values are placeholders until D-16 | Tests and docs refer to constants | Yes |
| N-04 | Defense-in-depth response headers (CSP, `nosniff`, `no-store`, `no-referrer`) are proposed and not in the spec | They support SEC-011 and SEC-013 but are extra behavior | Yes |
| N-05 | Packaged templates are read from the filesystem next to the `fhir_converter` package. A zipped or frozen install would not work | The existing loaders use `PackageLoader`, which may handle more install styles | Yes |
| N-06 | Local Python is 3.14.10, so 3.8 compatibility cannot be proven locally | Relies on CI and a syntax-level check | Informational |
| N-07 | Packaging check is done in a temporary copy before real assets exist | Avoids editing the repository early | Yes (build approval) |
| N-08 | Static files live at `fhir_converter/ui/static/` and are served by an explicit three-file allowlist | Part of D-25 and packaging | Yes |
| N-09 | Fixtures built in `tmp_path` versus committed under `tests/data/ui/` | Reviewability versus duplication | No |

## Risks

| Risk | Mitigation |
|---|---|
| Regex scanner misreads Liquid | Documented as best-effort. Calibrated against real files (I-5). Anything unreadable is reported as dynamic or unsupported, never dropped |
| False "potentially unreferenced" flags | Wording rules, incompleteness note, never definitive (FR-042, FR-060) |
| Packaging omits static files | I-1 before any configuration change |
| `http.server` differences across Python and platforms | Server tests in the CI matrix, I-4 |
| Token in the startup message is visible on the local console | Accepted local exposure, to be confirmed (SEC-019, D-11) |
| Python 3.9+ syntax slips in | `ast` feature-version test, mypy and flake8 in CI, avoid list in review |
| Scope creep toward editing or rendering | Out-of-scope list in the spec, tests asserting no write, render or compile call |
| Unratified constitution | Cited as draft, N-01 raised |
| Baseline unknown | Gated run before implementation (D-27) |

## Phases (ordering, not tasks)

No `tasks.md` is generated here.

1. **Gate**: developers review the spec and this plan, decide the open items, and give the go-ahead for the baseline and tooling (D-26, D-27, D-22).
2. **Investigate**: I-2, I-4, I-6, then I-1 in a temporary copy (all after the go-ahead).
3. **Core, pure modules**: `scanner`, `graph`, `search`, `security`, `errors`, then `catalog`, each with unit tests.
4. **Server and page**: `server`, `__main__`, static files, server and process tests.
5. **Security and regression**: security suite, regression diff, API and CLI checks, Goals-independence run.
6. **Docs and packaging**: docs page, `CHANGES.md`, `docs/test-cases.md` after running, real-asset packaging check I-1 step 4.
7. **Review**: measurements recorded, constitution check re-run, results reported with anything not run.

## Project Structure

### Documentation (this feature)

```text
specs/002-template-management-ui/
├── spec.md
├── plan.md              # This file
├── research.md          # Phase 0
├── data-model.md        # Phase 1
├── quickstart.md        # Phase 1
├── contracts/
│   └── ui-contract.md   # Phase 1
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2 (/speckit-tasks, not created here)
```

### Source code (repository root)

```text
fhir_converter/
├── ui/                      # NEW, all UI code
│   ├── __init__.py
│   ├── __main__.py
│   ├── catalog.py
│   ├── scanner.py
│   ├── graph.py
│   ├── search.py
│   ├── security.py
│   ├── errors.py
│   ├── server.py
│   └── static/
│       ├── index.html
│       ├── app.js
│       └── style.css
└── (all existing modules and templates: unchanged)

tests/
├── test_ui_catalog.py       # NEW
├── test_ui_scanner.py       # NEW
├── test_ui_graph.py         # NEW
├── test_ui_search.py        # NEW
├── test_ui_security.py      # NEW
├── test_ui_server.py        # NEW
├── test_ui_process.py       # NEW
├── test_ui_regression.py    # NEW
└── data/ui/                 # NEW, optional
```

**Structure Decision**: the existing single-project library layout. One new subpackage, no change to the package list in `pyproject.toml` (the existing `include = "fhir_converter"` covers it, to be confirmed by I-1).

## Complexity Tracking

| Item | Why needed | Simpler alternative rejected because |
|---|---|---|
| Per-run token plus Host and Origin checks | The spec requires both (SEC-002, SEC-018) | Host and Origin alone do not stop other local processes |
| Custom text scanner instead of a Liquid tokenizer | A tokenizer needs parsing, which FR-083 forbids, and python-liquid internals are fragile | Using the library lexer risks compile calls and internal-API breakage |
| Direct disk reads instead of the loader | Avoids the D-19 question at runtime and keeps a single filesystem module | Reading through the loader would couple the UI to `ChoiceLoader` behavior. The loader is still used in tests as the reference |
| Eight test modules | Separate pure, server, process and security concerns | One large module would mix fast unit tests with slower server and subprocess tests |

No constitution violation is claimed. N-01 is a wording tension for the developers to rule on.
