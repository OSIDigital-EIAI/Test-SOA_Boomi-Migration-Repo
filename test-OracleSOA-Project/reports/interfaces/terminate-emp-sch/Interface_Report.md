# ProcessTerminateEmpReq\_Sch

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

Once a day this job finds Philippine employees whose termination took effect in the latest time window and removes their access: it suspends their Google Workspace account permanently, moves their Active Directory account to the Disabled Accounts container, and emails the "suspended" distribution list for each person.

| Field | Value |
| --- | --- |
| Interface | ProcessTerminateEmpReq\_Sch |
| Type | Scheduled |
| Complexity | medium |
| Components to build | 3 |
| Mapping rules | 17 |
| Business rules | 3 |

## Interface Summary

This job removes access for people who have left TaskUs in the Philippines. Each day it asks HCM reporting for workers whose actual termination date falls in the latest time window, looks each one up in HCM to get their work email and names, then suspends their Google account, moves their Active Directory account to the disabled area, and emails the "suspended" distribution list.  Although the job is named after HCM's termination event feed, that feed is switched off in the code; the live logic uses an HCM reporting query restricted to country PH. Two other steps written into the job are also switched off: telling the "Interviewed" recruiting platform about the separation, and ending the person in TeleOpti.  In Boomi this becomes a scheduled process that calls the shared HCM query, Google and AD sub-processes. Two behaviours should be decided rather than copied: an error with one leaver stops processing of all remaining leavers in that run, and the time window is moved on even if the run fails, so those leavers would not be retried automatically.

**Complexity:** medium (score 52)

## Functional Analysis

1. **Start and read window** — Direct binding start from the scheduler; Select\_XXTS\_Scheduling for INTERFACE\_NAME ProcessTerminateEmpReq\_Sch; format yyyy-MM-ddTHH:mm:ss and append .000Z/.999Z.
2. **Query terminated workers** — OTBI: SELECT 0 s\_0, Actual Termination Date s\_1, Worker Creation Date s\_2, Employee Hire Date s\_3, Last Update Date s\_4, Person ID s\_5, Person Number s\_6 FROM "Workforce Management - Work Relationship Real Time" WHERE ("Location"."Country Code" = 'PH') AND ("Work Relationship Details"."Actual Termination Date" BETWEEN '\<min\>' AND '\<max\>'). Rowset parsed with oraext:parseXML.
3. **Any leavers?** — If count(Row) \> 0 (the ATOM alternative is always empty) process each row; otherwise go to the window update.
4. **Look up worker in HCM** — Per row (forEach parallel="no"): PersonNumber = Column6; HCM REST GET \<Credentials.dvm HCMURL\>:443/hcmCoreApi/resources/latest/emps?q=PersonNumber=\<pn\>, Content-Type application/vnd.oracle.adf.resourceitem+json, Authorization header from Credentials.dvm.
5. **Suspend Google account** — xform\_hcm\_to\_gsuite builds an Operation=Suspend request (suspended=true, Status "Permanently Suspended"); ProcessUpdateGSuitUser invoked without waiting for its callback.
6. **Disable AD account** — xform\_hcm\_to\_AD builds CN = initials + person number, Action Inactive, Status Permanent; ProcessUpdateOUADUser invoked without waiting for its callback.
7. **Notify suspended list** — XXSendNotification to EmailDL "suspended": subject "InstanceID\#\<id\>EmployeeTerminated\# PersonNumber:\<pn\>", content Date / Instance ID / PersonID / Termination Date / "Employee Terminated".
8. **Advance window** — Update\_Scheduling\_StartDateTime (start + INTERVAL + 1 s) and Update\_XXTS\_Schedule (last\_scheduled\_time = now), then direct-binding callback.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding / SOAP async | processterminateempreq\_DirectBinding / processterminateempreq\_sch\_client\_ep | none declared |
| outbound | Database | eis/DB/SOADemoLocalTx, XXTS\_SCHEDULING | WebLogic data source |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared |
| outbound | REST GET over HTTPS | \<HCMURL\>:443/hcmCoreApi/resources/latest/emps?q=PersonNumber=\<pn\> (composite default https://ebly-dev1.hcm.us2.oraclecloud.com:443/hcmCoreApi/resources/latest/emps) | HTTP Authorization header (value from Credentials.dvm) |
| outbound | SOAP async (composite) | oramds:/apps/Provider/GSuit/processupdategsuituser\_client\_ep.wsdl | none declared |
| outbound | SOAP async (composite) | oramds:/apps/Provider/AD/processupdateouaduser\_client\_ep.wsdl | none declared |
| outbound | SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification\_client\_ep.wsdl | none declared |
| outbound | REST GET (disabled) | https://ebly-dev1.hcm.us2.oraclecloud.com:443/hcmCoreApi/atomservlet/employee/termination (prod plan: eetm.hcm.us6) | HTTP Authorization header |
| outbound | REST POST (disabled) | https://taskus.interviewed.com/integrations/taskus/update\_employee?api\_key=\[redacted\] | API key in query string |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | HCM worker to Google suspend request | Email, User/primaryEmail, User/emails/address | items/WorkEmail | passthrough | — | — | low | — |
| MR-002 | HCM worker to Google suspend request | Operation | (none) | hardcoded | — | "Suspend" | low | — |
| MR-003 | HCM worker to Google suspend request | EID | items/PersonNumber | passthrough | — | — | low | — |
| MR-004 | HCM worker to Google suspend request | User/name/givenName, familyName | items/FirstName, LastName | passthrough | — | — | low | — |
| MR-005 | HCM worker to Google suspend request | User/suspended | (none) | hardcoded | — | "true" | low | — |
| MR-006 | HCM worker to Google suspend request | User/password, User/hashFunction | (none) | hardcoded | — | literal "" and "false" | low | — |
| MR-007 | HCM worker to Google suspend request | User/addresses/streetAddress, region, postalCode | items/AddressLine1, Region, PostalCode | passthrough | — | — | low | — |
| MR-008 | HCM worker to Google suspend request | User/organizations/name, title | (none) | hardcoded | — | "taskus.com", "SWE" | low | — |
| MR-009 | HCM worker to Google suspend request | User/phones/value | items/WorkMobilePhoneNumber | passthrough | — | — | low | — |
| MR-010 | HCM worker to Google suspend request | Status | (none) | hardcoded | — | "Permanently Suspended" | low | Used by gsuite-user-lifecycle in its notification. |
| MR-011 | HCM worker to AD disable request | CN | items/FirstName, LastName, PersonNumber | calculated | — | lower(first character of FirstName) + lower(first character of LastName) + PersonNumber (legacy idiom substring-before(x, substring(x,2))) | medium | Legacy idiom returns an empty initial when the 2nd character equals the 1st; Boomi should use the first character directly. |
| MR-012 | HCM worker to AD disable request | Employee\_ID | items/PersonNumber | passthrough | — | — | low | — |
| MR-013 | HCM worker to AD disable request | Employee\_Email | items/WorkEmail | passthrough | — | — | low | — |
| MR-014 | HCM worker to AD disable request | Action, Status | (none) | hardcoded | — | "Inactive", "Permanent" | low | ad-update-ou moves Permanent to OU=Disabled Accounts. |
| MR-015 | Leaver notification content | Email | EmailDL key "suspended" | transformed | — | cross reference lookup | low | — |
| MR-016 | Leaver notification content | Subject | flow id, Column6 | calculated | — | "InstanceID\#"+id+"EmployeeTerminated\# PersonNumber:"+Column6 | low | — |
| MR-017 | Leaver notification content | Content | now, flow id, Column6, Column1 | calculated | — | Date @!- Instance ID @!- PersonID @!- Termination Date @!- Message "Employee Terminated" | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Leaver selection | Workers whose location country code is PH and whose Actual Termination Date is within the window. | High |
| BR-002 | Permanent suspension | Google account suspended with status "Permanently Suspended"; AD account Action Inactive, Status Permanent (Disabled Accounts). | High |
| BR-003 | Leaver notification | Email the "suspended" distribution list for each leaver with person number and termination date. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Scheduled start 14:00 UTC** — Direct binding from SchedulerServlet
2. **Read XXTS window** — Select\_XXTS\_Scheduling
3. **Query PH terminations (OTBI)** — Work Relationship RT, Actual Termination Date in window
4. **Any leavers?** — count(Row) \> 0
5. **Get worker from HCM** — REST emps?q=PersonNumber=
6. **Suspend Google account** — ProcessUpdateGSuitUser, not awaited
7. **Notify error, advance window** — catchAll; remaining leavers skipped
8. **Disable AD account** — ProcessUpdateOUADUser Inactive/Permanent, not awaited
9. **Email suspended list** — XXSendNotification EmailDL suspended
10. **More leavers?** — forEach parallel=no
11. **Advance window, callback** — Update start\_date\_time and last\_scheduled\_time

## Data Model

_This interface owns no profile component: it uses the shared OTBI rowset (hcm-otbi-sql-query) and HCM emps profile (newhire-emp-sch) and the request profiles of gsuite-user-lifecycle and ad-update-ou._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| Scheduler servlet + XXTS DB adapters | Process schedule + proc-sub-schedule-window |
| ProcessHCMExtractSyncProv (OTBI) | Process Call hcm-otbi-sql-query |
| GetEmployee REST reference | op-hcm-emps-get on conn-hcm-rest |
| xform\_hcm\_to\_gsuite.xsl + ProcessUpdateGSuitUser (one-way) | map-term-hcm-to-gsuite-suspend + Process Call gsuite-user-lifecycle |
| xform\_hcm\_to\_AD.xsl + ProcessUpdateOUADUser (one-way) | map-term-hcm-to-ad-disable + Process Call ad-update-ou |
| Process catchAll | Per-leaver Try/Catch + global Try/Catch |

**Components**

| Component | Role |
| --- | --- |
| map-term-hcm-to-gsuite-suspend | Google suspend request map |
| map-term-hcm-to-ad-disable | AD disable request map |
| proc-terminate-emp-sch | Main scheduled process |

## Client questions

- **Q-001** Should leaver de-provisioning cover only employees located in the Philippines, as today, or also other countries such as the US?
  **If this is not answered:** Philippines only, as today.
- **Q-002** Should the recruiting platform (Interviewed) and TeleOpti still be told when an employee leaves, as the original design intended?
  **If this is not answered:** Do not notify Interviewed or TeleOpti.
- **Q-003** If de-provisioning fails for one leaver, should the job continue with the others and report the failure, and should the failed time window be retried?
  **If this is not answered:** Continue with the remaining leavers, report each failure, and do not advance the window if the run itself fails.
- **Q-004** Is the actual termination date the right trigger, or should access be removed on the last working day or a set time before?
  **If this is not answered:** Keep using the actual termination date in HCM.
- **Q-005** Who should receive the per-leaver "Employee Terminated" emails?
  **If this is not answered:** Use the current "suspended" distribution list from production.
