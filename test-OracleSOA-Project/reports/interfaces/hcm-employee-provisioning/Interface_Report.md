# ProcessEmployeeHCMProvABCSImpl

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

Validates a new hire received from the recruiting intakes and hands it to HCM worker creation. It acknowledges the caller immediately, checks in Oracle HCM (REST emps) that the hire's supervisor exists, converts the salutation (Mr, Ms ...) to the HCM lookup code using the SalutationLOV list of values, and then calls hcm-dat-generation without waiting. If the supervisor is not found it notifies HRIS and stops.

| Field | Value |
| --- | --- |
| Interface | ProcessEmployeeHCMProvABCSImpl |
| Type | Listener |
| Complexity | medium |
| Components to build | 3 |
| Mapping rules | 3 |
| Business rules | 2 |

## Interface Summary

This service is the gate between the recruiting systems and HCM. For each new hire it immediately tells the caller "Employee details received successfully", then checks that the hire's supervisor exists in HCM. If not, it emails the HRIS list ("supervisor ID ... does not exist in HCM") and stops, so the hire is not created until someone fixes the data and resends it. Otherwise it converts the salutation into the code HCM expects and passes the hire to the worker-creation service.  Because the caller is answered before any work is done, it never learns whether the hire was created. Worker creation is also called without waiting, so failures there are only visible through emails. Most of the composite is switched-off code from earlier designs (direct HCM creation through a Java service, Google account creation, payroll allowances, updating the Interviewed system), and one of those switched-off references holds an Interviewed API key in its web address, which should be rotated.  In Boomi this becomes a small sub-process: supervisor check, salutation lookup and a Process Call to hcm-dat-generation, returning the real outcome.

**Complexity:** medium (score 44)

## Functional Analysis

1. **Receive hire** — Synchronous-style SOAP process with EmployeeEBM.
2. **Acknowledge** — Reply candidateID + "Employee details received successfully".
3. **Check supervisor** — HCM REST emps?q=PersonNumber=\<SupervisorEmpID\>; count \> 0 to continue.
4. **Supervisor missing** — EmailDL hris: "SuperviserID\#\<id\> is not existed in HCM" with candidate ID; exit.
5. **Get salutation list** — GET emps?limit=1, href of link SalutationLOV, GET LOV items.
6. **Convert salutation** — Prefix := LookupCode where Meaning = Prefix.
7. **Create worker** — Invoke ProcessDATGenerationABCSImpl with the enriched EBM (no wait).

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 (request/reply) | processemployeehcmprovabcsimpl\_client\_ep | none declared |
| outbound | REST over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCoreApi/resources/latest/emps?q=PersonNumber=\<SupervisorEmpID\> and ?limit=1 | HTTP Authorization header (Credentials.dvm) |
| outbound | REST over HTTPS | SalutationLOV href returned by HCM | HTTP Authorization header (Credentials.dvm) |
| outbound | SOAP async (composite, no wait) | oramds:/apps/Provider/HCM/processdatgenerationabcsimpl\_client\_ep.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Salutation to HCM lookup code | Employee/Person/Prefix | SalutationLOV items LookupCode | conditional | Meaning = input Prefix | — | low | Empty if no match. |
| MR-002 | Acknowledgement | TransmissionAck/candidateID | Employee/CandidateID | passthrough | — | — | low | — |
| MR-003 | Acknowledgement | TransmissionAck/status | (none) | hardcoded | — | "Employee details received successfully" | low | Sent before processing. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Supervisor must exist | A hire is only forwarded when HCM has a person with PersonNumber = SupervisorEmpID. | High |
| BR-002 | Salutation code | Salutation text converted to the HCM SalutationLOV LookupCode. | Low |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive recruited hire** — From intakes
2. **Acknowledge caller** — Before any work
3. **Check supervisor in HCM** — REST emps by PersonNumber
4. **Supervisor found?**
5. **Notify HRIS, stop** — EmailDL hris
6. **Look up salutation code** — SalutationLOV
7. **Start worker creation** — hcm-dat-generation, no wait
8. **Notify error** — EmailDL notification
9. **End**

## Data Model

_No profile components owned here; uses the EBM and ack profiles of hcm-dat-generation and the HCM emps JSON profile of newhire-emp-sch._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive + early reply | Sub-process start; Return Documents after processing |
| GetEmployee (emps?q=PersonNumber) | op-hcm-emps-get on conn-hcm-rest (newhire-emp-sch) |
| GetEmployee limit=1 + GetEmployeeLov | op-hcm-lov-get (href from emps links) |
| xform\_SalutationLOV | map-salutation-lov |
| ProcessDATGeneration (no wait) | Process Call hcm-dat-generation (synchronous) |
| Disabled references | (not migrated) |

**Components**

| Component | Role |
| --- | --- |
| op-hcm-lov-get | GET LOV by href |
| map-salutation-lov | Salutation map |
| proc-sub-hcm-employee-provisioning | Main sub-process |

## Client questions

- **Q-001** Should the recruiting systems be told whether each hire was actually created in HCM, rather than just "received"?
  **If this is not answered:** Return the real outcome (created, rejected for missing supervisor, or failed).
- **Q-002** When a hire's supervisor is not yet in HCM, should the hire be held and retried automatically, or wait for HRIS to resend it?
  **If this is not answered:** Email HRIS and wait for a resend, as today.
- **Q-003** Can the Interviewed API key that appears in the integration configuration be rotated?
  **If this is not answered:** TaskUs rotates it before go-live.
