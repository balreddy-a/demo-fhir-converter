# Spec Inputs, Refined: Template Management UI (First Release)

Status: **REFINED DRAFT INPUTS for `/speckit-specify`. Not a specification. Nothing in this document is approved.** No `spec.md`, `plan.md` or `tasks.md` was created. No application code, template, test, dependency or configuration file was changed. No test, build or installation command was run. The only commands run for this refinement were read-only searches of the template files (section 2).

Audience: the developers who own `python-fhir-converter` (v0.3.0).

Date: 2026-11-05.

Replaces for review purposes: `docs/day2/03-template-ui-spec-inputs.md` (kept unchanged).

Sources: `docs/day2/01-template-ui-discovery.md`, `02-template-ui-decisions.md`, `03-template-ui-spec-inputs.md`, `CLAUDE.md`, `.specify/memory/constitution.md` (v1.0.0, an unratified draft), and the fourteen technical recommendations supplied with this task. This is an engineering proposal. It makes no claim that output is FHIR-valid and no compliance claim, and it is not a substitute for developer review.

## Approval status and tags

| Tag | Meaning |
|---|---|
| **[APPROVAL]** | A developer decision is still required. Every technical recommendation applied in this document carries this tag, because none has been formally approved |
| **[PROPOSED]** | A value or rule that I added where the sources are silent |
| (no tag) | Follows from the repository, `CLAUDE.md`, the constitution, or the discovery report |

Evidence tags: **Verified** (read or counted in this project), **Carried** (from the discovery documents), **Assumption** (not confirmed).

## Contents

1. What changed from document 03
2. New evidence gathered
3. Scope
4. Terminology
5. Functional requirements
6. Non-functional requirements
7. Security and privacy acceptance criteria
8. Compatibility (Python 3.8 to 3.12)
9. Test scenarios
10. Out of scope
11. Assumptions
12. Decision register
13. Traceability

---

## 1. What changed from document 03

### 1.1 The fourteen recommendations, as applied

| Ref | Recommendation | Applied in |
|---|---|---|
| R1 | Detect literal references in `include` tags, `batch_render` filters and `evaluate` tags | FR-040, FR-045 |
| R2 | Support direct and transitive relationship views | FR-041, FR-043 |
| R3 | Distinguish direct references, transitively reachable templates, no references detected, and dynamic or unsupported references | FR-044, FR-045 |
| R4 | Unreferenced-section detection is best-effort, never definitive | FR-042, FR-060 |
| R5 | Catalog is `.liquid` only. JSON stays outside unless explicitly approved | FR-014, SEC-006 |
| R6 | No template compilation check in Release 1 | FR-083, FR-081 |
| R7 | The user selects the format when comparing custom templates with packaged ones | FR-051, FR-050 |
| R8 | Suppress the standard-library server's default per-request console logging | FR-007 |
| R9 | Loopback binding, Host and Origin validation, and a local request protection mechanism | FR-002, SEC-002, SEC-018 |
| R10 | Automatically selected available local port by default | FR-005 |
| R11 | Catalog built at startup, restart to refresh | FR-006, NFR-006 |
| R12 | UI independent of the Goals Section work | FR-093 |
| R13 | Verify Poetry packaging before deciding on configuration changes | FR-094, COMP-006 |
| R14 | Python 3.8 to 3.12 and unchanged existing behavior | COMP-001 to COMP-008, FR-091 |

All fourteen are **[APPROVAL]**. They are the working proposal, not decisions the developers have made.

### 1.2 Contradictions found in document 03 and how they are resolved

| ID | Contradiction in 03 | Resolution here |
|---|---|---|
| X1 | FR-010, FR-013, FR-021 and SEC-006 treated `.json` as listable, searchable and readable. U-08 left `ValueSet.json` open | `.liquid` only everywhere (FR-014, FR-010, FR-013, FR-021, SEC-006). U-08 closed as proposed exclusion **[APPROVAL]** |
| X2 | U-04 and a conditional exclusion left a compile check possible | Excluded for Release 1 (FR-083). U-04 withdrawn |
| X3 | FR-081 said the UI does not load a Liquid environment, yet the catalog is meant to reuse loader behavior to read sources | Reworded. The UI may use the loader only to read text. It never renders and never calls the template compile step (FR-081, FR-083) |
| X4 | FR-001 printed the address at startup, while NFR-007 and `CLAUDE.md` bar new output channels, and U-13 asked about logging | A single startup message (address and port) is allowed, all per-request output is suppressed (FR-005, FR-007). **[APPROVAL]** |
| X5 | NFR-006 said a restart is needed, U-11 asked about refresh, and FR-021 did not say whether source is read at startup or on request | Catalog, metadata and relationships are built at startup. Source text is read from disk at request time. The page says so (FR-006) |
| X6 | FR-060 and U-03 disagreed on direct versus transitive reachability for sections | One rule: a section is highlighted only when no root template reaches it directly or transitively through detected references (FR-060) |
| X7 | FR-040 and FR-041 described includes only. FR-041 defined used-by as the inverse of includes | Generalized to the three reference forms (FR-040, FR-041) |
| X8 | FR-003 implied a startup option and U-05 left format selection open | The user directory is named at startup. The format is chosen in the UI, and is never inferred (FR-003, FR-051) |
| X9 | FR-002 allowed an optional `::1` listener while Host validation assumed one address | IPv4 loopback only by default, other forms need a decision (FR-002, SEC-002) |
| X10 | FR-042 said absence is never shown as proof, but a "none" state was also required | The wording is fixed to "No references detected" and the four states are separated (FR-042, FR-044) |
| X11 | FR-001 and U-14 left the port open | Automatic port is the default (FR-005) |
| X12 | FR-060 AC6 and test T27 mentioned the Goals, Health Concerns and Instructions sections | The UI result depends only on template contents. Tests must not assert any state for those sections (FR-093, T27, T34) |
| X13 | Whether `Header` and user top-level templates are roots was unstated | Defined in section 4 and FR-060 **[APPROVAL]** |
| X14 | Override and relationship views did not say which source (user or packaged) is used for an overridden template | Effective graph rule in FR-041 and FR-050 **[APPROVAL]** |
| X15 | SEC-002 left the protection mechanism undecided (U-12) | Host and Origin validation plus a per-run token (SEC-002, SEC-018) **[APPROVAL]** |

---

## 2. New evidence gathered

Read-only text searches of `fhir_converter/templates` (both formats), run during this refinement. The regular expressions are narrow, so counts are indicative, not a parser result.

| Check | Result | Tag |
|---|---|---|
| `include` tags with a single-quoted literal name | 619 | Verified |
| `include` tags with a double-quoted literal or a variable name | 0 | Verified |
| `batch_render:` filters with a single-quoted literal template name | 188 | Verified |
| `batch_render:` with a double-quoted or variable template name | 0 | Verified |
| `evaluate ... using` with a single-quoted literal name | 24 | Verified |
| `evaluate ... using` with a double-quoted or variable name | 0 | Verified |
| `render` tag in any template | 0 files | Verified |
| Literal references to a `.json` name through the three forms | 0 | Verified |
| Tags split across lines (open tag at end of line) | 0 | Verified |
| Files containing any of the three forms | 314 | Verified |
| `.gitignore` lines mentioning html, js, css, static or ui | only `htmlcov/` | Verified (partial search, not a full read) |
| `MANIFEST.in` | not present | Verified |

What this supports: in the packaged templates every template reference found is a single-quoted literal, so the first-release scanner can be literal-only and still cover the shipped files. It does **not** prove there are no other forms, and user templates may contain any form, so dynamic and unsupported handling stays required (FR-045). Tests for those cases must use fixtures, because the packaged templates contain none.

What it does not settle: whether Poetry includes static files under a new `fhir_converter/ui/` folder in a built package. That needs a build and inspection, which was not run (FR-094).

---

## 3. Scope

**In scope (all [APPROVAL])**
1. Read-only local browser UI, served by a Python standard-library HTTP server, code under `fhir_converter/ui/`, launched with `python -m fhir_converter.ui`.
2. Packaged C-CDA and STU3 templates, `.liquid` files only.
3. An optional user template directory. The user chooses the format to compare against.
4. List by format and category. Search by name and within source. View source and metadata.
5. Direct and transitive relationship views for three reference forms, with dynamic and unsupported references shown separately.
6. Identification of packaged templates overridden by user templates.
7. Best-effort highlight of potentially unreferenced C-CDA sections.
8. Generic error messages. Suppressed per-request console output.

**Out of scope:** section 10.

---

## 4. Terminology

| Term | Meaning |
|---|---|
| **Logical name** | The name used in a reference, such as `Section/Goal`. The file on disk is `Section/_Goal.liquid`. Root files such as `CCD.liquid` have no prefix. Mapping follows the loader's rule (`loaders.py`, Verified) |
| **Reference** | A literal quoted template name appearing in one of three forms: the `include` tag, the first argument of the `batch_render` filter, or the name after `using` in the `evaluate` tag |
| **Direct reference** | Template A contains a reference to template B |
| **Transitively reachable** | Template B can be reached from template A by following two or more direct references, and A does not reference B directly |
| **No references detected** | A template has zero detected literal references and zero dynamic or unsupported ones. It is a statement about the scan, never about the template's behavior |
| **Dynamic reference** | One of the three forms whose template name is not a quoted literal (for example a variable or an expression) |
| **Unsupported reference** | Any other construct that appears to load another template (for example a `render` tag), or a reference construct the scanner cannot read. None was found in the packaged templates (section 2) |
| **Root template (C-CDA)** | A top-level template in the C-CDA tree, such as `CCD` or `ConsultationNote`, determined from the catalog and not from a fixed list. `Header` is top-level and is treated as a root, because roots include it. User-directory top-level templates count as roots in the effective graph **[APPROVAL]** |
| **Effective source** | For a logical name present in both user and packaged sets, the user template, matching the loader's lookup order (`__main__.py:86-94`, Verified). Otherwise the only one that exists |
| **Effective graph** | The reference graph built from effective sources for the selected format |

---

## 5. Functional requirements

Each requirement has acceptance criteria (AC), positive scenarios (P) and negative scenarios (N). All test data is synthetic (constitution III). Criteria that would depend on the live package contents use a **fixture catalog** built inside the test, so they do not break when a template is added. IDs from document 03 are kept. New IDs are marked (new). No ID has been reused with a different meaning.

### 5.1 Launch and configuration

**TUI-FR-001 Launch.** The UI starts with `python -m fhir_converter.ui` and serves a local page. **[APPROVAL]**
- AC1: the command starts the server and prints one startup message with the local address and port, within 10 seconds on a developer machine. **[PROPOSED]** (10 seconds is mine)
- AC2: starting it needs no change to `pyproject.toml` scripts, `fhir_converter/__main__.py` or any existing module.
- AC3: `import fhir_converter` does not import `fhir_converter.ui` (checked by inspecting loaded modules).
- P: start with no arguments. Start with a user directory.
- N: `python -m fhir_converter` is unchanged (FR-091).

**TUI-FR-002 Loopback only.** The server accepts connections on the IPv4 loopback address only. **[APPROVAL]**
- AC1: the bound address is `127.0.0.1`, verified by inspecting the bound socket.
- AC2: no option in Release 1 binds to another address. Adding `::1` or any other address is a developer decision. **[APPROVAL]**
- P: a browser on the same machine connects.
- N: a connection from a non-loopback address is not accepted (verified where the environment allows, otherwise by inspecting the bound address).

**TUI-FR-003 Optional user template directory.** The user can name one directory of custom `.liquid` templates at startup, with the same meaning as `--template-dir` (searched before packaged templates). The format is not part of the startup option (FR-051). **[APPROVAL]**
- AC1: with a valid directory, its `.liquid` templates appear with origin "user" (FR-012).
- AC2: without the option, the catalog holds only packaged templates and the UI shows no user templates.
- AC3: the option name is undecided (D-12).
- P: a directory with two valid `.liquid` files is listed.
- N: missing path, a file instead of a directory, or an unreadable directory: see FR-004.

**TUI-FR-004 Startup validation.** A bad configuration produces a clear startup error and no partial server.
- AC1: for a missing, non-directory or unreadable user directory, the process exits with a non-zero code before serving and prints a fixed message with no exception text.
- AC2: no listening socket remains open after that failure.
- P: a valid directory starts normally.
- N: each of the three bad cases.

**TUI-FR-005 Port selection (new).** By default the server uses an automatically selected available local port. **[APPROVAL]**
- AC1: with no port option, the server binds to a port chosen by the operating system and reports the actual port in the startup message.
- AC2: two instances started at the same time use different ports and both work.
- AC3: if a developer approves an explicit port option (D-13), a port already in use produces a non-zero exit and a fixed message, and no listener remains.
- P: start two instances.
- N: explicit busy port (only if the option exists).

**TUI-FR-006 Catalog built at startup (new).** The catalog, metadata and relationship data are built once when the server starts. There is no refresh. A restart is required to see changes. **[APPROVAL]**
- AC1: a `.liquid` file added to the user directory after startup does not appear. A file removed after startup still appears in the catalog until restart.
- AC2: the page states that the catalog was built at startup and shows the build time. It offers no refresh control and no refresh route.
- AC3: source text shown by FR-030 and searched by FR-021 is read from disk at request time. The page states that metadata and relationships reflect the startup state. **[APPROVAL]** (D-17)
- AC4: a template that existed at startup but cannot be read at request time returns the generic error (FR-070), not a crash.
- P: add a file after startup, confirm it is absent. Restart, confirm it is present.
- N: a request for a refresh action is rejected as an unknown route.

**TUI-FR-007 Console output (new).** The default per-request console logging of the standard-library server is suppressed. **[APPROVAL]**
- AC1: during a session that exercises every feature, no request line, URL, query string, template name or path is written to standard output or standard error.
- AC2: the only console output is the startup message (FR-001, FR-005) and fatal startup errors (FR-004). No new logging channel is introduced (`CLAUDE.md` section 5).
- AC3: error responses do not log exception text (FR-070).
- P: capture both streams across a full session and compare with the expected startup message.
- N: an error request and a rejected request produce no console output.

### 5.2 Catalog: listing

**TUI-FR-010 List templates by format.** The UI lists templates for the selected format, C-CDA or STU3, as separate lists, `.liquid` files only.
- AC1: for a fixture package with N C-CDA and M STU3 `.liquid` templates, the C-CDA list has N entries and the STU3 list has M.
- AC2: against the real package, each count equals the number of `.liquid` files under `templates/ccda` and `templates/stu3` on disk at test time. The test does not hard-code 819 and 264, which were correct at discovery.
- AC3: files that are not `.liquid` (`metadata.json`, `ValueSet.json`, `cda_narrative.xsl`) are not counted.
- P: switch formats and see a different list.
- N: an unknown format value gets the generic error (FR-070).

**TUI-FR-011 Category.** Each template has a category derived from its location, and the list can be filtered by category.
- AC1: C-CDA categories cover at least root, `Section`, `Entry`, `Resource`, `Reference`, `DataType`, `Extension`, `Utils`, `ValueSet`, and "other". The discovery list omitted `Reference`, `Extension` and `Utils`, which exist (Verified). **[APPROVAL]** (D-08 for the taxonomy, and for STU3, whose structure was not re-read)
- AC2: every template has exactly one category and the per-category counts sum to the format total.
- AC3: filtering by a category returns only that category.
- AC4: a `ValueSet` folder may hold only `.json`, which is excluded (FR-014), so the category can be empty. The UI handles an empty category without error.
- P: filter `Section`.
- N: an unknown category gets the generic error or an empty list, whichever the spec chooses (D-08).

**TUI-FR-012 Origin.** Each template shows whether it comes from the package or from the user directory.
- AC1: every entry has the origin "package" or "user".
- AC2: only the override case (FR-050) shows a logical name with both origins.
- P: a fixture user template shows "user".
- N: with no user directory, no entry shows "user".

**TUI-FR-013 Logical names.** Templates are shown and addressed by logical name, mapped from the file with the loader's rule.
- AC1: `Section/_Goal.liquid` is shown as `Section/Goal`, and `CCD.liquid` as `CCD`.
- AC2: for every fixture template, the logical name resolves to that file through the loader's public behavior (a test compares the catalog mapping with the loader result). The private `_resolve_template_name` is not imported. **[APPROVAL]** (D-14)
- AC3: list order is deterministic (sorted by logical name).
- AC4: logical names always use forward slashes, on every platform.
- P: nested name `Entry/Problem/entry_act_entryRelationship`.
- N: a user file in a sub-folder without the `_` prefix is handled consistently with the loader (D-09).

**TUI-FR-014 Catalog scope is `.liquid` only (new).** Only files with the `.liquid` extension are catalogued, in the package and in the user directory. **[APPROVAL]**
- AC1: `.json`, `.xsl`, `.txt` and any other extension are not listed, counted, searched or viewable.
- AC2: a request for such a file by name returns the generic not-found error.
- AC3: a `.json` file in the user directory is ignored.
- AC4: a literal reference whose target is not a `.liquid` catalog entry is reported as "target outside catalog" (FR-041), not hidden.
- AC5: adding JSON files to the catalog later needs explicit approval and is not part of Release 1.
- P: fixture package and user directory each containing a `.json` file. Neither appears.
- N: requesting `ValueSet/ValueSet.json` by name is rejected.

### 5.3 Search

**TUI-FR-020 Search by name.** The user can search templates by name.
- AC1: a search returns every template whose logical name matches and none that do not.
- AC2: matching is a case-insensitive substring match on the logical name. **[PROPOSED]**
- AC3: an empty query returns the full list for the selected format. **[PROPOSED]**
- AC4: it combines with the format, category and origin filters.
- P: `goal` finds the Goal-named fixture templates.
- N: no match shows "no results", not an error. An overlong query gets the generic error (limit in D-16).

**TUI-FR-021 Search within source.** The user can search inside `.liquid` source text.
- AC1: a query returns the templates whose source contains the text and none that do not.
- AC2: each result identifies the template and, where practical, shows where the text occurs. **[PROPOSED]**
- AC3: results are capped at a stated maximum, and the UI says when the results were cut off (limit in D-16).
- AC4: files above the size cap are skipped and the UI says how many were skipped.
- AC5: a search never changes a file or the catalog.
- AC6: results use the file text at request time (FR-006 AC3).
- P: a query for a known string in the fixture.
- N: an unreadable or non-UTF-8 file is skipped without failing the search and without exposing exception text.

### 5.4 View

**TUI-FR-030 View source.** The user can open a template and read its source, read-only.
- AC1: the displayed text equals the file contents (decoded as UTF-8).
- AC2: the source is displayed as inert text. A fixture whose source holds script tags, HTML and Liquid tags shows them as text and nothing executes.
- AC3: no control changes the file.
- P: a packaged template and a user template.
- N: a name not in the catalog gets the generic error. A file over the size cap shows "too large to display" with its size.

**TUI-FR-031 View metadata.** The template view shows its metadata.
- AC1: shows format, logical name, category, origin, file size, and the file path relative to its root (never an absolute host path, SEC-008).
- AC2: for a C-CDA section template, shows the section template id (OID) when it can be read from a literal in the source, labelled best-effort. **[APPROVAL]**
- P: `Section/Goal` shows `2.16.840.1.113883.10.20.22.2.60` in the real package (Verified in earlier work).
- N: a section with no readable id shows "not detected", not an empty or invented value.

### 5.5 Relationships

**TUI-FR-040 Detect literal references in three forms (revised).** The scanner detects literal template references in `include` tags, `batch_render` filters and `evaluate` tags. **[APPROVAL]**
- AC1: `include 'X'` is detected, including the whitespace-control variants of the tag and tags with parameters after the name.
- AC2: in `batch_render: 'X', 'arg'`, the first argument is the template reference. The second argument is a variable name and is not a reference.
- AC3: in `evaluate v using 'X' ...`, the name after `using` is the reference.
- AC4: double-quoted literals are detected if the spec adopts them (none exist in the packaged templates, section 2). **[APPROVAL]** (D-04)
- AC5: each detected reference records its form (include, batch_render or evaluate) and its line number.
- AC6: text inside Liquid comment blocks is not treated as a reference. **[APPROVAL]** (D-04)
- AC7: the scan reads text only. It does not render, compile or evaluate anything (FR-081, FR-083).
- P: the fixture catalog contains one reference of each form and all three are found with the right form.
- N: the second argument of `batch_render` is not reported as a template. A reference inside a comment block is not reported. A name that appears only in a string, not in a supported construct, is not reported.

**TUI-FR-041 Direct references and used-by (revised).** For a template, the UI shows what it directly references and what directly references it, using the effective graph for the selected format. **[APPROVAL]**
- AC1: the "references" list equals the set of literal targets detected in the template's effective source. The "referenced by" list is the exact inverse of the "references" relation across the catalog.
- AC2: each entry shows the form(s) used and one of three target states: found in the catalog, not found in the catalog, or outside the catalog (not a `.liquid` entry, FR-014).
- AC3: duplicates are collapsed to one row per target, with the forms used.
- AC4: when a template has a user override, the effective source is the user template. The packaged version's own references are shown separately and labelled "packaged version". **[APPROVAL]** (D-15)
- AC5: lists are sorted by logical name.
- P: root `CCD` lists its sections and its `evaluate` target. A section lists the root templates that reference it.
- N: a reference to a missing template shows "not found in catalog". It is never dropped.

**TUI-FR-042 Best-effort labelling (revised).** Every relationship view and every flag that depends on the scan carries a fixed note that the result comes from a text scan of three reference forms and is best-effort, never definitive.
- AC1: the note appears on the template relationship view, in the section highlight view, and in any exported text from them (there is no export in Release 1).
- AC2: the UI never says a template is "unused", "dead", "orphaned" or "unreferenced" without "potentially" and the note. It never says "no references". The allowed phrase is "No references detected".
- AC3: a test scans all UI text for the forbidden words (D-18 lists them).
- P: any relationship view.
- N: UI strings containing the forbidden definitive wording fail the test.

**TUI-FR-043 Transitive relationships (new).** The UI shows templates reachable transitively, in both directions, separately from direct references. **[APPROVAL]**
- AC1: for template A, "transitively reachable" lists templates reachable from A by two or more steps that A does not reference directly. "Transitively reaching" lists templates that reach A by two or more steps and do not reference it directly.
- AC2: a template that is both directly and transitively reachable appears only in the direct list.
- AC3: each transitive entry shows at least one shortest chain of logical names that explains it.
- AC4: cycles terminate. Every template appears at most once, and a template is never listed as reachable from itself.
- AC5: the computation is bounded by the size of the catalog and finishes within the NFR-002 limit on the real package.
- AC6: for a C-CDA template, the view shows the root templates that reach it, split into "directly" and "transitively".
- Example from the fixture: Root `CCD` references `Section/Alpha`. `Section/Alpha` references `Entry/Alpha/entry` by `batch_render`. `Entry/Alpha/entry` references `Resource/Alpha`. For `CCD`: direct is `Section/Alpha`, transitive is `Entry/Alpha/entry` and `Resource/Alpha`.
- P: that example, plus a cycle between two utility templates.
- N: a self-reference does not list the template as transitively reachable from itself.

**TUI-FR-044 Four distinct relationship states (new).** The relationship view separates four states, each with its own heading and wording, so they cannot be confused. **[APPROVAL]**
1. **Direct references** (FR-041).
2. **Transitively reachable templates** (FR-043).
3. **No references detected**, shown for a template only when it has zero detected literal references and zero dynamic or unsupported ones.
4. **Dynamic or unsupported references** (FR-045).
- AC1: a template with direct references shows state 1 and does not show state 3.
- AC2: a template with only dynamic references shows state 4 and does not show state 3.
- AC3: states 1, 2 and 4 each show an explicit "none" when empty, except that state 3 is shown only in the case above.
- AC4: for the "referenced by" side, dynamic references cannot be attributed to a target. The UI instead states the catalog-wide count of dynamic and unsupported references and that the "referenced by" list may be incomplete when that count is above zero.
- AC5: each state uses distinct headings and wording, verified by a test on the rendered text.
- P: a fixture template for each state.
- N: a template with a dynamic include and no literals is not described as having no references.

**TUI-FR-045 Dynamic and unsupported references (new).** The scanner reports references it cannot resolve instead of guessing or skipping them. **[APPROVAL]**
- AC1: an `include`, `batch_render` or `evaluate` whose template name is not a quoted literal is listed as a dynamic reference with its form and line number.
- AC2: any other construct that appears to load a template (for example a `render` tag) or a reference construct the scanner cannot read is listed as an unsupported reference with its line number.
- AC3: the UI shows the form and line number, and may show the tag text up to a length limit (D-16). Template text is not patient data, but the limit stops a very long line from flooding the page.
- AC4: the scanner never invents a target for a dynamic reference.
- AC5: the count of dynamic and unsupported references is available per template and for the whole catalog (used by FR-044 AC4 and FR-060 AC3).
- P: fixture templates with `include some_variable`, a `render` tag, and an unterminated reference tag.
- N: none of the three is silently dropped.

### 5.6 Overrides and format selection

**TUI-FR-050 Identify overrides (revised).** The UI identifies packaged templates that a user template overrides. **[APPROVAL]**
- AC1: when a user template has the same logical name as a packaged template of the selected format, both are shown and linked. The user one is marked "overrides packaged" and the packaged one "overridden by user template".
- AC2: both sources can be viewed.
- AC3: a user template with no packaged counterpart in the selected format is shown as "user only".
- AC4: a filter shows only overrides, with the count.
- AC5: matching uses the same mapping as FR-013.
- P: a fixture user `CCD.liquid` overrides packaged `CCD` in the C-CDA view.
- N: a user file with a similar but different name is not flagged.

**TUI-FR-051 User-selected format for comparison (new).** The user selects the format, C-CDA or STU3, in the UI, and that choice decides which packaged templates the user directory is compared with. The format is never inferred. **[APPROVAL]**
- AC1: the UI has an explicit format control. The default is C-CDA. **[PROPOSED]**
- AC2: the same user directory content appears in both format views, and the UI states that user directory templates are not tied to a format.
- AC3: override status, relationships, root usage and section highlights are all computed for the selected format only.
- AC4: a user `CCD.liquid` is "overrides packaged" in the C-CDA view and "user only" in the STU3 view, because STU3 has no `CCD` template (Carried, not re-listed).
- AC5: switching format does not need a restart. The data for both formats is built at startup (FR-006).
- P: that `CCD.liquid` example.
- N: the UI never labels a user directory as "C-CDA" or "STU3" by guessing from its contents.

### 5.7 Unreferenced sections

**TUI-FR-060 Highlight potentially unreferenced C-CDA sections (revised).** The UI highlights C-CDA `Section` templates that no root template reaches, directly or transitively, using the detected references in the effective graph. The result is best-effort and never definitive. **[APPROVAL]**
- AC1: in the fixture catalog, a section that no root reaches by a detected reference is highlighted. A section a root references directly is not. A section referenced only by another section that a root reaches is not.
- AC2: the highlight text reads "Potentially unreferenced (best-effort)" with the FR-042 note.
- AC3: when any root template, or any template reachable from a root, has dynamic or unsupported references, each highlight also shows "analysis may be incomplete" with the count (FR-045 AC5).
- AC4: roots are the top-level C-CDA templates found in the catalog, including `Header` and any top-level user templates (section 4). The list is derived, not hard-coded. **[APPROVAL]** (D-06)
- AC5: the count of highlighted sections is shown and the list can be filtered to them.
- AC6: no such highlight exists for STU3, which has no root and section structure (Carried).
- AC7: the result depends only on template contents. No section name is hard-coded or given special handling (FR-093).
- P: a fixture with `Section/Orphan` highlighted, `Section/OnlyViaSection` not highlighted, and a user-directory root template that references `Section/Orphan` making it not highlighted in the effective graph.
- N: a section referenced only through a dynamic include is highlighted with the incompleteness note, and is not called unreferenced.

### 5.8 Errors

**TUI-FR-070 Generic errors.** Every error shown uses a short fixed message and an error code, never exception text. **[APPROVAL]**
- AC1: for each failure case (unknown format, unknown category, unknown or non-catalog template name including non-`.liquid` names, invalid or traversal-style name, unreadable file, file too large, query too long, user directory problem, missing or wrong request token, rejected Host or Origin), the response holds only the fixed message and code.
- AC2: a test forces an internal exception containing a distinctive marker string and confirms the marker never appears in the page, body, headers or console output.
- AC3: responses never contain a stack trace or an absolute host path.
- AC4: unknown routes and unsupported methods get a generic message.
- P: a request for a non-existent template.
- N: an injected read failure and an injected parse failure.

### 5.9 Read-only and exclusions

**TUI-FR-080 Read-only.** The UI has no way to create, edit, upload, rename or delete a template, and performs no writes.
- AC1: no control and no route accepts content for storage. Only read requests (GET and HEAD) are accepted. **[PROPOSED]**
- AC2: a snapshot (names, sizes, hashes) of the packaged template tree and the user directory is identical before and after a test session that exercises every feature.
- AC3: POST, PUT, PATCH and DELETE get the generic error and change nothing.
- P: a full session leaves both trees unchanged.
- N: each non-read method is rejected.

**TUI-FR-081 No rendering (revised).** The UI does not render a template or run a conversion.
- AC1: no route accepts input data. No render entry point is called during a session, verified by instrumenting the renderer methods in a test.
- AC2: no converted output is shown or stored.
- AC3: the UI may use the loader only to read template text (FR-083 AC2).
- P: any browsing session.
- N: a request that passes a document or asks for a render is rejected with the generic error.

**TUI-FR-082 No patient-data input.** The UI has no upload, paste area or input for clinical documents.
- AC1: no form field or route accepts document content.
- AC2: the only user inputs are format, category, origin filter, a name query, a source query, a template name chosen from the catalog, and the startup directory and port options.
- P and N: covered by FR-080 and FR-081.

**TUI-FR-083 No compilation check (new).** Release 1 has no template compilation or syntax check. **[APPROVAL]**
- AC1: the UI shows no "compiles", "valid" or "passes" status for a template, and no text implies one.
- AC2: during a session, the template compile entry point (`get_template` or equivalent) is never called, verified by instrumentation in a test. Reading text through `read_text` or the loader's source method is allowed because it does not compile (**Assumption**: confirm that in python-liquid 1.12.1 during planning, D-19).
- AC3: a reference shown as "found in catalog" means only that a file with that logical name exists. It does not mean the template loads or renders.
- P: a session with an instrumented compile function records zero calls.
- N: a fixture template with a Liquid syntax error is listed and viewable as text, and is not flagged as invalid.

### 5.10 Constraints expressed as requirements

**TUI-FR-090 No new dependencies.**
- AC1: `pyproject.toml` dependency sections are unchanged. Every import under `fhir_converter/ui/` comes from the Python standard library or an existing declared runtime dependency.
- AC2: the served pages load nothing from another host (no CDN, font service or analytics). A test scans the served assets for external URLs, and the page renders with outbound network blocked. **[PROPOSED]**
- AC3: no frontend framework, bundler or package manager step is used.

**TUI-FR-091 No change to existing behavior.** **[APPROVAL]**
- AC1: conversion output for the existing synthetic fixtures (`TD-CCDA-01` to `12`, excluding `08`, which fails by design) and the two `tests/data/ccda` fixtures is identical before and after the feature, with zero differences.
- AC2: the public API (`render_fhir_string`, `render_fhir`, `render_to_fhir`, `make_environment`, `render_to_dir`, `render_files_to_dir`) keeps its names, signatures and behavior.
- AC3: the existing CLI keeps its arguments, output and exit codes. The TC-CLI cases in `docs/test-cases.md` give the same results as recorded (manual, because `__main__.py` has no unit tests).
- AC4: the existing pytest suite passes with the recorded baseline count, plus the new tests.
- AC5: no existing template, `ValueSet.json`, id-generation input or date rule changes.

**TUI-FR-092 Isolation of new code.**
- AC1: all new application code lives under `fhir_converter/ui/`, and no existing module is edited to support it. **[APPROVAL]**
- AC2: filesystem access for the UI is confined to one module, so the catalog logic can be tested without a running server (discovery B.3).

**TUI-FR-093 Independence from the Goals Section work (new).** The UI does not depend on the separate `001-wire-goals-section` feature. **[APPROVAL]**
- AC1: no UI code, fixture or test assumes that any particular section (Goals, Health Concerns, Instructions or another) is wired or unwired.
- AC2: the UI computes everything from the template files present at startup, so its output changes automatically if the Goals work is later approved and merged.
- AC3: tests that touch the real package assert only things that do not depend on that work (name mapping, OID display, counts that follow the files on disk). They do not assert whether any named section is highlighted.
- AC4: neither feature needs the other to be merged first.
- P: run the UI test suite against a tree with and without a wired `Section/Goal` and observe no test change.
- N: a test that hard-codes "Goals is highlighted" fails review.

**TUI-FR-094 Verify Poetry packaging before any configuration change (new).** Whether the built package includes the UI's static files is verified before deciding that `pyproject.toml` or ignore files must change. **[APPROVAL]** (running a build is not done here)
- AC1: a source distribution and a wheel are built with the project's build tooling and their contents are listed. Every UI asset is present in each. **Not run in this refinement.**
- AC2: if all assets are present, no configuration change is made.
- AC3: if any asset is missing, a written proposal for the smallest configuration change goes to the developers for approval before any edit.
- AC4: the check is repeated after a clean install of the built wheel by running `python -m fhir_converter.ui` (TUI-COMP-006).
- Evidence so far: `pyproject.toml` has `packages = [{include = "fhir_converter"}, ...]`, `MANIFEST.in` is absent, `.gitignore` has no lines for html, js or css (partial search). Whether Poetry honors VCS ignore rules for this is **Assumption**. Poetry was not installed locally when last checked (`CLAUDE.md` section 6), so the build may need an approved install.
- P: all assets present in both artifacts.
- N: an asset missing from the wheel triggers the proposal step, not a silent edit.

---

## 6. Non-functional requirements

| ID | Requirement | Acceptance criterion |
|---|---|---|
| TUI-NFR-001 | **Performance (proposed).** **[APPROVAL]** | Against the real package, the first list for a format returns in under 2 seconds and opening a template in under 1 second, on a normal developer machine. The numbers are mine. Measure and record the actual results |
| TUI-NFR-002 | **Search and relationship cost.** | A source search and a transitive-relationship request over the real package finish within a stated time bound (D-16) and do not block other requests indefinitely |
| TUI-NFR-003 | **Accessibility (design aim, not a conformance claim).** **[APPROVAL]** | Keyboard operation for every function, visible focus, semantic headings, labelled controls, a contrast ratio chosen in the spec. The four relationship states (FR-044) are distinguishable by text, not by color alone |
| TUI-NFR-004 | **Browser support.** **[APPROVAL]** | Current versions of Chrome, Edge, Firefox and Safari. No legacy browsers |
| TUI-NFR-005 | **Determinism.** | Lists, search results, relationship lists and chains are sorted and stable for the same input |
| TUI-NFR-006 | **Startup catalog (revised).** | The catalog is built once at startup. The page states the build time and that a restart refreshes it (FR-006) |
| TUI-NFR-007 | **Code standards.** | New code is fully type-annotated and passes mypy with the project settings. flake8 passes (max line length 120, ignore `E203,E721`, max complexity 10). Docstrings use `Args:`, `Returns:`, `Raises:`. No `logging` and no new output channel beyond FR-007 |
| TUI-NFR-008 | **Testability.** | The catalog and relationship logic is unit tested without a running server. The server layer is tested separately. New tests follow `tests/test_loaders.py` |
| TUI-NFR-009 | **Documentation.** | A README or docs page for the UI, a `CHANGES.md` entry, and `docs/test-cases.md` statuses set only after cases are run |
| TUI-NFR-010 | **Resource limits.** | Maximum served file size, search result count, query length, search or graph time bound, request timeout and displayed tag-text length are defined and enforced (D-16) |
| TUI-NFR-011 | **Portability.** | Same behavior on Windows, Linux and macOS, including path handling |
| TUI-NFR-012 | **Honest wording.** | UI text and docs make no claim that templates or output are FHIR-valid or conformant, and no HIPAA, SOC 2, HITRUST or ISO claim (`CLAUDE.md` section 8) |
| TUI-NFR-013 | **Startup build time (new, proposed).** **[APPROVAL]** | Building the catalog and relationships for the real package takes under 5 seconds on a normal developer machine. The number is mine. Measure and record it |
| TUI-NFR-014 | **Scanner accuracy statement (new).** | The docs state the scan is a text scan of three reference forms, list what it does not detect, and say results are best-effort |

---

## 7. Security and privacy acceptance criteria

| ID | Criterion | Test approach |
|---|---|---|
| TUI-SEC-001 | Server binds to the IPv4 loopback address only (FR-002) | Inspect the bound address |
| TUI-SEC-002 | **Host and Origin validation (revised).** The `Host` header must be an allowed value: the bound loopback address with the chosen port, and optionally `localhost` with that port **[APPROVAL]** (D-10). When an `Origin` header is present it must match the server's own origin. Anything else gets the generic rejection. This also covers DNS rebinding | Requests with a foreign Host, a foreign Origin, a different port, and the allowed values |
| TUI-SEC-003 | No path is built from request input. Template access goes through an allowlist built from catalog enumeration | Code review and the traversal tests |
| TUI-SEC-004 | Traversal rejected. Names containing `..`, absolute paths, drive letters, backslashes or NUL get the generic error (discovery B.5) | Table of malicious names, plus URL-encoded and double-encoded forms **[PROPOSED]** |
| TUI-SEC-005 | User directory containment. It is resolved once at startup, must exist and be a directory, and a symlink that leaves it is refused | Fixture with a symlink to an outside file. Skipped, and reported, where the platform cannot create symlinks |
| TUI-SEC-006 | **Only `.liquid` files are readable (revised).** Any other extension is refused. Larger files than the cap are not returned (D-16) | Requests for `.json`, `.txt` and an oversized `.liquid` fixture |
| TUI-SEC-007 | No exception text in responses (FR-070) | Marker-string injection |
| TUI-SEC-008 | No absolute host paths in responses. Paths are relative to the template root **[PROPOSED]** | Search every response in a session for the fixture's absolute temp path |
| TUI-SEC-009 | Read-only enforced (FR-080) | Tree hash before and after, rejected methods |
| TUI-SEC-010 | Templates are code and are never executed. Source is shown as text only and is never rendered, imported or compiled (FR-081, FR-083) | Inert-source fixture and instrumented render and compile calls |
| TUI-SEC-011 | Output encoding. All template-derived text in a page, including logical names, tag text from FR-045 and search snippets, is escaped | Fixture template with HTML, script and attribute-breaking characters in its name and body |
| TUI-SEC-012 | No patient data is handled. No documents in, no converted output out. All test data is synthetic and nothing from `data/sample` is used (constitution III) | Review of fixtures and routes |
| TUI-SEC-013 | No persistence of activity. No files, caches to disk, history or stored queries | Tree snapshot, review |
| TUI-SEC-014 | **Console output (revised).** Default per-request logging is suppressed (FR-007). Query strings, template names and the request token never reach the console. **[APPROVAL]** | Capture standard output and error during a full session |
| TUI-SEC-015 | No secrets, identifiers or addresses in fixtures, docs or code (`CLAUDE.md` section 8) | Review and search |
| TUI-SEC-016 | No compliance claims. This feature and its docs are engineering descriptions, not an attestation of any control | Text review |
| TUI-SEC-017 | The pre-existing exception-content issue is not worsened. The UI calls no render path (decisions Q10). Opening a separate tracking item is an **[APPROVAL]** | Review |
| TUI-SEC-018 | **Per-run request protection token (new).** **[APPROVAL]** The server creates an unpredictable value from a cryptographically secure standard-library source at startup. Every data request must carry it, and a missing or wrong value gets the generic rejection. The value differs on every start and is never written to a file or logged. The way it reaches the browser (for example in the address shown at startup) is a plan-stage choice | Requests with no token, a wrong token and the right token. Two runs give different tokens. The token is absent from files and from console output other than the startup message |
| TUI-SEC-019 | The startup message exposes the token only on the local console. This is an accepted local exposure to be confirmed **[APPROVAL]** (D-11) | Review |

SEC-002 and SEC-018 are separate controls. Host and Origin checks stop other web pages and rebinding. The token stops other local processes and other users on the same machine from calling the server without being given the address. Neither is authentication of a person. No claim is made that either meets any compliance control.

---

## 8. Compatibility (Python 3.8 to 3.12)

| ID | Requirement | Acceptance criterion |
|---|---|---|
| TUI-COMP-001 | Everything under `fhir_converter/ui/`, including its `__main__`, works on Python 3.8 to 3.12. The existing CLI module uses 3.11 syntax (`StrEnum`) and is not a model (`CLAUDE.md` section 4) | pytest, flake8 and mypy pass on the full CI matrix |
| TUI-COMP-002 | No syntax or library features newer than 3.8 in `fhir_converter/ui/`. Examples to avoid: built-in generics such as `list[str]` in runtime-evaluated annotations, `X or Y` union syntax with the pipe operator, `match` statements, `str.removeprefix` and `removesuffix`, `functools.cache`, parenthesized multi-item `with` statements. Use `typing` equivalents | A 3.8 run of the full suite, and an import of every UI module |
| TUI-COMP-003 | Only the standard library and already declared runtime dependencies are imported | Import review and an import test in a clean environment |
| TUI-COMP-004 | The standard-library server pieces used (server class, handler, random token source) exist and behave the same on 3.8 to 3.12. The exact choices belong in the plan | CI matrix with a startup-and-request test on each version |
| TUI-COMP-005 | Same behavior on Windows, Linux and macOS, including path separators, case sensitivity and symlink availability. Logical names always use forward slashes | CI on all three, and a test using Windows-style separators in requested names |
| TUI-COMP-006 | `python -m fhir_converter.ui` works from a source checkout and from an installed package. Static asset inclusion is verified by FR-094 before any configuration change is considered | The FR-094 build and install check |
| TUI-COMP-007 | Adding the UI changes neither the import time nor the behavior of `import fhir_converter` | Module inspection test (FR-001 AC3) |
| TUI-COMP-008 | Automatic port selection (FR-005) works on all three platforms and five Python versions | CI test that starts the server with no port option and connects to the reported port |

---

## 9. Test scenarios

All data is synthetic. **Fixture catalog** means a temporary tree of invented `.liquid` files created inside the test. Suggested fixture contents (invented names):

| Fixture template | Purpose |
|---|---|
| `CCD` (root) | `include 'Section/Alpha'`, `evaluate x using 'Utils/Id'` |
| `Header` (root) | `include 'Section/Beta'` |
| `Section/Alpha` | `batch_render: 'Entry/Alpha/entry'`, `include 'Section/OnlyViaSection'`, `include 'Entry/DoesNotExist'` |
| `Entry/Alpha/entry` | `include 'Resource/Alpha'` |
| `Resource/Alpha` | `include 'DataType/Code'` |
| `Section/Beta` | referenced only from `Header` |
| `Section/OnlyViaSection` | referenced only by `Section/Alpha` |
| `Section/Orphan` | referenced by nothing |
| `Section/Dyn` | `include some_variable`, a `render` tag, an unterminated tag, a reference inside a comment block |
| `Utils/CycleA`, `Utils/CycleB` | reference each other |
| `Utils/SelfRef` | references itself |
| A template whose source holds script, HTML and Liquid text, and one with a deliberate Liquid syntax error | Inert display, no compile check |
| `.json`, `.xsl` and `.txt` files in package and user trees | `.liquid`-only scope |
| User tree: `CCD.liquid` (override), `Section/UserOnly`, `Custom.liquid` (a top-level user root referencing `Section/Orphan`) | Overrides and effective graph |

| # | Scenario | Covers |
|---|---|---|
| T01 | Start with packaged templates only. List C-CDA and STU3 and compare counts with `.liquid` files on disk (no hard-coded numbers) | FR-001, FR-010 |
| T02 | Category filter returns only that category. Counts sum to the total. An empty category is handled | FR-011 |
| T03 | Every template has an origin. With no user directory, none is "user" | FR-012 |
| T04 | Logical-name mapping for root and nested names matches loader behavior without using the private method | FR-013 |
| T05 | Name search: match, no match, empty query, case difference, overlong query | FR-020 |
| T06 | Source search: match, none, cap reached and reported, oversized file skipped and counted, unreadable and non-UTF-8 files skipped | FR-021 |
| T07 | View source matches the file. The script, HTML and Liquid fixture is shown inert and escaped | FR-030, SEC-010, SEC-011 |
| T08 | Metadata fields present. Section OID shown when a literal exists, "not detected" otherwise | FR-031 |
| T09 | Direct references: one of each form found with the right form and line number. Second `batch_render` argument not reported. Comment-block reference not reported. Missing target shown as "not found in catalog". A reference to a `.json` name shown as "outside catalog" | FR-040, FR-041 |
| T10 | Used-by is the exact inverse of references across the fixture | FR-041 |
| T11 | Transitive view: `CCD` has direct `Section/Alpha`, transitive `Entry/Alpha/entry` and `Resource/Alpha`, with chains shown. Direct targets are not repeated as transitive | FR-043 |
| T12 | Cycles and the self-reference terminate and list each template once, never as reachable from itself | FR-043 |
| T13 | The four states appear with distinct headings. A template with only a dynamic include shows state 4 and not "No references detected" | FR-044, FR-045 |
| T14 | Dynamic include, `render` tag and unterminated tag are each listed with form and line, and none is dropped | FR-045 |
| T15 | The "referenced by" view shows the catalog-wide dynamic count and the incompleteness note when that count is above zero | FR-044 |
| T16 | Override: user `CCD.liquid` is "overrides packaged" in C-CDA view and "user only" in STU3 view. Both sources viewable. Override count and filter work | FR-050, FR-051 |
| T17 | Effective graph: with the user `CCD.liquid` override, `CCD` relationships come from the user source, and the packaged version's references are shown separately | FR-041 |
| T18 | Unreferenced highlight: `Section/Orphan` highlighted, `Section/Beta` and `Section/OnlyViaSection` not. With the user root `Custom` present, `Section/Orphan` is not highlighted. No highlight in the STU3 view | FR-060 |
| T19 | Highlight text says "potentially" and carries the best-effort note. A section reached only through a dynamic include is highlighted with the incompleteness note | FR-042, FR-060 |
| T20 | A scan of all UI text finds none of the forbidden definitive words | FR-042 AC3 |
| T21 | Each error case returns the fixed message and code. A marker string in an injected exception never appears in the page, body, headers or console | FR-070, SEC-007 |
| T22 | Non-read methods and unknown routes are rejected. A refresh route does not exist. Tree snapshot is unchanged after a full session | FR-006, FR-080, SEC-009 |
| T23 | No route accepts input data. Renderer methods and the compile entry point are never called (instrumented). A syntax-error fixture is listed and not flagged | FR-081, FR-082, FR-083 |
| T24 | Traversal names (plain, encoded, backslash, drive letter, NUL, absolute) rejected with no file read | SEC-003, SEC-004 |
| T25 | `.json`, `.xsl` and `.txt` files in both trees are not listed, searchable or viewable. Requests for them are rejected | FR-014, SEC-006 |
| T26 | Symlink escaping the user directory is refused (where supported, skip reported otherwise) | SEC-005 |
| T27 | Real-package spot checks that do not depend on the Goals work: `Section/Goal` maps to `Section/_Goal.liquid` and shows its OID. Counts follow the files on disk. No assertion about which sections are highlighted | FR-013, FR-031, FR-093 |
| T28 | Existing CLI cases (TC-CLI-05, TC-CLI-11 and the rest) rerun manually and compared with recorded results | FR-091 AC3 |
| T29 | Bound address is `127.0.0.1` and no option changes it | FR-002, SEC-001 |
| T30 | Host and Origin: foreign Host, foreign Origin, wrong port and the allowed values behave as required | SEC-002 |
| T31 | Token: no token, wrong token and right token. Two runs give different tokens. The token is not in any file and not in console output beyond the startup message | SEC-018, SEC-014 |
| T32 | Two instances started together get different automatic ports. Each works | FR-005, COMP-008 |
| T33 | Bad user directory (missing, file, unreadable) fails at startup with a non-zero exit and fixed message and no open listener | FR-004 |
| T34 | The same UI test suite gives identical results on a tree with and without a wired `Section/Goal`, and no test names a specific real section as highlighted | FR-093 |
| T35 | Captured standard output and error across a full session equal the startup message only. No request lines. Error and rejected requests print nothing | FR-007 |
| T36 | A file added to the user directory after startup is absent until restart. A removed file still listed shows the generic error when opened. The page shows the build time and the restart note | FR-006 |
| T37 | Served assets contain no external URLs. The page works with outbound network blocked | FR-090 |
| T38 | Conversion output before and after is identical for the named fixtures. Public API signatures are unchanged. Existing suite count is unchanged | FR-091 |
| T39 | `import fhir_converter` does not load the UI package | FR-001, COMP-007 |
| T40 | Full CI matrix (3.8 to 3.12, three platforms) runs pytest, flake8 and mypy over the new code | COMP-001 to COMP-005 |
| T41 | Packaging check: build the sdist and wheel, list contents, confirm every UI asset, then install the wheel and start the UI. Needs approval to run. **Not run here** | FR-094, COMP-006 |
| T42 | Keyboard-only walk through list, search, view, relationships, overrides and highlights | NFR-003 |
| T43 | Timing of list, view, search, transitive view and startup build against the real package, recorded | NFR-001, NFR-002, NFR-013 |

Not planned: any test that renders a document through the UI, and any test that asserts FHIR validity (no validator runs in this project).

---

## 10. Out of scope (first release)

- Creating, editing, uploading, duplicating, renaming or deleting templates, packaged or user.
- Rendering a template, running a conversion, previewing output, or accepting any clinical document.
- A template compilation or syntax check, and any status that implies a template is valid.
- Cataloguing, searching or viewing `.json`, `.xsl` or any non-`.liquid` file, including `ValueSet.json` and `metadata.json`, unless explicitly approved later.
- Detecting references in forms other than `include`, `batch_render` and `evaluate`. Others are reported as unsupported, not resolved.
- Authentication of people, accounts, multi-user access, remote or network access, and binding to any address but IPv4 loopback.
- A catalog refresh while the server runs.
- Persistence, history, favourites or saved searches.
- A frontend framework, bundler, web framework, ASGI server or any new dependency.
- A new console script, a change to `pyproject.toml` scripts, or any change to the existing CLI.
- Any configuration change before FR-094 shows one is needed, and then only with approval.
- Fixing or depending on the unwired Goals, Health Concerns or Instructions sections, or the `001-wire-goals-section` work.
- Fixing the pre-existing exception-content behavior (separate item).
- Any FHIR validation or conformance statement.
- Exposing or changing `TemplateSystemLoader._resolve_template_name` or any public API.

---

## 11. Assumptions

| # | Assumption | Basis |
|---|---|---|
| A1 | The developers will review and may change everything here. Nothing is approved | Instruction |
| A2 | Users are developers and integrators, so raw logical names, line numbers and Liquid source are acceptable to show | Decisions Q2, not user-researched |
| A3 | A text scan of the three literal forms covers the packaged templates, because every reference found there is a single-quoted literal | Section 2 (Verified, narrow patterns) |
| A4 | User templates may use forms the packaged ones do not, so dynamic and unsupported handling is still needed | Section 2 |
| A5 | The catalog, metadata and graph for both formats are built once at startup and held in memory | Recommendation R11. Memory size is unmeasured |
| A6 | Source text is read from disk on request. Reading through the loader does not compile the template | Not confirmed for python-liquid 1.12.1 (D-19) |
| A7 | A user directory is not tied to a format, so the same files appear in both format views | `--template-dir` has no format marker (Carried) |
| A8 | The effective graph (user over packaged) is the right basis for relationship views | Follows the loader's search order. Needs approval (D-15) |
| A9 | The standard-library server suffices for one local user with a few concurrent requests, and its default logging can be suppressed without changing the library's public surface | Not tested |
| A10 | Template source is not patient data, so showing it carries lower privacy risk than showing converted output | Discovery D.4 |
| A11 | Python 3.8 rules apply to `fhir_converter/ui/` because it is inside the package | `CLAUDE.md` section 4 |
| A12 | `.liquid` counts of 819 and 264 were correct at discovery and may change, so tests must not hard-code them | Discovery |
| A13 | The proposed performance, accessibility and browser targets are placeholders | Decisions Q11 |
| A14 | `Header` behaves as a root because every document root includes it | Carried |

---

## 12. Decision register

Every item is **[APPROVAL]** and none is approved. "Proposed answer" is the working assumption used in this document.

| ID | Decision needed | Proposed answer used here | Status |
|---|---|---|---|
| D-01 | Confirm the read-only scope and the standard-library, `fhir_converter/ui/`, `python -m fhir_converter.ui` design | As proposed in `02` and `03` | Proposed, not approved |
| D-02 | Confirm the `.liquid`-only catalog (R5), and whether `.json` files are ever added later | `.liquid` only in Release 1 | Proposed, not approved |
| D-03 | Confirm no compile check in Release 1 (R6) | Excluded | Proposed, not approved |
| D-04 | Reference scan details: double-quoted literals, handling of comment blocks and raw blocks, whitespace-control variants | Quoted literals in single quotes required. Double quotes, comment-block exclusion and whitespace variants adopted as proposed | Proposed, not approved |
| D-05 | Direct versus transitive and the four-state wording (R2, R3) | As in FR-043 and FR-044 | Proposed, not approved |
| D-06 | Definition of root templates, including `Header` and user top-level templates | As in section 4 | Proposed, not approved |
| D-07 | Wording rules for best-effort labelling and the forbidden words list (D-18) | As in FR-042 | Proposed, not approved |
| D-08 | Final category taxonomy for C-CDA and STU3, and behavior for an unknown category | C-CDA list in FR-011. STU3 not decided | Open |
| D-09 | Handling of a user-directory file in a sub-folder without the `_` prefix | Consistent with the loader | Open |
| D-10 | Allowed Host values: loopback address with port only, or also `localhost` with port | Both, with the chosen port | Proposed, not approved |
| D-11 | Per-run token, how it reaches the browser, and acceptance of the token appearing in the console startup message | Token required. Transport decided in the plan | Proposed, not approved |
| D-12 | Name and form of the user directory startup option | Not decided | Open |
| D-13 | Allow an explicit port option, or automatic only | Automatic by default, explicit option optional | Proposed, not approved |
| D-14 | Re-implement the name-mapping rule instead of exposing the private loader method | Re-implement and test against the loader | Proposed, not approved |
| D-15 | Effective graph for overridden templates, and display of the packaged version separately | As in FR-041 AC4 | Proposed, not approved |
| D-16 | Concrete limits: maximum file size, search result count, query length, time bounds, request timeout, tag-text display length | Not decided | Open |
| D-17 | Read source from disk at request time while metadata and relationships stay at startup state | As in FR-006 | Proposed, not approved |
| D-18 | Forbidden definitive UI words: "unused", "dead", "orphan", "orphaned", "unreferenced" without "potentially", "no references" | As listed | Proposed, not approved |
| D-19 | Verify in python-liquid 1.12.1 that reading template text through the loader does not compile it | To be verified in planning | Open |
| D-20 | Suppress per-request console logging (R8), and permit the single startup message | Suppress. One startup message | Proposed, not approved |
| D-21 | Automatic port as the default (R10) | As in FR-005 | Proposed, not approved |
| D-22 | Packaging verification before any configuration change (R13), including approval to build and to install Poetry if absent | As in FR-094 | Proposed, not approved |
| D-23 | Independence from the Goals work (R12) | As in FR-093 | Proposed, not approved |
| D-24 | Performance, startup, accessibility and browser targets | Placeholders in NFR-001, NFR-003, NFR-004, NFR-013 | Open |
| D-25 | Single HTML page with plain JavaScript, no framework | As proposed in `02` Q3 | Proposed, not approved |
| D-26 | Ratify the constitution (v1.0.0 draft) before the spec cites it as final | Cited as a draft until then | Open |
| D-27 | Baseline pytest run before implementation, and approval to install flake8, mypy or Poetry if absent | Baseline first | Open |
| D-28 | Open a separate item for the exception-content issue | Separate item | Proposed, not approved |
| D-29 | Whether the new `fhir_converter/ui/` code is excluded from coverage or included | Included. Not discussed in discovery | Open |

---

## 13. Traceability

### 13.1 Original 17 scope items

| Scope item | Requirements |
|---|---|
| 1 Read-only local browser UI | FR-080, FR-081, FR-082, FR-083, FR-002, SEC-009 |
| 2 Standard-library HTTP server | FR-090, COMP-004 |
| 3 Code under `fhir_converter/ui/` | FR-092, COMP-001 |
| 4 Launch with `python -m fhir_converter.ui` | FR-001, FR-005 |
| 5 Packaged C-CDA and STU3 | FR-010, FR-011, FR-014 |
| 6 Optional user directory | FR-003, FR-004, FR-012, FR-051 |
| 7 List by format and category | FR-010, FR-011 |
| 8 Search by name | FR-020 |
| 9 Search within source | FR-021 |
| 10 View source and metadata | FR-030, FR-031 |
| 11 Includes and used-by | FR-040, FR-041, FR-042, FR-043, FR-044, FR-045 |
| 12 Overrides | FR-050, FR-051 |
| 13 Unreferenced C-CDA sections | FR-060, FR-042 |
| 14 Generic errors | FR-070, FR-007, SEC-007 |
| 15 No editing, deletion, rendering or preview | FR-080, FR-081, FR-082, FR-083 |
| 16 No new dependencies | FR-090, COMP-003 |
| 17 No change to conversion, public API or CLI | FR-091, FR-092, FR-093, COMP-007 |

### 13.2 The fourteen refinement recommendations

| Ref | Requirements | Tests |
|---|---|---|
| R1 three reference forms | FR-040, FR-045 | T09, T14 |
| R2 direct and transitive views | FR-041, FR-043 | T10, T11, T12 |
| R3 four distinct states | FR-044, FR-045 | T13, T14, T15 |
| R4 best-effort, never definitive | FR-042, FR-060, NFR-014 | T18, T19, T20 |
| R5 `.liquid` only | FR-014, FR-010, FR-013, FR-021, SEC-006 | T01, T25 |
| R6 no compilation check | FR-083, FR-081, SEC-010 | T23 |
| R7 user selects format | FR-051, FR-050, FR-003 | T16, T17 |
| R8 suppress request logging | FR-007, SEC-014 | T35 |
| R9 loopback, Host and Origin, local protection | FR-002, SEC-001, SEC-002, SEC-018, SEC-019 | T29, T30, T31 |
| R10 automatic port | FR-005, COMP-008 | T32 |
| R11 catalog at startup, restart to refresh | FR-006, NFR-006, NFR-013 | T36, T43 |
| R12 independent of Goals work | FR-093, FR-060 AC7 | T27, T34 |
| R13 verify Poetry packaging first | FR-094, COMP-006 | T41 |
| R14 Python 3.8 to 3.12, existing behavior unchanged | COMP-001 to COMP-008, FR-091 | T28, T38, T39, T40 |

### 13.3 Contradictions resolved

X1 to X15 in section 1.2 map to: X1 FR-014. X2 FR-083. X3 FR-081, FR-083. X4 FR-005, FR-007. X5 FR-006. X6 FR-060. X7 FR-040, FR-041. X8 FR-003, FR-051. X9 FR-002, SEC-002. X10 FR-042, FR-044. X11 FR-005. X12 FR-093, T27, T34. X13 section 4, FR-060. X14 FR-041, FR-050. X15 SEC-002, SEC-018.

## Next step

Wait for developer review, especially the decision register (section 12). After the decisions are confirmed or changed, `/speckit-specify` can be run with this document as input. Nothing further is created until then.
