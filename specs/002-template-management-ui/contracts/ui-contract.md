# Contract: Template Management UI interfaces

Design contract for the planned interfaces. Nothing exists yet. Items tagged **[APPROVAL]** are not decided.

## Command line

`python -m fhir_converter.ui [options]` (FR-001).

| Option | Status |
|---|---|
| User template directory | Name **TBD** (D-12). Working name `--template-dir` mirrors the existing CLI meaning and is not decided. Optional. Must exist, be a directory and be readable (FR-004) |
| Port | **Not exposed** until D-13 is approved. Default is an automatically selected free port (FR-005) |
| Bind address | No option. IPv4 loopback only (FR-002) |

Exit codes: 0 on normal interrupt. Non-zero with a fixed message for a bad user directory or, if an explicit port is approved, a busy port. No exception text (FR-004, FR-005).

Console output: one startup message with the address, port and the token-bearing address, then nothing else during a session (FR-007, **[APPROVAL]** D-11, D-20).

No change to `python -m fhir_converter` or the `fhir_converter_cli` script (FR-091).

## HTTP

Host and port: `127.0.0.1:<port>`. Methods: GET and HEAD only. All other methods get the generic rejection. No route accepts a request body.

### Required checks on every request

| Check | Applies to | Failure |
|---|---|---|
| `Host` is `127.0.0.1:<port>` or `localhost:<port>` (D-10) | All | Generic rejection |
| `Origin`, if present, equals the server's own origin | All | Generic rejection |
| Per-run token in a custom request header | `/api/` routes only, proposed (D-11) | Generic rejection |

### Static routes

| Path | Content |
|---|---|
| `/` | `static/index.html` |
| `/static/app.js` | `static/app.js` |
| `/static/style.css` | `static/style.css` |

Any other path under `/static/` is not found. No directory listing.

### API routes (token required)

Names, formats and filters are query values, never path segments. A name is looked up in the catalog and never turned into a path (SEC-003). `format` is `CCDA` or `STU3`.

| Route | Query | Returns |
|---|---|---|
| `GET /api/meta` | none | Build time, formats, counts, category counts, override count, whether a user directory is configured, skipped-file count, fixed notes (best-effort, restart, user directory not tied to a format) |
| `GET /api/templates` | `format`, optional `category`, `origin`, `q`, `overrides`, `unreferenced` | Sorted list: logical name, category, origin, size, override flag, section flag |
| `GET /api/template` | `format`, `name`, optional `origin` | Metadata and the relationship view with the four states, transitive chains, `packaged_version` when overridden |
| `GET /api/source` | `format`, `name`, optional `origin` | Source text read at request time, or "too large to display" with the size |
| `GET /api/search/source` | `format`, `q` | Matching templates, skipped count, cut-off flag |
| `GET /api/sections/unreferenced` | `format` (C-CDA only) | Highlighted sections with incompleteness note and counts |

No route renders, compiles, accepts documents, writes, refreshes or deletes.

### Response format

JSON, UTF-8. Security headers (**[PROPOSED]**, N-04): same-origin `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `Cache-Control: no-store`, `Referrer-Policy: no-referrer`. No CORS headers.

### Errors (FR-070)

Body: `{"error": {"code": "<CODE>", "message": "<fixed text>"}}`. Never exception text, stack traces, absolute paths or the unrecognised input value.

| Code (provisional) | Used for |
|---|---|
| `BAD_REQUEST` | Unknown format, unknown category or origin, invalid or traversal-style name, query too long |
| `NOT_FOUND` | Name not in catalog, non-`.liquid` name, unknown route |
| `UNAVAILABLE` | File unreadable or removed since startup, file too large |
| `FORBIDDEN` | Wrong Host, wrong Origin, missing or wrong token |
| `METHOD_NOT_ALLOWED` | Any method other than GET and HEAD |

Exact code names and HTTP status mapping are plan-level choices and may change.

## Unchanged interfaces (FR-091)

`render_fhir_string`, `render_fhir`, `render_to_fhir`, `make_environment`, `render_to_dir`, `render_files_to_dir`, the CLI arguments and exit codes, all templates, ids, the date rule. A regression test records the function signatures.

## Non-claims

The interface displays template text and relationships. It makes no statement that templates or converter output are valid or conformant, and no compliance claim.
