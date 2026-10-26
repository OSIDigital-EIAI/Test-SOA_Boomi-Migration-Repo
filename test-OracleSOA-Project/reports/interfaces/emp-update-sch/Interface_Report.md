# ProcessEmpUpdateReq\_Sch

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

This scheduled job reads name changes recorded in Oracle HCM Cloud (the HCM "empupdate" event feed) and passes them on: a changed preferred ("known as") name becomes a new Google email alias, other name changes are sent to TeleOpti workforce management, and trainers are emailed when a person's name is updated.

| Field | Value |
| --- | --- |
| Interface | ProcessEmpUpdateReq\_Sch |
| Type | Scheduled |
| Complexity | medium |
| Components to build | 3 |
| Mapping rules | 8 |
| Business rules | 3 |

## Interface Summary

When HR changes or corrects an employee's name in HCM, HCM records an event. This job reads those events for its time window and acts on three kinds: a corrected name is sent to TeleOpti; an update to the preferred ("known as") name adds that name as a Google email alias; any other name update is emailed to the trainers list and also sent to TeleOpti.  Much of what the job was built to do is switched off: it does not rename the Google account or the Active Directory display name. And the TeleOpti service it calls has its own update step switched off, so in practice the job only creates Google aliases and emails trainers.  There are also defects to decide on rather than copy. The logic that picks out the "old" name is overwritten before use and one branch can never run. The trainer and error emails are written in a format the notification utility does not recognise, so they are probably not being delivered today. And if the feed returns nothing usable, the time window is never moved forward.

**Complexity:** medium (score 55)

## Functional Analysis

1. **Start and read window** — Direct binding from the scheduler; Select\_XXTS\_Scheduling for INTERFACE\_NAME ProcessEmpUpdateReq\_Sch; format and append .000Z/.999Z.
2. **Read empupdate feed** — REST GET /empupdate?updated-min=\<min\>&updated-max=\<max\>, Content-Type/Accept application/atom+xml, Authorization from Credentials.dvm.
3. **Feed returned?** — If count(reply) \> 0 loop over the entries; otherwise do nothing (no window update, no callback).
4. **Route each entry** — forEach parallel="no" over atom entries; branch on entry summary.
5. **Name corrected: update TeleOpti** — summary = "Person Name Corrected": PersonNumber from content; GET emps?q=PersonNumber=; if names contain U+FFFD re-derive "Last,First" from PersonName in content; old-name logic (overwritten); Google update (skipped) and AD update (skipped); ProcessUpdateTeleOptUser with EmployeeID, ReqNumber "PersonUpdate", email, first/middle/last name.
6. **Preferred name: add Google alias** — summary = "Person Name Updated" and Changed Attributes contains "KnownAs": GET emps; ProcessCreateGSuiteAlias with email = WorkEmail, alias = PreferredName.
7. **Other name update: notify and update TeleOpti** — summary = "Person Name Updated": email EmailDL "trainers" (subject "InstanceID\#\<id\>\#\#Employee Details Updated for PersonNumber \#\#\<pn\>"); then same as step 5 but without ReqNumber and without any Google invoke.
8. **Advance window** — Update\_Scheduling\_StartDateTime and Update\_XXTS\_Schedule; direct-binding callback.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding / SOAP async | processempupdatereq\_DirectBinding / processempupdatereq\_sch\_client\_ep | none declared |
| outbound | REST GET | https://ebly-dev1.hcm.us2.oraclecloud.com:443/hcmCoreApi/atomservlet/employee/empupdate (prod plan https://eetm.hcm.us6.oraclecloud.com:443/...) | HTTP Authorization header from Credentials.dvm |
| outbound | REST GET | \<HCMURL\>:443/hcmCoreApi/resources/latest/emps?q=PersonNumber=\<pn\> | HTTP Authorization header from Credentials.dvm |
| outbound | SOAP (composite) | oramds:/apps/Provider/GSuit/processcreategsuitealias\_client\_ep.wsdl | none declared |
| outbound | SOAP async (composite) | oramds:/apps/Provider/TeleOpti/processupdateteleoptuser\_client\_ep.wsdl | none declared |
| outbound | SOAP (composite, disabled) | oramds:/apps/Provider/GSuit/processupdategsuituser\_client\_ep.wsdl; oramds:/apps/Provider/AD/processupdateaduser\_client\_ep.wsdl | none declared |
| outbound | Database | eis/DB/SOADemoLocalTx, XXTS\_SCHEDULING | WebLogic data source |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | HCM worker to TeleOpti update request | Employee/EmployeeID | items/PersonNumber | passthrough | — | — | low | — |
| MR-002 | HCM worker to TeleOpti update request | Employee/ReqNumber | (none) | conditional | branch A (Person Name Corrected) only | "PersonUpdate" | low | Absent in branch C. |
| MR-003 | HCM worker to TeleOpti update request | Employee/Person/EmailAddress | items/WorkEmail | passthrough | — | — | low | — |
| MR-004 | HCM worker to TeleOpti update request | Employee/Person/FirstName, LastName, MiddleName | items/FirstName, LastName, MiddleName | conditional | names contain U+FFFD | re-derive First/Last from atom content PersonName "Last,First" | medium | — |
| MR-005 | HCM worker to Google alias request | email | items/WorkEmail | passthrough | — | — | low | — |
| MR-006 | HCM worker to Google alias request | alias | items/PreferredName | passthrough | KnownAs in Changed Attributes | — | low | Not converted to an email address here; gsuite-create-alias sends it as given. |
| MR-007 | Atom entry to routing values | PersonNumber | entry/content | transformed | — | substring between 'PersonNumber" : "' and '"' | medium | Use a JSON parse (fn-atom-content-parse). |
| MR-008 | Atom entry to routing values | Route | entry/summary, content "Changed Attributes" | conditional | "Person Name Corrected" \| "Person Name Updated" + KnownAs \| "Person Name Updated" | — | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Preferred name alias | When the preferred (KnownAs) name changes, add it as a Google alias on the work email. | Medium |
| BR-002 | TeleOpti name sync | Name corrections and non-KnownAs name updates are sent to TeleOpti. | Low |
| BR-003 | Trainer notice | Email the trainers list for each non-KnownAs "Person Name Updated" event. | Low |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Scheduled start** — Direct binding; schedule unknown
2. **Read XXTS window** — Select\_XXTS\_Scheduling
3. **Get empupdate feed** — HCM ATOM REST
4. **Feed returned?** — count(reply) \> 0; else nothing (no callback)
5. **Notify error, advance window** — catchAll
6. **Route entry by summary** — Corrected / Updated+KnownAs / Updated / other
7. **Get worker from HCM** — REST emps?q=PersonNumber=
8. **Create Google alias** — ProcessCreateGSuiteAlias
9. **Email trainers** — EmailDL trainers
10. **Update TeleOpti name** — ProcessUpdateTeleOptUser, not awaited
11. **More entries?** — forEach parallel=no
12. **Advance window, callback**

## Data Model

_No profile components are owned by this interface; it uses the shared Atom and emps profiles (newhire-emp-sch) and the request profiles of the called services._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| EmpUpdateFeed REST reference | op-hcm-atom-feed-get (collection = empupdate) |
| substring parsing of entry content | fn-atom-content-parse |
| GetEmployee REST reference | op-hcm-emps-get |
| Check\_UpdateorCorrection if/elseif | Route shape on summary + Decision on KnownAs |
| Assign to ProcessUpdateTeleOptUser | map-empupd-to-teleopti + Process Call teleopti-update-user |
| Assign to ProcessCreateGSuiteAlias | map-empupd-to-gsuite-alias + Process Call gsuite-create-alias |

**Components**

| Component | Role |
| --- | --- |
| map-empupd-to-teleopti | TeleOpti update request map |
| map-empupd-to-gsuite-alias | Google alias request map |
| proc-emp-update-sch | Main scheduled process |

## Client questions

- **Q-001** When an employee's name is changed or corrected in HCM, which systems should be updated: Google account name, Active Directory display name, TeleOpti?
  **If this is not answered:** Build as today: create Google aliases for preferred-name changes and email trainers; no Google, AD or TeleOpti renames.
- **Q-002** How often should name changes be processed (for example hourly, daily)?
  **If this is not answered:** Run once a day at 15:00 UTC with the other employee jobs.
- **Q-003** Are trainers currently receiving the "Employee Details Updated" emails?
  **If this is not answered:** Send the trainer email in the working format to the current trainers list.
- **Q-004** Should a preferred name be added to Google as a full email address (for example preferred.lastname@taskus.com) or exactly as entered in HCM?
  **If this is not answered:** Pass the preferred name exactly as entered, as today.
