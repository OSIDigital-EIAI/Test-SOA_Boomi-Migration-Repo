# ProcessEmpMaternityLeave\_Sch

Test-OracleSOA-Project · Scheduled · simple complexity

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

Daily job that disables the Active Directory account of employees whose maternity leave starts today and re-enables the account of employees whose maternity leave ended, by reading maternity absences from Oracle HCM and calling ad-update-ou (ProcessUpdateOUADUser) for each person.

| Field | Value |
| --- | --- |
| Interface | ProcessEmpMaternityLeave\_Sch |
| Type | Scheduled |
| Complexity | simple |
| Components to build | 1 |
| Mapping rules | 4 |
| Business rules | 3 |

## Interface Summary

Every day just after midnight UTC this job looks in HCM for employees whose maternity leave (caesarean, miscarriage or normal) starts that day and asks the Active Directory interface to disable their account as "Temporary". It then looks for employees whose maternity leave ended since yesterday so it can re-enable their accounts.  The second half has a defect: it re-uses the list of people starting leave instead of the people returning, so someone going on leave is disabled and then immediately re-enabled in the same run, while people coming back are never re-enabled automatically. The account name is also passed as "First Last", which is not the format the AD interface expects. Errors for individual people are silently ignored.  In Boomi this is a small scheduled process that runs two HCM queries and calls the AD update sub-process for each person; TaskUs must confirm the intended behaviour and the account-name format before it is built.

**Complexity:** simple (score 20)

## Functional Analysis

1. **Scheduled start** — 00:30 UTC via direct binding (or SOAP client).
2. **Find leave starting today** — OTBI maternity absences with Date Start = today.
3. **Disable AD account** — Per person ad-update-ou Status Temporary, Action Inactive.
4. **Find leave ended** — OTBI maternity absences with Date End between yesterday and today (result not used).
5. **Enable AD account** — Per person of the START list: ad-update-ou Action Active.
6. **Record run** — Update last\_scheduled\_time (direct) or callback (client).

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding / SOAP async | processempmaternityleave\_direct\_ep / processempmaternityleave\_client\_ep | none declared |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared |
| outbound | SOAP async one-way (composite) | oramds:/apps/Provider/AD/processupdateouaduser\_client\_ep.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Absence rows to AD OU update request | CN | Column8, Column9 (First Name, Last Name) | transformed | — | concat with a space | low | ad-update-ou expects its own CN form (see that interface); format to confirm. |
| MR-002 | Absence rows to AD OU update request | Employee\_ID | Column1 (Person Number) | passthrough | — | — | low | — |
| MR-003 | Absence rows to AD OU update request | Status | (constant) | hardcoded | Start of leave | "Temporary"; not set for end of leave | low | — |
| MR-004 | Absence rows to AD OU update request | Action | (constant) | hardcoded | Start / end | "Inactive" at start, "Active" at end | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Maternity absence types | Only Maternity Caesarean, Miscarriage and Normal absences. | High |
| BR-002 | Disable on start | Absence Date Start = today -\> AD Inactive, Status Temporary. | High |
| BR-003 | Enable on return | Absence Date End between yesterday and today -\> AD Active (intended; currently applied to the start list). | High |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Daily start 00:30 UTC**
2. **Find leave starting today** — OTBI
3. **Disable AD accounts** — ad-update-ou Inactive, per person
4. **Find leave ended** — Result unused (defect)
5. **Enable AD accounts** — Iterates start list (defect)
6. **Started by scheduler?**
7. **Update last run time**
8. **Callback client**

## Data Model

_No profile component is owned by this interface; it uses the OTBI rowset of hcm-otbi-sql-query and the request of ad-update-ou._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| ProcessHCMExtractSyncProv | Process Call hcm-otbi-sql-query |
| ProcessUpdateOUADUser one-way | Process Call ad-update-ou |
| Update\_XXTS\_Schedule | Boomi schedule (no table needed) or window sub-process of newhire-emp-sch |
| Empty catchAll | Try/Catch + util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| proc-maternity-leave-ad-sch | Scheduled process |

## Client questions

- **Q-001** Should AD accounts be disabled on the first day of maternity leave and re-enabled the day after it ends?
  **If this is not answered:** Disable on the start date, re-enable the day after the end date, using the CN format ad-update-ou expects.
- **Q-002** Are Maternity Caesarean, Maternity Miscarriage and Maternity Normal still the complete list of absence types that should trigger this?
  **If this is not answered:** Keep these three, configurable.
- **Q-003** Who should be told when an account cannot be disabled or re-enabled?
  **If this is not answered:** Email the IT service desk list.
