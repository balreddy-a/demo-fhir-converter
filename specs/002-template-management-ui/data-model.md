# Data Model: Template Management UI

In-memory model only. Nothing is persisted. Frozen dataclasses (Python 3.7+), built once at startup, never mutated. Field names are provisional and may change in implementation. Spec references are in brackets.

## TemplateEntry (one `.liquid` file)

| Field | Type | Notes |
|---|---|---|
| `logical_name` | str | Forward slashes, loader rule applied [FR-013] |
| `origin` | "package" or "user" | [FR-012] |
| `format` | "CCDA", "STU3" or none | Packaged entries have a format. User entries have none and are projected into both views [FR-051] |
| `category` | str | Derived from location. Rule provisional [FR-011, D-08] |
| `size` | int | Bytes at startup [FR-031] |
| `relative_path` | str | Relative to its root. Never absolute [SEC-008] |
| `absolute_path` | Path | Held internally only, never returned [SEC-003] |
| `section_id` | str or none | Section OID when a literal is found, else none [FR-031 AC2, **[APPROVAL]**] |
| `scan` | ScanResult | Result of scanning this file at startup [FR-040] |

## Reference (inside ScanResult)

| Field | Type | Notes |
|---|---|---|
| `kind` | "literal", "dynamic" or "unsupported" | [FR-045] |
| `form` | "include", "batch_render", "evaluate" or "other" | [FR-040] |
| `target` | str or none | Logical name for literal references only |
| `line` | int | 1-based |
| `text` | str | Tag text, length-capped for display [FR-045 AC3, D-16] |

## ScanResult

`references: tuple[Reference, ...]`, plus derived counts: `literal_count`, `dynamic_count`, `unsupported_count`. The scan never records source values beyond tag text.

## FormatView (one per format, built at startup)

| Field | Notes |
|---|---|
| `entries` | Effective entries by logical name (user wins over packaged) [FR-041] |
| `packaged` | Packaged entries by logical name, including overridden ones [FR-050] |
| `user` | User entries by logical name, same for both views [FR-051] |
| `overrides` | Logical names where user and packaged both exist [FR-050] |
| `roots` | Top-level effective names, C-CDA only [FR-060, D-06] |
| `forward`, `reverse` | Adjacency of found literal targets, with forms [FR-041] |
| `unmapped_user_files` | Count only, provisional [N-02] |

## RelationshipView (computed per request)

| Field | Notes |
|---|---|
| `direct` | Targets with forms and target state: found, not found, outside catalog [FR-041] |
| `transitive` | Reachable minus direct minus self, each with a shortest chain [FR-043] |
| `dynamic_or_unsupported` | Items with form and line [FR-045] |
| `no_references_detected` | True only if zero literals and zero dynamic or unsupported [FR-044 AC1, AC2] |
| `referenced_by`, `reached_from` | Same shape on the reverse side, plus `catalog_dynamic_count` and `incomplete` [FR-044 AC4] |
| `packaged_version` | Present when the template is overridden [FR-041 AC4] |
| `root_reach` | Roots that reach it, split direct and transitive [FR-043 AC6] |

## SectionFlag

`name`, `potentially_unreferenced: bool`, `analysis_incomplete: bool`, `dynamic_count: int`. C-CDA only [FR-060].

## Catalog metadata (for `meta`)

`built_at`, `formats`, per-format counts, per-category counts, `user_directory_configured: bool`, `override_count`, `unmapped_user_files`, and the fixed best-effort and restart notes.

## Validation rules

- A name is accepted only if it is a key in the catalog for the requested format and origin [SEC-003].
- Format must be one of the allowed values. Category and origin must be in their fixed sets. Query length is limited [D-16].
- Files that are not `.liquid`, are oversized, unreadable, not UTF-8 or escape the user root are not served [FR-014, SEC-005, SEC-006].
- Cycles and self references never produce duplicates or self-reachability [FR-043 AC4].

## State

The catalog is built once and has no refresh transition. A restart is the only way to change it [FR-006].
