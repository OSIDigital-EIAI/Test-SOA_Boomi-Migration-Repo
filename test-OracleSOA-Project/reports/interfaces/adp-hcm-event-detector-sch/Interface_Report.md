# ProcessNewHireEmpBIReq\_Sch

Test-OracleSOA-Project · Scheduled · medium complexity

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

Scheduled job meant to detect HCM employee events for ADP payroll (new hires, terminations, rehires, name and marital-status changes) and pass them to adp-employee-masterdata. As exported only terminations are active: it reads the scheduler window, queries HCM for work relationships whose actual termination date falls in that window, maps each to an employee request with action "terminate", and calls the ADP master-data service if any were found.

| Field | Value |
| --- | --- |
| Interface | ProcessNewHireEmpBIReq\_Sch |
| Type | Scheduled |
| Complexity | medium |
| Components to build | 2 |
| Mapping rules | 6 |
| Business rules | 2 |

## Interface Summary

This job is the trigger that decides which employees go into the ADP master-data file. It was designed to pick up new hires, rehires, terminations, and name and marital-status changes from HCM. In the exported version only the termination check is switched on: each run asks HCM for employees whose termination date falls in the scheduler window and, if there are any, sends them to the ADP file builder as terminations.  The step that moves the scheduler window forward is also switched off, so every run looks at the same dates and would send the same terminations again. The new-hire check that is switched off actually contains a termination query too, so it could not simply be re-enabled.  Before the ADP interfaces can be scoped TaskUs must decide which employee events ADP needs and how they should be detected. In Boomi this becomes a scheduled process using the shared window and OTBI sub-processes, a small map to the employee request and a call to the ADP master-data sub-process.

**Complexity:** medium (score 40)

## Functional Analysis

1. **Start and read window** — SOAP or direct binding; XXTS window to updatedMin/updatedMax.
2. **Read HCM feeds (disabled)** — empupdate and empassignment Atom feeds — skipped.
3. **New-hire query (disabled)** — Skipped; contains a termination-date query.
4. **Find terminations** — OTBI Actual Termination Date between window dates.
5. **Build employee request** — xform\_hcm\_to\_adp.
6. **Call ADP master data** — If at least one employee.
7. **Advance window (disabled)** — Scheduler update skipped; client callback sent.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding / SOAP async | ProcessNewHireEmpBIReqDirectBinding / processnewhireempbireq\_sch\_client\_ep | none declared |
| outbound | SOAP async (composite) | oramds:/apps/Provider/HCM/processhcmextractprovabcs\_client\_ep.wsdl | none declared |
| outbound | SOAP async (composite) | oramds:/apps/Provider/ADP/processemployeeadpprovabcs\_client\_ep.wsdl | none declared |
| outbound | REST GET Atom (disabled) | \<HCM host\>/hcmCoreApi/atomservlet/employee (empupdate, empassignment) | not determinable from the disabled scopes |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | HCM events to ADP employee request | EmployeeID | Terminate rowset Column5 (Person Number) | passthrough | — | — | low | — |
| MR-002 | HCM events to ADP employee request | EffectiveStartDate | Column6 (Termination Date) | passthrough | — | — | low | — |
| MR-003 | HCM events to ADP employee request | LastReviewedDate | Column3 (Last Update Date) | calculated | — | replace / with -, add 1 day, date part | low | — |
| MR-004 | HCM events to ADP employee request | AdjustedServiceDate | $Terminate\_RowSetVar/rowset/Row/Column6 (absolute) | passthrough | — | — | low | Defect: first row's value for every employee. |
| MR-005 | HCM events to ADP employee request | CustomAttributes/Description | (constant per source) | hardcoded | terminations | "terminate"; disabled sources: newhire, namechange, taxdependentschange, rehire | low | — |
| MR-006 | HCM events to ADP employee request | EmployeeID / EffectiveStartDate (disabled feeds) | Atom content PersonNumber / EffectiveDate | conditional | "Person Name Corrected/Updated" -\> namechange; "Person Legislative Information Updated" with MaritalStatus -\> taxdependentschange; "Person Assignment Created" with ActionCode REHIRE -\> rehire | substring parsing | medium | Inactive while the feed scopes are skipped. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Terminations in window | Work relationships with Actual Termination Date between window start and end dates. | High |
| BR-002 | Call only when needed | ADP master-data service called only when at least one employee was found. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Scheduled start, read window**
2. **Read HCM feeds (disabled)** — Skipped
3. **Find terminations in window** — OTBI
4. **Map to employee request**
5. **Any employees?**
6. **Call ADP master data** — Async
7. **Callback (window not advanced)**

## Data Model

_No profile component is owned by this interface; it uses the OTBI rowset profile of hcm-otbi-sql-query and the employee request of adp-employee-masterdata._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| XXTS window (update skipped) | Sub: Schedule Window Get and Advance |
| Async OTBI termination query | Process Call hcm-otbi-sql-query |
| xform\_hcm\_to\_adp | map-events-to-adp-request |
| Async call to ProcessEmployeeADPProvABCS | Process Call proc-adp-employee-masterdata |

**Components**

| Component | Role |
| --- | --- |
| map-events-to-adp-request | Event to employee request map |
| proc-adp-hcm-event-detector-sch | Scheduled process |

## Client questions

- **Q-001** Which employee events must be sent to ADP — new hires, rehires, terminations, name changes, marital-status changes, transfers — and how should each be detected?
  **If this is not answered:** Send new hires, rehires, terminations, name and marital-status changes, detected from HCM each run, with the window advanced after each successful run.
- **Q-002** How often should employee events be sent to ADP?
  **If this is not answered:** Once a day.
