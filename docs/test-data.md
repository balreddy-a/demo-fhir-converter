# Test Data

All data is **synthetic**. Names, ids and addresses are invented ("Alex Example", "SYN-0001", "Exampleville", NPI `9999999999`). Files are in `data\synthetic\`. Do not add real patient data to this folder.

The pre-existing `data\sample` folder is not used here: its provenance is **UNKNOWN**.

## 1. Fixture inventory

### 1.1 C-CDA (`data\synthetic\ccda\`, run with `--template-name CCD --source-format CCDA`)

All share one base document (document id `TD-nn`, patient id `SYN-0001`, one Allergies section) and differ in one respect.

| ID | File | Variation | Verified result (exploratory run) |
|---|---|---|---|
| TD-CCDA-01 | `TD-CCDA-01_valid_basic.ccda` | Complete baseline. Document time `20240115103000-0500` | Bundle with Composition, Organization, Practitioner, Patient, DocumentReference. Patient female, birthDate 1980-01-01 |
| TD-CCDA-02 | `TD-CCDA-02_no_birthtime.ccda` | No `birthTime` | Patient present, no `birthDate` |
| TD-CCDA-03 | `TD-CCDA-03_no_gender.ccda` | No `administrativeGenderCode` | Patient present, no `gender` |
| TD-CCDA-04 | `TD-CCDA-04_unmapped_gender.ccda` | Gender code `ZZ` | Patient present, no `gender` |
| TD-CCDA-05 | `TD-CCDA-05_date_no_timezone.ccda` | Document time `20240115103000` (no zone) | `Composition.date` = `2024-01-15` |
| TD-CCDA-06 | `TD-CCDA-06_date_day_only.ccda` | Document time `20240115` | `Composition.date` = `2024-01-15` |
| TD-CCDA-07 | `TD-CCDA-07_date_year_only.ccda` | Document time `2024` | `Composition.date` = `2024` |
| TD-CCDA-08 | `TD-CCDA-08_date_malformed.ccda` | Document time `not-a-date` | Conversion fails: `Malformed HL7 datetime not-a-date` |
| TD-CCDA-09 | `TD-CCDA-09_no_recordtarget.ccda` | No `recordTarget` element | No Patient entry. Other entries present |
| TD-CCDA-10 | `TD-CCDA-10_no_sections.ccda` | Empty `structuredBody` | Succeeds, no Composition sections |
| TD-CCDA-11 | `TD-CCDA-11_nullflavor_name.ccda` | Patient `<name nullFlavor="UNK"/>` | Patient present, no `name` |
| TD-CCDA-12 | `TD-CCDA-12_no_address.ccda` | No `addr` | Patient present, no `address` |

### 1.2 STU3 (`data\synthetic\stu3\`, run with `--source-format STU3`)

| ID | File | Template | Variation | Verified result |
|---|---|---|---|---|
| TD-STU3-01 | `TD-STU3-01_patient_valid.json` | `Patient` | Complete | Output equals input fields |
| TD-STU3-02 | `TD-STU3-02_patient_minimal.json` | `Patient` | Only `resourceType`, `id` | `{"resourceType":"Patient","id":"syn-p2"}` |
| TD-STU3-03 | `TD-STU3-03_patient_empty_fields.json` | `Patient` | `name: []`, `gender: ""`, `birthDate: null` | Empty fields removed: only `resourceType`, `id` |
| TD-STU3-04 | `TD-STU3-04_no_resourcetype.json` | `Patient` | No `resourceType` | Passes through as `{"id":"syn-p4","gender":"male"}`, no error |
| TD-STU3-05 | `TD-STU3-05_immunization.json` | `Immunization` | `date` with time zone, `wasNotGiven` | `occurrenceDateTime` `2024-01-15T10:30:00.000-05:00`, `wasNotGiven` kept |

### 1.3 Invalid input (`data\synthetic\invalid\`)

| ID | File | Used with | Verified result |
|---|---|---|---|
| TD-INV-01 | `TD-INV-01_malformed.xml` | CCDA | `XMLSyntaxError` tag mismatch |
| TD-INV-02 | `TD-INV-02_empty.xml` | CCDA | `XMLSyntaxError: Document is empty` |
| TD-INV-03 | `TD-INV-03_not_cda.xml` | CCDA | No error. Bundle with empty references |
| TD-INV-04 | `TD-INV-04_malformed.json` | STU3 | `Json5EOF` unclosed object |
| TD-INV-05 | `TD-INV-05_empty.json` | STU3 | `Json5EOF: No JSON data found` |
| TD-INV-06 | `TD-INV-06_ignore.txt` | CCDA via `--from-dir` | Skipped by the extension filter. Direct render fails with `XMLSyntaxError` |

### 1.4 Custom template (`data\synthetic\templates\`)

| ID | File | Purpose | Verified result |
|---|---|---|---|
| TD-TPL-01 | `Mini.liquid` | One-line user template for `--template-dir` | With TD-CCDA-01 gives `{"resourceType":"Bundle","type":"collection","entry":[{"resource":{"resourceType":"Basic","id":"TD-01"}}]}` |

### 1.5 Inline data (no file needed)

| ID | Value | Use |
|---|---|---|
| TD-DTM-01 | `2024`, `202401`, `20240115`, `20240115103000`, `20240115103000-0500`, `20240115103000+0530`, `20240115103000.123` | `hl7_to_fhir_dtm` |
| TD-DTM-02 | `abc`, empty string, `2024011` | `hl7_to_fhir_dtm` error cases |
| TD-MAP-01 | `F` / `ValueSet/Gender`, `ZZ` / `ValueSet/Gender`, `HP` / `ValueSet/AddressUse`, `ZZ` / `ValueSet/AddressUse` | `get_property` |

Verified `hl7_to_fhir_dtm` results: `2024` gives `2024`; `202401` gives `2024-01`; `20240115` gives `2024-01-15`; `20240115103000` gives `2024-01-15` (time dropped, no zone); `20240115103000-0500` gives `2024-01-15T10:30:00-05:00`; `20240115103000+0530` gives `2024-01-15T10:30:00+05:30`; `20240115103000.123` gives `2024-01-15`. The values `abc`, empty and `2024011` raise `ValueError: Malformed HL7 datetime`.

Verified `get_property`: `F` gives `female`; `HP` gives `home`; unmapped `ZZ` gives an empty string for both `code` and `system`.

## 2. Baseline input (TD-CCDA-01)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ClinicalDocument xmlns="urn:hl7-org:v3" xmlns:sdtc="urn:hl7-org:sdtc">
  <realmCode code="US"/>
  <typeId root="2.16.840.1.113883.1.3" extension="POCD_HD000040"/>
  <templateId root="2.16.840.1.113883.10.20.22.1.2"/>
  <id root="2.16.840.1.113883.19.5.99999.1" extension="TD-01"/>
  <code code="34133-9" codeSystem="2.16.840.1.113883.6.1" displayName="Summarization of Episode Note"/>
  <title>Synthetic Test Document</title>
  <effectiveTime value="20240115103000-0500"/>
  <confidentialityCode code="N" codeSystem="2.16.840.1.113883.5.25"/>
  <languageCode code="en-US"/>
  <recordTarget>
    <patientRole>
      <id root="2.16.840.1.113883.19.5" extension="SYN-0001"/>
      <addr use="HP"><streetAddressLine>1 Example Street</streetAddressLine><city>Exampleville</city>
        <state>CA</state><postalCode>90000</postalCode><country>US</country></addr>
      <patient>
        <name use="L"><given>Alex</given><family>Example</family></name>
        <administrativeGenderCode code="F" codeSystem="2.16.840.1.113883.5.1"/>
        <birthTime value="19800101"/>
      </patient>
    </patientRole>
  </recordTarget>
  <author>
    <time value="20240115103000-0500"/>
    <assignedAuthor>
      <id root="2.16.840.1.113883.4.6" extension="9999999999"/>
      <assignedPerson><name><given>Sam</given><family>Synthetic</family></name></assignedPerson>
    </assignedAuthor>
  </author>
  <custodian><assignedCustodian><representedCustodianOrganization>
    <id root="2.16.840.1.113883.19.5"/><name>Synthetic Test Clinic</name>
  </representedCustodianOrganization></assignedCustodian></custodian>
  <component><structuredBody>
    <component><section>
      <templateId root="2.16.840.1.113883.10.20.22.2.6.1"/>
      <code code="48765-2" codeSystem="2.16.840.1.113883.6.1" displayName="Allergies"/>
      <title>Allergies</title><text>No known allergies</text>
    </section></component>
  </structuredBody></component>
</ClinicalDocument>
```

## 3. Baseline output (TD-CCDA-01, abbreviated)

Entry order and key fields from a verified run. `DocumentReference` is omitted because it embeds a copy of the source document as base64 zlib-compressed text.

```json
{
  "resourceType": "Bundle",
  "type": "batch",
  "entry": [
    {"resource": {"resourceType": "Composition", "id": "1a00a129-950a-c7da-e67f-6b7459abf925",
      "identifier": {"use": "official", "value": "2.16.840.1.113883.19.5.99999.1"},
      "status": "final",
      "type": {"coding": [{"code": "34133-9", "display": "Summarization of Episode Note", "system": "http://loinc.org"}]},
      "date": "2024-01-15T10:30:00-05:00",
      "title": "Synthetic Test Document",
      "confidentiality": "N",
      "attester": [{"party": {"reference": "Practitioner/"}}],
      "section": [{"title": "Allergies",
        "text": {"status": "generated", "div": "<div xmlns=\"http://www.w3.org/1999/xhtml\">Allergies</div>"},
        "code": {"coding": [{"code": "48765-2", "display": "Allergies", "system": "http://loinc.org"}]}}],
      "subject": {"reference": "Patient/05cadb76-f5ab-1d6a-fae5-8539f75dd453"},
      "custodian": {"reference": "Organization/5ceefb31-278f-dd6c-2fe0-e4074026d4c9"},
      "author": [{"reference": "Practitioner/3b0d80c2-5119-2eb6-f9fc-7001e9a76be6"}]},
     "request": {"method": "PUT", "url": "Composition/1a00a129-950a-c7da-e67f-6b7459abf925"}},
    {"resource": {"resourceType": "Organization", "id": "5ceefb31-278f-dd6c-2fe0-e4074026d4c9",
      "identifier": [{"system": "urn:ietf:rfc:3986", "value": "urn:uuid:2.16.840.1.113883.19.5"}],
      "name": "Synthetic Test Clinic"}},
    {"resource": {"resourceType": "Practitioner", "id": "3b0d80c2-5119-2eb6-f9fc-7001e9a76be6",
      "identifier": [{"system": "http://hl7.org/fhir/sid/us-npi", "value": "9999999999"}],
      "name": [{"family": "Synthetic", "given": ["Sam"]}]}},
    {"resource": {"resourceType": "Patient", "id": "05cadb76-f5ab-1d6a-fae5-8539f75dd453",
      "identifier": [{"system": "urn:oid:2.16.840.1.113883.19.5", "value": "SYN-0001"}],
      "name": [{"use": "usual", "family": "Example", "given": ["Alex"]}],
      "birthDate": "1980-01-01", "gender": "female",
      "address": [{"use": "home", "line": ["1 Example Street"], "city": "Exampleville",
                   "state": "CA", "country": "US", "postalCode": "90000"}]}}
  ]
}
```

`meta.profile` and per-entry `request` blocks are present on the real output and trimmed here. IDs are deterministic for the same input. `DocumentReference.date` is the current time and varies per run.

## 4. STU3 input and output pairs

TD-STU3-01 (`Patient`):

```json
{"resourceType":"Patient","id":"syn-p1","active":true,
 "name":[{"use":"official","family":"Example","given":["Alex"]}],
 "gender":"female","birthDate":"1980-01-01"}
```

Output is the same content.

TD-STU3-03 (`Patient`) input `{"resourceType":"Patient","id":"syn-p3","name":[],"gender":"","birthDate":null}` gives `{"resourceType":"Patient","id":"syn-p3"}`.

TD-STU3-05 (`Immunization`) is shown in full in [api-guide.md](api-guide.md).

## 5. Data maintenance

- Fixtures are static files. To add a case, copy `TD-CCDA-01_valid_basic.ccda`, change one element, and name it `TD-CCDA-nn_<variation>.ccda`.
- Keep fixtures free of real names, MRNs, dates of birth, addresses and NPIs.
- Regenerate expected values by re-running the case; do not hand-edit them.
- Test output is written to a temp folder and should not be committed.
