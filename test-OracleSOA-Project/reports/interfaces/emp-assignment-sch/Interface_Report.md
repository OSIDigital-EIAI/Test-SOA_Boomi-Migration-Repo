# ProcessEmpAssignmentReq\_Sch

Test-OracleSOA-Project · Scheduled · complex complexity

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

This scheduled job reads assignment changes recorded in Oracle HCM Cloud (the HCM "empassignment" event feed) and adjusts Google Workspace and Active Directory for Philippine employees: rehires get their Google account restored, employees whose assignment becomes inactive (but who are not terminated) are temporarily suspended in Google and moved to the Pending Termination area in AD, reactivated employees are restored, and department changes update the AD record.

| Field | Value |
| --- | --- |
| Interface | ProcessEmpAssignmentReq\_Sch |
| Type | Scheduled |
| Complexity | complex |
| Components to build | 6 |
| Mapping rules | 30 |
| Business rules | 6 |

## Interface Summary

This job keeps Google and Active Directory access in step with changes to an employee's assignment in HCM. Each day it reads HCM's assignment events for its time window, checks that the person is in the Philippines, and then acts depending on the event: a rehire restores the Google account; an assignment made inactive (other than a termination) temporarily suspends Google and moves the AD account to "Pending Termination"; an assignment made active again reverses that; and a change of organisation updates the AD account.  There is a confirmed defect in the country check: the query text is overwritten before it is sent, so HCM receives only the person number instead of a full query. Whether this has been failing or simply returning nothing in production cannot be told from the code, but either way the actions above may not be happening today. This needs to be confirmed with TaskUs before the job is rebuilt.  Location, job and manager changes are recognised but do nothing, because the TeleOpti updates behind them are switched off. In Boomi the job becomes a scheduled process that parses the HCM events properly (as JSON rather than by searching text), routes them, and calls the shared Google and AD sub-processes.

**Complexity:** complex (score 68)

## Functional Analysis

1. **Start and read window** — Direct binding; Select\_XXTS\_Scheduling for ProcessEmpAssignmentReq\_Sch; format, append .000Z/.999Z.
2. **Read empassignment feed** — REST GET /empassignment?updated-min&updated-max with Atom accept headers and Authorization from Credentials.dvm.
3. **Any entries?** — If count(entry) \> 0 loop over entries (forEach parallel="no").
4. **Look up country** — OTBI via ProcessHCMExtractSyncProv. Intended: SELECT Employee Mailing County Code FROM Worker Assignment Real Time WHERE Person Number = \<pn\>; actual: the second copy overwrites the SQL with "'\<pn\>'" only.
5. **Philippines only** — Continue only if Row\[1\]/Column1 = "PH".
6. **Rehire** — summary "Person Assignment Created" and ActionCode "REHIRE": GET emps; xform\_hcm\_to\_gsuite (Operation Rehire) -\> ProcessUpdateGSuitUser (not awaited); TeleOpti create skipped.
7. **Assignment inactive** — summary "Person Assignment Updated", AssignmentStatusType new = INACTIVE, ActionCode new != TERMINATION: GET emps; Google suspend (TemporarySuspension); AD Inactive/Temporary via ProcessUpdateOUADUser.
8. **Assignment active** — Same with new = ACTIVE: Google suspended=false; AD Active via ProcessUpdateOUADUser.
9. **Organisation change** — summary "Person Assignment Updated" and content contains OrganizationId change: GET emps; ProcessUpdateADUser with Employee\_ID and DN "CN=" + initials + person number; TeleOpti update skipped.
10. **Location / manager / job change** — Matched but the only actions (TeleOpti updates) are skipped: no effect.
11. **Advance window** — Update\_Scheduling\_StartDateTime and Update\_XXTS\_Schedule, then callback.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding / SOAP async | processempassignmentreq\_DirectBinding / processempassignmentreq\_sch\_client\_ep | none declared |
| outbound | REST GET | https://ebly-dev1.hcm.us2.oraclecloud.com:443/hcmCoreApi/atomservlet/employee/empassignment (prod plan eetm.hcm.us6) | HTTP Authorization header from Credentials.dvm |
| outbound | REST GET | \<HCMURL\>:443/hcmCoreApi/resources/latest/emps?q=PersonNumber=\<pn\> | HTTP Authorization header from Credentials.dvm |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared |
| outbound | SOAP async (composite) | oramds:/apps/Provider/GSuit/processupdategsuituser\_client\_ep.wsdl | none declared |
| outbound | SOAP async (composite) | oramds:/apps/Provider/AD/processupdateouaduser\_client\_ep.wsdl | none declared |
| outbound | SOAP (composite) | oramds:/apps/Provider/AD/processupdateaduser\_client\_ep.wsdl | none declared |
| outbound | SOAP (composite, disabled) | oramds:/apps/Provider/TeleOpti/processcreateteleoptiuser\_client\_ep.wsdl; processupdateteleoptuser\_client\_ep.wsdl | none declared |
| outbound | Database | eis/DB/SOADemoLocalTx, XXTS\_SCHEDULING | WebLogic data source |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-101 | HCM worker to Google rehire request | Email, User/primaryEmail, User/emails/address | items/WorkEmail | passthrough | — | — | low | — |
| MR-102 | HCM worker to Google rehire request | Operation | (none) | hardcoded | — | "Rehire" | low | — |
| MR-103 | HCM worker to Google rehire request | EID | items/PersonNumber | passthrough | — | — | low | — |
| MR-104 | HCM worker to Google rehire request | User/name/givenName, familyName | items/FirstName, LastName | passthrough | — | — | low | — |
| MR-105 | HCM worker to Google rehire request | Alias | items/PreferredName | passthrough | — | — | low | — |
| MR-106 | HCM worker to Google rehire request | User/password, hashFunction, organizations name/title | (none) | hardcoded | — | literal "", "false", "taskus.com", "SWE" | low | — |
| MR-107 | HCM worker to Google rehire request | User/addresses (streetAddress, region, postalCode), phones/value | items/AddressLine1, Region, PostalCode, WorkMobilePhoneNumber | passthrough | — | — | low | — |
| MR-201 | HCM worker to Google unsuspend request | Email, User/primaryEmail, User/emails/address | items/WorkEmail | passthrough | — | — | low | — |
| MR-202 | HCM worker to Google unsuspend request | Operation | (none) | hardcoded | — | "Suspend" | low | — |
| MR-203 | HCM worker to Google unsuspend request | EID | items/PersonNumber | passthrough | — | — | low | — |
| MR-204 | HCM worker to Google unsuspend request | User/name/givenName, familyName | items/FirstName, LastName | passthrough | — | — | low | — |
| MR-205 | HCM worker to Google unsuspend request | User/suspended | (none) | hardcoded | — | "false" | low | — |
| MR-206 | HCM worker to Google unsuspend request | User/password, hashFunction, organizations name/title | (none) | hardcoded | — | literal "", "false", "taskus.com", "SWE" | low | — |
| MR-207 | HCM worker to Google unsuspend request | User/addresses (streetAddress, region, postalCode), phones/value | items/AddressLine1, Region, PostalCode, WorkMobilePhoneNumber | passthrough | — | — | low | — |
| MR-301 | HCM worker to Google temporary suspend request | Email, User/primaryEmail, User/emails/address | items/WorkEmail | passthrough | — | — | low | — |
| MR-302 | HCM worker to Google temporary suspend request | Operation | (none) | hardcoded | — | "Suspend" | low | — |
| MR-303 | HCM worker to Google temporary suspend request | EID | items/PersonNumber | passthrough | — | — | low | — |
| MR-304 | HCM worker to Google temporary suspend request | User/name/givenName, familyName | items/FirstName, LastName | passthrough | — | — | low | — |
| MR-305 | HCM worker to Google temporary suspend request | User/suspended | (none) | hardcoded | — | "true" | low | — |
| MR-306 | HCM worker to Google temporary suspend request | User/password, hashFunction, organizations name/title | (none) | hardcoded | — | literal "", "false", "taskus.com", "SWE" | low | — |
| MR-307 | HCM worker to Google temporary suspend request | User/addresses (streetAddress, region, postalCode), phones/value | items/AddressLine1, Region, PostalCode, WorkMobilePhoneNumber | passthrough | — | — | low | — |
| MR-308 | HCM worker to Google temporary suspend request | Status | (none) | hardcoded | — | "TemporarySuspension" | low | — |
| MR-401 | HCM worker to AD active request | CN | items/FirstName, LastName, PersonNumber | calculated | — | lower(first char FirstName)+lower(first char LastName)+PersonNumber (substring-before(x,substring(x,2)) idiom) | medium | Idiom fails when the 2nd character equals the 1st. |
| MR-402 | HCM worker to AD active request | Employee\_ID / Employee\_Email | items/PersonNumber / items/WorkEmail | passthrough | — | — | low | — |
| MR-403 | HCM worker to AD active request | Action | (none) | hardcoded | — | "Active" | low | — |
| MR-501 | HCM worker to AD inactive request | CN | items/FirstName, LastName, PersonNumber | calculated | — | lower(first char FirstName)+lower(first char LastName)+PersonNumber (substring-before(x,substring(x,2)) idiom) | medium | Idiom fails when the 2nd character equals the 1st. |
| MR-502 | HCM worker to AD inactive request | Employee\_ID / Employee\_Email | items/PersonNumber / items/WorkEmail | passthrough | — | — | low | — |
| MR-503 | HCM worker to AD inactive request | Action, Status | (none) | hardcoded | — | "Inactive", "Temporary" | low | — |
| MR-601 | Organisation change to AD update request | Employee\_ID | entry/content PersonNumber | transformed | — | substring parse of content | low | — |
| MR-602 | Organisation change to AD update request | DN | items/FirstName, LastName, PersonNumber | calculated | — | "CN=" + lower(initial First) + lower(initial Last) + PersonNumber | medium | Department/DisplayName are not sent; ad-update-user derives department from OTBI. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Philippines only | Only act on employees whose Employee Mailing County Code is PH. | High |
| BR-002 | Inactive but not terminated | AssignmentStatusType new INACTIVE and ActionCode new not TERMINATION -\> temporary suspension in Google and Pending Termination in AD. | High |
| BR-003 | Reactivation | AssignmentStatusType new ACTIVE and ActionCode new not TERMINATION -\> unsuspend Google, AD Active. | High |
| BR-004 | Rehire | Person Assignment Created with ActionCode REHIRE -\> Google Operation Rehire. | High |
| BR-005 | Organisation change | OrganizationId change -\> AD update (department derived downstream). | Medium |
| BR-006 | First match wins | Branches are evaluated in order: rehire, inactive, active, organisation, location, manager, job; only the first match runs. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Scheduled start 15:00 UTC** — Direct binding
2. **Read XXTS window**
3. **Get empassignment feed** — HCM ATOM
4. **Any entries?** — count(entry) \> 0
5. **Look up country (OTBI)** — SQL overwritten - defect
6. **Country is PH?** — Row\[1\]/Column1 = PH
7. **Notify error, advance window** — catchAll
8. **Route by event type** — REHIRE / INACTIVE / ACTIVE / Org / Loc / Mgr / Job
9. **Get worker from HCM** — REST emps?q=PersonNumber=
10. **Google rehire** — Operation Rehire
11. **Google suspend, AD pending** — TemporarySuspension; AD Inactive/Temporary
12. **Google unsuspend, AD active** — suspended=false; AD Active
13. **AD department update** — ProcessUpdateADUser
14. **More entries?** — forEach parallel=no
15. **Advance window, callback**

## Data Model

_No profile components are owned by this interface; it uses the shared Atom, emps and OTBI rowset profiles and the called services' request profiles._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| EmpAssignmentFeed REST | op-hcm-atom-feed-get (collection = empassignment) |
| Substring parsing of content | fn-atom-content-parse |
| ProcessHCMExtractSyncProv country lookup (defective SQL) | hcm-otbi-sql-query with corrected SQL |
| CheckAssignmentStatus if/elseif chain | Route shape |
| Five XSLs | Five maps of this interface |
| One-way invokes of Google / AD composites | Synchronous Process Calls with result checks |

**Components**

| Component | Role |
| --- | --- |
| map-asg-to-gsuite-rehire | Google rehire map |
| map-asg-to-gsuite-active | Google unsuspend map |
| map-asg-to-gsuite-inactive | Google temporary suspend map |
| map-asg-to-ad-active | AD active map |
| map-asg-to-ad-inactive | AD inactive map |
| proc-emp-assignment-sch | Main scheduled process |

## Client questions

- **Q-001** Are Google suspensions and AD moves currently happening when an employee's assignment is made inactive, reactivated or rehired in HCM?
  **If this is not answered:** Build the intended behaviour (PH employees: rehire, inactive, active and organisation changes actioned) with the country check fixed.
- **Q-002** Should location, job and manager changes in HCM trigger any updates (for example in TeleOpti or AD)?
  **If this is not answered:** No action for location, job or manager changes.
- **Q-003** Should these access changes apply only to Philippine employees, or to all countries?
  **If this is not answered:** Philippines only.
- **Q-004** When one HCM change includes both a status change and an organisation change, should both be applied?
  **If this is not answered:** Apply only the status change, as today.
- **Q-005** Should a rehired employee also get a TeleOpti account again?
  **If this is not answered:** No TeleOpti account on rehire.
