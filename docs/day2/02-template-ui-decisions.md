# Decisions: Template UI First Release (Q1 to Q13)

Status: **PROPOSED, awaiting developer review.** Nothing here is approved. This is a requirements clarification exercise. No `spec.md`, `plan.md` or `tasks.md` was created for this feature, and no application code, template, test, dependency or configuration file was changed. No command was run.

Audience: the developers who own `python-fhir-converter` (v0.3.0).

Date: 2026-11-05.

Inputs: `docs/day2/01-template-ui-discovery.md`, `CLAUDE.md`, `.specify/memory/constitution.md` (v1.0.0, an unratified draft), and the ten scope recommendations given with this task. Where a fact comes from the discovery report, it keeps that report's tag (Verified, Carried, Assumption). Anything new in this document that I could not confirm is marked as uncertain.

This is an engineering proposal. It makes no claim that any output is FHIR-valid and no compliance claim. It is not a substitute for review by the developers.

## Proposed first-release scope

**In scope**
- A local, read-only web page served from Python's standard library.
- Packaged templates for both C-CDA and STU3, plus an optional user template directory.
- List, search, view source, and inspect a template (kind, size, resolved name, includes, used-by).
- Mark which templates come from the package and which come from a user directory, and mark user templates that override a packaged one.
- Show relationships and root-template usage where they can be detected reliably, labelled as best-effort.
- Highlight C-CDA section templates that no root template includes.

**Out of scope**
- Creating, editing, uploading or deleting templates.
- Rendering a document, uploading an input file, or showing converted output.
- Any change to conversion behavior, ids, public API, CLI options, exit codes or dependencies.
- Authentication, remote access, and any frontend or backend framework.
- Fixing the unwired Goals, Health Concerns or Instructions sections (separate work, see Q9).

## Summary table

| Q | Topic | Proposed decision | Developer approval needed | Uncertainty |
|---|---|---|---|---|
| Q1 | Meaning of "managing" | Read-only: list, search, view, inspect | Yes | Low |
| Q2 | Users | Developers and integrators | Yes (confirm) | Medium |
| Q3 | Surface | Local browser page, standard library | Yes | Medium |
| Q4 | Code placement | New isolated subpackage inside `fhir_converter` | Yes | Medium |
| Q5 | Entry point | `python -m` on the new subpackage, no `pyproject.toml` or CLI change | Yes | Low |
| Q6 | Templates in scope | Packaged (both formats) plus optional user directory, all read-only | Yes | Medium |
| Q7 | Render preview | Excluded. Compile check only if verified | Yes | Medium |
| Q8 | Override flagging | Yes, by logical name | Covered by scope approval | Medium |
| Q9 | Unwired sections | Flag unreferenced C-CDA sections, best-effort | Covered by scope approval | Medium |
| Q10 | Exception-content issue | Track separately. UI shows generic errors only | Yes | Low |
| Q11 | Performance, accessibility, browsers | Proposed targets, to be confirmed | Yes | High |
| Q12 | Baseline run | Run pytest first. Ask before installing flake8 or mypy | Yes | Low |
| Q13 | Constitution | Developers ratify or amend before the spec is finalized | Yes | Low |

I cannot give approval on the developers' behalf. "Covered by scope approval" means the decision follows directly from the Q1 approval and does not need its own sign-off.

---

## Q1. What does "managing" mean for the first release?

**Proposed decision.** Read-only operations only: list, search, view source, and inspect. No create, edit, upload, duplicate or delete.

**Reasoning.** The discovery report found that a write path changes conversion output for every consumer, that the template cache has `auto_reload` off, and that the project has no validator to catch a bad edit (discovery sections 3, 9 and K1, K2). Read-only delivers the missing visibility (gaps G1 to G5) with no effect on conversion, which also satisfies the rule against changing existing behavior. "Inspect" is defined here as: kind, size, resolved logical name, origin, includes, used-by, and for C-CDA sections the template id (OID) they handle.

**Developer approval required.** Yes. The scope is a recommendation until a developer confirms it.

**Remaining uncertainty.** Low. One open edge: whether a "does this template load" check counts as inspect. It reads and parses but does not render. See Q7.

---

## Q2. Who are the users?

**Proposed decision.** Developers of this library and application integrators. No requirement to serve analysts or non-technical users.

**Reasoning.** The project is a library and CLI whose consumers write pipelines. A technical audience lets the UI show template names, paths and Liquid source as they are, with no wizards or simplified wording. It also supports keeping the UI small.

**Developer approval required.** Yes, to confirm.

**Remaining uncertainty.** Medium. No user research exists in the repository, so this is inferred from the project's purpose. If non-technical users are expected, wording and accessibility needs change.

---

## Q3. Which UI surface?

**Proposed decision.** A local browser interface: a small HTTP server from the Python standard library, serving static HTML, CSS and plain JavaScript, with a small JSON API behind it. No framework, no build step, no package manager for the page, and nothing loaded from a CDN.

**Reasoning.** This is option B in the discovery report, and it matches your recommendations 3 and 4. It adds no dependency (constitution V, `CLAUDE.md` section 7) and needs no change to how the library is built. A static report (option C) or CLI sub-commands (option A) are cheaper but are not the browser interface requested. The server must bind to `127.0.0.1` only, build no file path from request input, and check `Host` and `Origin` (constitution technical constraints, discovery B.5 and B.7).

**Developer approval required.** Yes. It adds a new shipped surface to a project that has none.

**Remaining uncertainty.** Medium.
- Plain JavaScript is not a framework, but it adds a new file type to the repository. Confirm that is acceptable.
- Server behavior on Python 3.8 through 3.12 and on Windows, Linux and macOS is not tested. CI covers all three.
- Whether `http.server` is adequate for concurrent requests from one local browser tab is an assumption.

---

## Q4. Where does the UI code live?

**Proposed decision.** A new, isolated subpackage inside the package, for example `fhir_converter/ui/`. It is not imported by any existing module, and no existing module is edited to support it. All of it stays Python 3.8 compatible and fully type-annotated.

**Reasoning.** The target users include integrators, who normally install the package and do not have the source tree. A top-level folder outside the package would not reach them. The package already ships non-Python files (1,083 `.liquid` files), so shipping static assets has a precedent. Keeping it isolated and unimported means the library's public API and import behavior are unchanged (recommendation 10).

**Alternative.** A top-level folder (not shipped). It keeps the wheel and the 3.8 rule out of scope for UI code, but integrators installed from a package index could not use it. I would pick this only if the developers do not want the UI shipped to every user.

**Developer approval required.** Yes. Placement decides what ships to every library user.

**Remaining uncertainty.** Medium.
- That Poetry includes the new static files in the built package is not confirmed. The existing `.liquid` files suggest it works, but I did not check the build configuration.
- The effect on wheel size is unmeasured.
- It needs a decision on whether the ignore rules or packaging settings need attention, which would count as configuration change and needs approval.

---

## Q5. Is a new console script or CLI change acceptable?

**Proposed decision.** No new console script, no change to `pyproject.toml` scripts, no change to `fhir_converter/__main__.py`. The server starts with `python -m fhir_converter.ui`, using a new `__main__` module inside the new subpackage. An optional user template directory is passed as a startup argument of that new module, with the same meaning as `--template-dir`.

**Reasoning.** `python -m` on a subpackage needs no entry in `pyproject.toml`, so it avoids the approval-gated configuration change and leaves the existing CLI, its arguments and exit codes untouched. The existing CLI has no unit tests (`CLAUDE.md` section 6), so not touching it avoids a regression risk.

**Developer approval required.** Yes, to confirm that no script entry is wanted now.

**Remaining uncertainty.** Low. A console script could be added later as a separate approved change.

---

## Q6. Which templates are in scope?

**Proposed decision.** Packaged C-CDA templates, packaged STU3 templates, and templates from one optional user directory. Everything is read-only, including the packaged ones and the user directory.

**Reasoning.** This matches recommendations 5 and 6. The user directory is the only supported customization mechanism (`--template-dir`), so a developer reviewing their overrides is a core use case.

**Design notes.**
- The existing `--template-dir` takes any folder and carries no format marker, and the example folder `data/templates/ccda` suggests people organize by format themselves. The UI therefore cannot tell from the folder which format a user file targets. Proposed handling: the user chooses the format (C-CDA or STU3) when comparing, and override detection runs against that format.
- Name mapping must follow the loader rule (`Section/_Goal.liquid` is `Section/Goal`). The existing mapping function is private. Importing a private method would couple the UI to it, and exposing it would change the public API. Proposed handling: reimplement the small rule in the new catalog and add a test that checks it against the loader's public behavior.

**Developer approval required.** Yes, specifically for showing the contents of the user directory. Custom templates may hold organization-specific logic. The page is local only.

**Remaining uncertainty.** Medium.
- The user-directory format question above has no verified answer.
- Behavior with symlinks, very large files and non-UTF-8 files is not tested.
- Whether python-liquid's file loader refuses names outside its search path is an unverified assumption, so the catalog should not rely on it (discovery B.5).

---

## Q7. Is a render preview wanted?

**Proposed decision.** No. The first release has no conversion endpoint, no file upload, no input paste box, and no display of converted output. A "does this template load and compile" check is optional and is included only if it is first verified to read and parse the template without rendering anything.

**Reasoning.** This matches recommendation 9. Converter output embeds the whole source document, and a failed render has been seen to include partial output in its error message (discovery B.6). Excluding rendering removes patient-data handling from the UI altogether, which is the largest privacy risk identified. The compile check involves no patient data.

**Developer approval required.** Yes, to confirm both the exclusion and whether the optional compile check is wanted.

**Remaining uncertainty.** Medium. Whether python-liquid parses on load, and so whether `get_template` catches syntax errors but not a missing include, is an **Assumption** from the discovery report. It must be checked before a "passes" result is shown to users, otherwise the UI would imply more than it proves. If it cannot be verified, the compile check is dropped from release 1.

---

## Q8. Should overrides from the user directory be shown and flagged?

**Proposed decision.** Yes. For each template, show its origin (package or user). When a user template has the same logical name as a packaged template of the selected format, mark it "overrides packaged" and offer both sources for viewing.

**Reasoning.** An override silently changes conversion, because the user directory is searched first (`__main__.py:86-94`, discovery G5). Making that visible is a core part of recommendation 6 and costs nothing at runtime.

**Developer approval required.** Covered by the Q1 and Q6 approvals.

**Remaining uncertainty.** Medium.
- A user file can also override a template that a packaged template includes. The page can show it by name, but it cannot show which conversion would use it without rendering.
- The format ambiguity from Q6 applies.

---

## Q9. Should unwired sections be flagged?

**Proposed decision.** Yes, for C-CDA. Flag any `Section/` template that no root template includes, using the include relationships that can be read from the template text, and label it "best-effort, based on text search". For STU3, show used-by relationships but do not show an "unreferenced by a root template" flag, because STU3 has one template per resource type and no root and section structure.

**Reasoning.** This matches recommendations 7 and 8. The discovery report found three unwired sections (Goals, Health Concerns, Instructions) by searching the root templates, and none of them was visible without that manual search. Reading includes from text is reliable only for the form actually seen in the repository, so the label matters.

**Relationship to other work.** The `001-wire-goals-section` feature, which is still awaiting developer approval, would wire Goals into `CCD.liquid`. The UI computes the result from whatever the templates contain, so it does not depend on that work and must not assume either state.

**Developer approval required.** Covered by the Q1 approval.

**Remaining uncertainty.** Medium.
- The text search assumes every include is a single-quoted literal. Discovery K9 notes this is an **Assumption**. A dynamic or differently quoted include would be missed, producing a false "unreferenced" flag.
- Whether the C-CDA root templates reference sections in any other way (for example a different tag) is not fully checked.
- STU3 structure was not re-read in this discovery.
- "Reliably detectable" should be agreed as: literal includes only, with anything else reported as "not detected".

---

## Q10. How is the exception-content issue handled?

**Proposed decision.** Track it as its own work item, outside this feature, and do not fix it here. The UI in release 1 never renders, so it does not trigger the problem. Any error the UI shows (unknown name, unreadable directory, load failure) uses a short fixed message with a code, and never displays `str(exception)`.

**Reasoning.** The issue is pre-existing: a decode failure was seen to carry partial output into the exception text, which the CLI prints (discovery B.6 and K5). It conflicts with `CLAUDE.md` section 5 ("never put patient data into exception messages") but fixing it changes existing error behavior, which is out of scope under recommendation 10. A rule of "generic messages only" in the UI keeps this feature from adding to it.

**Developer approval required.** Yes, to open a separate item and to confirm the "generic message only" rule.

**Remaining uncertainty.** Low for the UI. The scope of the underlying issue (which error paths carry data) is not mapped.

---

## Q11. Performance, accessibility and browser expectations?

**Proposed decision.** The repository states none of these, so these are proposals for the developers to accept or change:
- **Performance.** Listing all 1,083 packaged templates and opening any one should feel immediate on a normal developer machine. Proposed target: first list shown in under 2 seconds. This number is mine, not from the repository.
- **Accessibility.** Semantic HTML, full keyboard operation, visible focus, readable contrast, text alternatives for any icon. This is a design aim. It is not a claim of conformance to any standard.
- **Browsers.** Current versions of Chrome, Edge, Firefox and Safari. No support for legacy browsers.

**Reasoning.** Without targets the later spec cannot have measurable success criteria. Developer and integrator users on a local page do not need heavy targets, so the proposals are light.

**Developer approval required.** Yes, because the numbers and the browser list are proposed, not derived.

**Remaining uncertainty.** High. No measurement has been taken, and the 2-second figure has no basis in the repository. Catalog caching and the fact that the template cache has `auto_reload` off (edits are not seen until restart) both affect what the page shows and should be stated in the UI.

---

## Q12. May a fresh baseline be run?

**Proposed decision.** Yes for the test suite, before any implementation begins:
`.\.venv\Scripts\python.exe -m pytest -q`, recording the real passed count (expected 475, unverified). Ask first before installing flake8, mypy or Poetry, because the local environment may not have them and installing tools should not be done silently.

**Reasoning.** The constitution and `CLAUDE.md` require a measured before-and-after, and the earlier discoveries did not run the suite. Running pytest does not change code or configuration. Installing tools is a different category, even though they are declared development dependencies. CI runs flake8 and mypy, so they can serve as the check if they are not installed locally.

**Developer approval required.** Yes. I did not run anything for this task, per the instruction not to run implementation commands.

**Remaining uncertainty.** Low. Note that the working tree already shows modified files under `data/sample`, which this work does not touch, so a clean-tree assumption should not be made.

---

## Q13. Is the constitution ready to ratify?

**Proposed decision.** Developers ratify it, or amend it, before the feature specification is finalized. Until then, the spec cites it as "v1.0.0 draft". I have no authority to ratify it and have not changed it.

**Reasoning.** The file still contains a temporary "SYNC IMPACT REPORT" comment to remove before commit, and its ratification date reads `TODO(RATIFICATION_DATE)`. Its rules were derived from `CLAUDE.md` with no new principles, so there is little risk of surprise, but a spec should not cite a draft as if it were final. One related gap for review: the constitution has a technical-constraints bullet for any future UI or server (loopback only, no paths from request input) and no principle about read-only scope. That bullet may be enough, or the developers may want a sentence added.

**Developer approval required.** Yes, only the developers can ratify.

**Remaining uncertainty.** Low. Who ratifies, and when, is not stated in the repository.

---

## Items that need a developer call before the spec

1. Confirm the read-only scope (Q1) and the Q3 surface, since everything else depends on them.
2. Choose placement (Q4). The proposal ships the UI to all library users. The alternative does not reach integrators.
3. Decide how to handle the format of a user template directory (Q6), because the existing mechanism does not record it.
4. Decide whether the optional compile check stays (Q7), after verification.
5. Accept or change the proposed performance, accessibility and browser targets (Q11).
6. Open a separate item for the exception-content issue (Q10).

## Facts this document depends on that are not yet verified

- That Poetry packages new static files under `fhir_converter/ui/` (Q4).
- That python-liquid parses templates on load (Q7).
- That every include in the C-CDA root templates is a single-quoted literal (Q9).
- That the standard-library server behaves correctly on Python 3.8 to 3.12 on all three CI platforms (Q3).
- That `FileExtensionLoader` does not read outside its search path (Q6). The design should not depend on it.

## Next step

Wait for developer review of this document. After the decisions are confirmed or changed, a feature specification can be written. Nothing further is created until then.
