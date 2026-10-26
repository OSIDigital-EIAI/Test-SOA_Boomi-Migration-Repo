# ProcessINTReadFileABCSImpl (REST CreateEmployee)

Test-OracleSOA-Project · Listener · medium complexity

[← All interfaces](../../Extraction_Report.md)

- [Interface Overview](#interface-overview)
- [Interface Summary](#interface-summary)
- [Functional Analysis](#functional-analysis)
- [Technical Analysis](#technical-analysis)
- [Mappings](#mappings)
- [Business Rules](#business-rules)
- [Flow Diagram](#flow-diagram)
- [Data Model](#data-model)
- [Architecture](#architecture)

---

## Interface Overview

REST entry point (POST /create, JSON Employees) of the Jobvite intake composite. It converts the Jobvite JSON into the INT employee record, immediately replies "Employee details received successfully" with the first candidate ID, then runs the same per-hire validation and creation as int-file-intake and, after a three-minute wait per hire, forwards the original request to dat-profile-generation-rest.

| Field | Value |
| --- | --- |
| Interface | ProcessINTReadFileABCSImpl (REST CreateEmployee) |
| Type | Listener |
| Complexity | medium |
| Components to build | 2 |
| Mapping rules | 4 |
| Business rules | 1 |

## Interface Summary

This is the web-service door into the hiring process used by the Jobvite file intake. Each call carries one or more hires in Jobvite JSON. The service converts them to the standard TaskUs new-hire record, answers straight away that the details were received, and then processes each hire in the same way as the file intake: validation, the Philippine SSS duplicate check and creation in HCM. After each hire it waits three minutes and passes the original request to the DAT profile generation service, which handles salary, talent profile and allowances.  Because the reply is sent before any checks, the Jobvite side always sees success; problems are only emailed to the HRIS lists. In Boomi this becomes a Web Services Server listener that maps the JSON and calls the validation sub-process built for int-file-intake. Since the only known caller is another TaskUs integration, it should call that sub-process directly instead of making an HTTP call to itself.

**Complexity:** medium (score 36)

## Functional Analysis

1. **Receive Jobvite JSON** — POST /create (pick onMessage CreateEmployee).
2. **Map to INT record** — xform\_rest\_to\_inputvar.
3. **Acknowledge** — Reply candidateID and "Employee details received successfully".
4. **Validate and create each hire** — Shared forEach as in int-file-intake.
5. **Follow-up** — Wait 3 minutes, POST the original request to dat-profile-generation-rest.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | REST / JSON over HTTP (caller uses http://129.144.156.139:9073/soa-infra/resources/default/ProcessINTReadFileABCSImpl!2.0/CreateEmployee/create) | /create (service CreateEmployee) | HTTP Basic (OWSM wss\_http\_token\_service\_policy) |
| outbound | REST over HTTPS (composite) | https://129.144.156.28:443/soa-infra/resources/default/ProcessDatProfileGeneration!2.0/JobVite\_Rest | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Jobvite Employees JSON to INT record | Employee/\* (about 75 scalar fields) | Employees/Employee/\* same names | looped | each Employee | copy | low | — |
| MR-002 | Jobvite Employees JSON to INT record | Employee/PhoneNumber | PhoneNumber | conditional | contains "+" | translate(PhoneNumber, "+", "") | low | — |
| MR-003 | Jobvite Employees JSON to INT record | EducationDetails\[\] / PreviousEmploymentDetails\[\] | same nested groups | looped | — | copy | low | — |
| MR-004 | Jobvite Employees JSON to INT record | Response candidateID / status | Employee\[1\]/ID | hardcoded | — | status "Employee details received successfully" | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Early acknowledgement | The caller is told the details were received before any validation. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **POST /create received** — Jobvite JSON
2. **Map to INT record**
3. **Reply received** — Before processing
4. **Validate and create hires** — Shared sub-process (int-file-intake)
5. **Wait 3 min, DAT profile** — Per hire
6. **Done**

## Data Model

_No profile components owned here: the request uses the INT record layout of int-file-intake (ReadFileService.xsd in JSON form) and the response is the same small ack as emp-int-rest-intake._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| REST binding /create + OWSM | Web Services Server listener with JSON profile and basic auth |
| xform\_rest\_to\_inputvar | map-jobvite-json-to-int |
| Shared forEach | Process Call proc-sub-int-validate-create (int-file-intake) |
| wait + Rest\_DatProfile | Process Call dat-profile-generation-rest after confirmed creation |

**Components**

| Component | Role |
| --- | --- |
| map-jobvite-json-to-int | JSON to INT map |
| proc-jobvite-create-employee-rest | Listener process |

## Client questions

- **Q-001** Does any system other than the Jobvite file intake call the TaskUs "/create" hire service?
  **If this is not answered:** Only the Jobvite file intake uses it; no external endpoint is published.
- **Q-002** Should the Jobvite side be told which hires were rejected, rather than always receiving "received successfully"?
  **If this is not answered:** Return per-hire outcomes.
