# ProcessUpdateTeleOptiUser

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

Intended to update an agent's TeleOpti person record (name, job, department, location, manager) from HCM and, for "PersonUpdate" requests, to update the agent's direct reports as well. As exported every UpdatePerson call is disabled: the live path only reads HCM, checks eligibility and looks the person up in TeleOpti by employment number.

| Field | Value |
| --- | --- |
| Interface | ProcessUpdateTeleOptiUser |
| Type | Listener |
| Complexity | medium |
| Components to build | 3 |
| Mapping rules | 6 |
| Business rules | 2 |

## Interface Summary

This service was meant to keep agents' details in TeleOpti in line with HCM: when an agent's name or job information changes, it finds them in TeleOpti by employee number and updates their record, and for a manager it was also meant to update everyone reporting to them.  In the exported code every update step is switched off. What still runs is the HCM lookup, an eligibility check (department in an allowed list, Associate or Senior Associate, assignment category "Eligible for OT ND") and the TeleOpti lookup. So TeleOpti is never updated. The code also has inconsistencies with the creation service: it checks for "Eligible for OT ND" (with a space) where creation checks "Eligible for OTND", and it would set the time zone to Western Europe where creation uses Singapore. On the not-eligible path it answers the caller twice, and on error it never answers.  TaskUs needs to decide whether TeleOpti updates, and the cascade to direct reports, are wanted at all before this is built.

**Complexity:** medium (score 32)

## Functional Analysis

1. **Receive employee** — Async SOAP process with EmployeeEBM.
2. **Read HCM details** — OTBI by Person Number.
3. **Eligible?** — Departments.dvm allow-list, ASSOCIATE / SENIOR ASSOCIATE, "Eligible for OT ND"; otherwise callback (twice).
4. **Find TeleOpti person** — GetPersonsByQuery by employment number with auth header.
5. **Update person (disabled)** — Copy OTBI values, xform\_EBO\_to\_UpdateTeleOptiUser, set Id from the lookup, UpdatePerson.
6. **Cascade to reports (disabled)** — If ReqNumber = PersonUpdate: OTBI for workers whose Manager Person Number = this person, then look up and update each in TeleOpti.
7. **Callback** — processResponse (no status set).

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | processupdateteleoptuser\_client\_ep | none declared |
| outbound | SOAP 1.1 over HTTPS | https://taskus.teleopticloud.com/SDK/TeleoptiCccSdkService.svc | Custom SOAP header TeleoptiAuthenticationHeader |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | EBM + HCM row to UpdatePerson | Id | GetPersonsByQuery PersonDto/Id | passthrough | — | — | low | — |
| MR-002 | EBM + HCM row to UpdatePerson | FirstName / LastName | OTBI s\_9 / s\_10 | transformed | — | substring(...,1,25) | low | — |
| MR-003 | EBM + HCM row to UpdatePerson | JobTitle / Department / Location | OTBI s\_3 / s\_2 / s\_4 | passthrough | — | — | low | — |
| MR-004 | EBM + HCM row to UpdatePerson | ISEmployeeID / ManagerEmailAddress | OTBI s\_7 / s\_6 | passthrough | — | — | low | — |
| MR-005 | EBM + HCM row to UpdatePerson | TimeZoneId | (none) | hardcoded | — | "W. Europe Standard Time" | low | Differs from create (Singapore). |
| MR-006 | EBM + HCM row to UpdatePerson | ApplicationLogOnName / EmploymentNumber / Email | Person/FirstName / EmployeeID / Person/EmailAddress | passthrough | — | — | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Update eligibility | Department in Departments.dvm, Job ASSOCIATE / SENIOR ASSOCIATE, Assignment Category "Eligible for OT ND". | Medium |
| BR-002 | Cascade to reports | For PersonUpdate, also update every worker whose manager is this person (disabled). | Low |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive employee change** — From emp-update-sch
2. **Read HCM details** — OTBI
3. **Eligible agent?** — Dept list, job, OT ND
4. **Find TeleOpti person** — By employment number
5. **Person found?**
6. **Notify error** — No callback
7. **Update person (disabled)** — UpdatePerson skipped
8. **Callback** — Twice on not-eligible path

## Data Model

_No profile components are owned by this interface; personDto comes from the TeleOpti SDK WSDL._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback | Process start + Return Documents |
| GetPersonsByQuery | op-teleopti-get-persons (teleopti-create-user) |
| UpdatePerson (disabled) | op-teleopti-update-person |
| xform\_EBO\_to\_UpdateTeleOptiUser | map-ebm-to-teleopti-updateperson |

**Components**

| Component | Role |
| --- | --- |
| op-teleopti-update-person | UpdatePerson |
| map-ebm-to-teleopti-updateperson | Update map |
| proc-teleopti-update-user | Main process |

## Client questions

- **Q-001** Should employee changes in HCM update TeleOpti automatically, and should a manager's change also update everyone reporting to them?
  **If this is not answered:** Not built.
- **Q-002** Which time zone should TeleOpti records use: Singapore (as on creation) or Western Europe (as in the update code)?
  **If this is not answered:** Singapore Standard Time, as on creation.
- **Q-003** Is the eligibility category spelled "Eligible for OTND" or "Eligible for OT ND" in HCM?
  **If this is not answered:** Use the value found in HCM for both.
