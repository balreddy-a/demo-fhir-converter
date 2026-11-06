# Data Model: Goals Section to Goal

Mapping view only. Source paths are from `Resource/_Goal.liquid` as it exists today. No new field mappings are added by this feature.

## Source: Goal entry (C-CDA)

Located at `section(2.16.840.1.113883.10.20.22.2.60)/entry/observation`.

| Source path | Used for |
|---|---|
| `observation.id` | `Goal.identifier` |
| `observation.code` | `Goal.description`, and `target.measure` when there is no translation |
| `observation.code.translation` | `target.measure` when present |
| `observation.statusCode.code` | `Goal.lifecycleStatus` (guarded, see rules) |
| `observation.effectiveTime.value` | `target.dueDate` |
| `observation.value.low.value` and `.unit` | `target.detailQuantity` |
| `observation.entryRelationship.observation.value` (first) | `Goal.priority` |

## Target: Goal (FHIR R4)

| Element | R4 cardinality | Source | Rule |
|---|---|---|---|
| `id` | 0..1 | hash of the entry observation (unchanged expression) | Must stay stable |
| `identifier` | 0..* | `observation.id` | Optional |
| `lifecycleStatus` | **1..1** | `statusCode.code` | Exact member of the R4 set, else the entry is skipped |
| `description` | **1..1** | `observation.code` | Spec gap D4: missing code would be invalid |
| `subject` | **1..1** | patient reference | Always present when the Goal is emitted |
| `priority` | 0..1 | nested observation value | Omit if absent |
| `target.measure` | 0..1 | translation or code | Omit empty structures (FR-008) |
| `target.detailQuantity` | 0..1 | `value.low` | Only when a value exists |
| `target.dueDate` | 0..1 | `effectiveTime.value` | Omit if absent |

## Validation and state rules

| Condition | Result |
|---|---|
| Status code is exactly an R4 `GoalLifecycleStatus` code | Goal emitted (Resource and Subject reference together) |
| Status missing, empty, or not in the set | Entry skipped entirely. No stub, no default, no mapping |
| Entry has no `observation` | Skipped (no status) |
| Optional parts missing | Omitted from the Goal |
| Render-time failure | Whole document fails today. Skipped only if D3-B is approved |

## Relationships

- Goal.subject refers to the Patient entry produced by the document header, through `fullPatientId`.
- The Resource entry and the Subject-reference entry share one id and are merged into one bundle entry by `parse_fhir`.
