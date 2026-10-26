# ProcessUpdateEmployeeEmail

Test-OracleSOA-Project · Listener · simple complexity

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

Sets a worker's work email address in Oracle HCM Cloud. Given a person number, an effective date and an email address, it calls the HCM WorkerService mergePerson operation with a PersonEmail of type W1 (work) marked as primary.

| Field | Value |
| --- | --- |
| Interface | ProcessUpdateEmployeeEmail |
| Type | Listener |
| Complexity | simple |
| Components to build | 4 |
| Mapping rules | 6 |
| Business rules | 1 |

## Interface Summary

This small service writes a new employee's company email address into Oracle HCM. The new-hire job calls it with the employee number, the date the address takes effect and the address itself, and it updates the employee's primary work email in HCM.  It answers the caller with an empty reply whether it worked or not, so the new-hire job cannot tell a failure from a success; failures are only emailed to the notification list. It also uses an older version of Oracle's worker web service, which should be checked against the current HCM release before the rebuild.

**Complexity:** simple (score 12)

## Functional Analysis

1. **Receive request** — Synchronous process with PersonNumber, DateFrom, EmailAddress.
2. **Build mergePerson** — PersonNumber, PersonEmail DateFrom/PersonNumber/EmailType W1/PrimaryFlag true/EmailAddress.
3. **Update HCM** — WorkerService.mergePerson with HTTP Authorization header.
4. **Reply** — Empty processResponse (result not set).

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 synchronous | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateEmployeeEmail/processupdateemployeeemail\_client\_ep | none declared |
| outbound | SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmEmploymentCoreWorkerV2/WorkerService | HTTP Authorization header (Credentials.dvm); WSDL advertises WS-Security |
| outbound | SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification\_client\_ep.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Email request to mergePerson | attributeList/PersonNumber | PersonNumber | passthrough | — | — | low | — |
| MR-002 | Email request to mergePerson | attributeList/PersonEmail/DateFrom | DateFrom | passthrough | — | — | low | — |
| MR-003 | Email request to mergePerson | attributeList/PersonEmail/PersonNumber | PersonNumber | passthrough | — | — | low | — |
| MR-004 | Email request to mergePerson | attributeList/PersonEmail/EmailType | (none) | hardcoded | — | "W1" | low | Work email. |
| MR-005 | Email request to mergePerson | attributeList/PersonEmail/PrimaryFlag | (none) | hardcoded | — | "true" | low | — |
| MR-006 | Email request to mergePerson | attributeList/PersonEmail/EmailAddress | EmailAddress | passthrough | — | — | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Primary work email | The supplied address is stored as email type W1 with PrimaryFlag true from DateFrom. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive email update** — From newhire-emp-sch
2. **Update HCM work email** — WorkerService.mergePerson
3. **Notify error, reply** — EmailDL notification
4. **Reply** — Empty response

## Data Model

_This interface owns no profile components; the mergePerson request/response shapes come from the WorkerService WSDL imported into the connector operation._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / reply | Sub-process start + Return Documents |
| Assign\_UpdateEmail | map-email-to-mergeperson |
| UpdateEmail (WorkerService mergePerson) | conn-hcm-worker-soap + op-hcm-merge-person |
| catchAll | Try/Catch + util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| conn-hcm-worker-soap | HCM WorkerService connection |
| op-hcm-merge-person | mergePerson operation |
| map-email-to-mergeperson | Request map |
| proc-sub-hcm-update-email | Main sub-process |

## Client questions

- **Q-001** Is the Oracle HCM SOAP worker service (version 2) still enabled in your HCM release, or should the new process use the HCM REST API to update work emails?
  **If this is not answered:** Keep the SOAP WorkerService mergePerson as today.
- **Q-002** If the work email cannot be saved in HCM, should the new-hire process stop for that person or continue and report it?
  **If this is not answered:** Continue the new hire and report the failure to the notification list.
