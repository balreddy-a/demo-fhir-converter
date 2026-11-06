# Research: Template Management UI

Date: 2026-11-05. Read-only inspection only. No command that runs project code, builds or installs was executed. Items marked **Open** are investigations in [plan.md](plan.md) section 9.1 and are not resolved here.

## R-1: HTTP server

- **Decision**: `http.server.ThreadingHTTPServer` with a `BaseHTTPRequestHandler` subclass, bound to `127.0.0.1`, port 0 by default.
- **Rationale**: standard library only (no dependency). Catalog and graph are immutable after startup, so threads need no locks.
- **Alternatives**: single-threaded `HTTPServer` (a slow request blocks the page), a web framework (new dependency, rejected by the spec), `socketserver` directly (more code for no gain).
- **Open**: identical behavior on 3.8 to 3.12 and on three platforms (I-4).

## R-2: Locating and reading templates

- **Decision**: read `.liquid` files directly from disk. Packaged folders are `<fhir_converter package dir>/templates/ccda` and `/stu3`. The user folder is the one named at startup.
- **Rationale**: one filesystem module (FR-092), no dependence on python-liquid loader internals at runtime, no accidental compile. `fhir_converter/templates` has no `__init__.py` (Verified), so `importlib` package resources are not the natural route.
- **Alternatives**: read through `TemplateSystemLoader` and `read_text` (couples to `ChoiceLoader`, depends on D-19), `importlib_resources` (declared but not imported by the package, would add a first use), `PackageLoader` listing (it cannot list, Verified in discovery).
- **Risk**: a zipped or frozen install would break direct reads (N-05).
- **Loader still used in tests** to cross-check the name mapping (FR-013 AC2).

## R-3: Does loader source-reading compile? (D-19)

- **Preliminary, read-only**: in python-liquid 1.12.1, `BaseLoader.load` calls `get_source` then `env.from_string` (parse), and `Environment.get_template` calls `loader.load` (`liquid/builtin/loaders/base_loader.py:126-148`, `liquid/environment.py:342-366`). So `get_source` returns text without parsing.
- **Open**: the `ChoiceLoader` and `TemplateSystemLoader` `get_source` path and a runtime test with a syntax-error template (I-2). D-19 stays open until then.

## R-4: Reference scanner approach

- **Decision**: regular expressions over template text, in small functions, after blanking comment blocks with newlines kept.
- **Rationale**: FR-083 forbids compiling. A Liquid tokenizer needs parsing, and python-liquid internals are fragile (`CLAUDE.md` section 4). Evidence: all 619 `include`, 188 `batch_render` and 24 `evaluate` references in the packaged templates are single-quoted literals, with no `render` tag and no dynamic names (Verified, narrow patterns).
- **Alternatives**: python-liquid lexer or `parse` (compile), Python `ast`-style full parser (large, not justified).
- **Known limit**: a text scan, reported as best-effort (FR-042). Calibration against the real files is I-5.

## R-5: Relationship algorithms

- **Decision**: per-format effective graph, precomputed adjacency, transitive sets by breadth-first search with a parent map, root reach by multi-source search.
- **Rationale**: about 800 nodes per format, so on-request computation is cheap and cycles are handled by a visited set.
- **Alternatives**: precompute all-pairs transitive closure (memory and no benefit at this size).

## R-6: Token transport (D-11, **[APPROVAL]**)

- **Proposal**: token in the URL fragment of the startup address, read by the page script and sent in a custom header on every API call.
- **Rationale**: a fragment is not sent in the request line and not in `Referer`. A custom header blocks simple cross-site requests. Host and Origin checks remain.
- **Alternatives**: token in the query string (appears in logs and referrers), a cookie (needs cookie handling and CSRF reasoning), no token (spec requires one).

## R-7: Packaging (D-22)

- **Evidence so far**: `packages = [{include = "fhir_converter"}, ...]`, no `MANIFEST.in`, `.gitignore` has no html, js or css rule (partial search, Verified). Poetry was not installed locally when last checked (`CLAUDE.md` section 6).
- **Open**: build and inspect (I-1). Planned in a temporary copy of the repository so the working tree is untouched. Needs approval to install tooling and build.

## R-8: Python 3.8 to 3.12

- **Decision**: conservative 3.8 subset, `typing` generics, no `X | Y` unions, no `match`, no `removeprefix`, no `functools.cache`. A test parses every UI module with `ast` and `feature_version=(3, 8)`.
- **Fact**: the local `.venv` is Python 3.14.10, so local runs cannot prove 3.8 (Verified). CI is the authority.
- **Note**: existing library modules use `from __future__ import annotations` (seen in `loaders.py`), which defers annotation evaluation. UI code should not depend on that for runtime-evaluated constructs.

## R-9: Static page

- **Decision**: three files, served from an explicit allowlist, plain JavaScript using `textContent`. A test scans them for `innerHTML`, `eval`, `document.write` and external URLs.
- **Alternatives**: server-rendered HTML (more escaping surface in Python), a framework (rejected by the spec).

## R-10: Console output (D-20, **[APPROVAL]**)

- **Fact**: `BaseHTTPRequestHandler` logs each request to standard error by default through `log_message`. Plan: override `log_message` and `log_error` to do nothing. Only the startup message and fatal startup errors print.
- **Open**: confirmation that the startup message, which carries the token, is acceptable (D-11, SEC-019).

## R-11: Constitution tension (N-01)

- Principle I limits Python changes to loaders, renderers, parsers, filters, tags and CLI. A new `ui` package is additive and read-only but not in that list. The constitution is an unratified draft, so the wording can be clarified. **Open**, needs a developer ruling.
