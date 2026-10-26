# Migration Plan — ProcessEmpUpdateReq_Sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.782Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessEmpUpdateReq_Sch |
| Identifier | emp-update-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | medium (55) |
| Components to build | 3 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/ProcessEmpUpdateReq_Sch_cfgplan_prod.xml |
| Depends on | newhire-emp-sch, gsuite-create-alias, teleopti-update-user, util-send-notification, hcm-dat-generation |

## 2. End-to-End Process Architecture

BPEL composite started by the XXTS scheduler servlet; reads the HCM empupdate ATOM feed and the emps REST resource (URI built from Credentials.dvm), routes on the Atom summary, and calls ProcessCreateGSuiteAlias, ProcessUpdateTeleOptUser and XXSendNotification. Google update and AD update references are wired but disabled.

Scheduled Boomi process "Employee Name Update Sch": proc-sub-schedule-window, op-hcm-atom-feed-get (collection empupdate) and fn-atom-content-parse (owned by newhire-emp-sch), Route on summary, op-hcm-emps-get, then the two maps of this interface and Process Calls to gsuite-create-alias, teleopti-update-user and util-send-notification, each inside a per-entry Try/Catch. Always advance or report the window.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| EmpUpdateFeed REST reference | op-hcm-atom-feed-get (collection = empupdate) | Shared component. |
| substring parsing of entry content | fn-atom-content-parse |  |
| GetEmployee REST reference | op-hcm-emps-get |  |
| Check_UpdateorCorrection if/elseif | Route shape on summary + Decision on KnownAs |  |
| Assign to ProcessUpdateTeleOptUser | map-empupd-to-teleopti + Process Call teleopti-update-user |  |
| Assign to ProcessCreateGSuiteAlias | map-empupd-to-gsuite-alias + Process Call gsuite-create-alias |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Start and read window | Direct binding from the scheduler; Select_XXTS_Scheduling for INTERFACE_NAME ProcessEmpUpdateReq_Sch; format and append .000Z/.999Z. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel |
| 2 | Read empupdate feed | REST GET /empupdate?updated-min=<min>&updated-max=<max>, Content-Type/Accept application/atom+xml, Authorization from Credentials.dvm. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/WADLs/EmpUpdateFeed.wadl |
| 3 | Feed returned? | If count(reply) > 0 loop over the entries; otherwise do nothing (no window update, no callback). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel |
| 4 | Route each entry | forEach parallel="no" over atom entries; branch on entry summary. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel |
| 5 | Name corrected: update TeleOpti | summary = "Person Name Corrected": PersonNumber from content; GET emps?q=PersonNumber=; if names contain U+FFFD re-derive "Last,First" from PersonName in content; old-name logic (overwritten); Google update (skipped) and AD update (skipped); ProcessUpdateTeleOptUser with EmployeeID, ReqNumber "PersonUpdate", email, first/middle/last name. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel |
| 6 | Preferred name: add Google alias | summary = "Person Name Updated" and Changed Attributes contains "KnownAs": GET emps; ProcessCreateGSuiteAlias with email = WorkEmail, alias = PreferredName. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel |
| 7 | Other name update: notify and update TeleOpti | summary = "Person Name Updated": email EmailDL "trainers" (subject "InstanceID#<id>##Employee Details Updated for PersonNumber ##<pn>"); then same as step 5 but without ReqNumber and without any Google invoke. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel |
| 8 | Advance window | Update_Scheduling_StartDateTime and Update_XXTS_Schedule; direct-binding callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel |

## 4. Payload Profiles

> **Not available.** No profile components are owned by this interface; it uses the shared Atom and emps profiles (newhire-emp-sch) and the request profiles of the called services.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | Employee/EmployeeID | items/PersonNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel | HCM worker to TeleOpti update request |
| MR-002 | Employee/ReqNumber | (none) | conditional | branch A (Person Name Corrected) only | "PersonUpdate" | low | Absent in branch C. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel | HCM worker to TeleOpti update request |
| MR-003 | Employee/Person/EmailAddress | items/WorkEmail | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel | HCM worker to TeleOpti update request |
| MR-004 | Employee/Person/FirstName, LastName, MiddleName | items/FirstName, LastName, MiddleName | conditional | names contain U+FFFD | re-derive First/Last from atom content PersonName "Last,First" | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel | HCM worker to TeleOpti update request |
| MR-005 | email | items/WorkEmail | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel | HCM worker to Google alias request |
| MR-006 | alias | items/PreferredName | passthrough | KnownAs in Changed Attributes |  | low | Not converted to an email address here; gsuite-create-alias sends it as given. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel | HCM worker to Google alias request |
| MR-007 | PersonNumber | entry/content | transformed |  | substring between 'PersonNumber" : "' and '"' | medium | Use a JSON parse (fn-atom-content-parse). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel | Atom entry to routing values |
| MR-008 | Route | entry/summary, content "Changed Attributes" | conditional | "Person Name Corrected" \| "Person Name Updated" + KnownAs \| "Person Name Updated" |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel | Atom entry to routing values |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Preferred name alias | When the preferred (KnownAs) name changes, add it as a Google alias on the work email. | Medium | Decision + map-empupd-to-gsuite-alias + gsuite-create-alias. |
| BR-002 | TeleOpti name sync | Name corrections and non-KnownAs name updates are sent to TeleOpti. | Low | map-empupd-to-teleopti + teleopti-update-user (whose update step is disabled today). |
| BR-003 | Trainer notice | Email the trainers list for each non-KnownAs "Person Name Updated" event. | Low | util-send-notification with proper "@!-" content. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding / SOAP async | processempupdatereq_DirectBinding / processempupdatereq_sch_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/composite.xml |
| outbound REST GET | https://ebly-dev1.hcm.us2.oraclecloud.com:443/hcmCoreApi/atomservlet/employee/empupdate (prod plan https://eetm.hcm.us6.oraclecloud.com:443/...) | HTTP Authorization header from Credentials.dvm | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/WADLs/EmpUpdateFeed.wadl |
| outbound REST GET | <HCMURL>:443/hcmCoreApi/resources/latest/emps?q=PersonNumber=<pn> | HTTP Authorization header from Credentials.dvm | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/WADLs/GetEmployee.wadl |
| outbound SOAP (composite) | oramds:/apps/Provider/GSuit/processcreategsuitealias_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/composite.xml |
| outbound SOAP async (composite) | oramds:/apps/Provider/TeleOpti/processupdateteleoptuser_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/composite.xml |
| outbound SOAP (composite, disabled) | oramds:/apps/Provider/GSuit/processupdategsuituser_client_ep.wsdl; oramds:/apps/Provider/AD/processupdateaduser_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/composite.xml |
| outbound Database | eis/DB/SOADemoLocalTx, XXTS_SCHEDULING | WebLogic data source | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |
| DB adapters (XXTS_SCHEDULING) | Same three pure-SQL statements as the other schedulers (select window, advance start by INTERVAL+1 s, set last_scheduled_time). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| newhire-emp-sch | Shared window and HCM REST/ATOM components | synchronous |
| gsuite-create-alias | Sub: GSuite Alias CREATE | synchronous |
| teleopti-update-user | TeleOpti User UPDATE | fire-and-forget |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google Update request (Operation Update); output not sent because the invoke is skipped. | Not built unless Google renames are re-enabled. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Process-level catchAll only: notification (likely undelivered due to separator format), window advanced, callback. |
| Retries | DB adapter retries only. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| EmailDL.dvm (trainers, notification) | dvm | yes |
| Credentials.dvm (Authorization, HCMURL) | dvm | yes |
| XXTS_SCHEDULING row for ProcessEmpUpdateReq_Sch | other | no |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud ATOM / REST | HTTPS REST | <HCM host>/hcmCoreApi/atomservlet/employee/empupdate; /hcmCoreApi/resources/latest/emps | Oracle (TaskUs tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | EmpUpdateFeed | https://eetm.hcm.us6.oraclecloud.com:[redacted] | [redacted].xml |
| DEV | EmpUpdateFeed | https://ebly-dev1.hcm.us2.oraclecloud.com:[redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

When HR changes or corrects an employee's name in HCM, HCM records an event. This job reads those events for its time window and acts on three kinds: a corrected name is sent to TeleOpti; an update to the preferred ("known as") name adds that name as a Google email alias; any other name update is emailed to the trainers list and also sent to TeleOpti.

Much of what the job was built to do is switched off: it does not rename the Google account or the Active Directory display name. And the TeleOpti service it calls has its own update step switched off, so in practice the job only creates Google aliases and emails trainers.

There are also defects to decide on rather than copy. The logic that picks out the "old" name is overwritten before use and one branch can never run. The trainer and error emails are written in a format the notification utility does not recognise, so they are probably not being delivered today. And if the feed returns nothing usable, the time window is never moved forward.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Map \| HCM Emp to GSuite Alias Request | map-empupd-to-gsuite-alias | transform.map | prof-hcm-emps-json |
| 2 | Map \| HCM Emp to TeleOpti Update Request | map-empupd-to-teleopti | transform.map | prof-hcm-emps-json, prof-employee-ebm-xml |
| 3 | Employee Name Update Sch | proc-emp-update-sch | process | proc-sub-schedule-window, op-hcm-atom-feed-get, fn-atom-content-parse, op-hcm-emps-get, map-empupd-to-teleopti, map-empupd-to-gsuite-alias, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/BPEL/ProcessEmpUpdateReq_Sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/ProcessEmpUpdateReq_Sch_cfgplan_prod.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/Adapters/Update_Scheduling_StartDateTime_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/WADLs/EmpUpdateFeed.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/WADLs/GetEmployee.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/FormatEmailContent.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Credentials.dvm

## Open questions

### Still outstanding

REVIEW-NEEDED: When an employee's name is changed or corrected in HCM, which systems should be updated: Google account name, Active Directory display name, TeleOpti? (Today the Google and AD updates are switched off and the TeleOpti update is switched off inside the TeleOpti service, so only aliases and trainer emails happen.)
  - If this is not answered: Build as today: create Google aliases for preferred-name changes and email trainers; no Google, AD or TeleOpti renames.

REVIEW-NEEDED: How often should name changes be processed (for example hourly, daily)? (The job is not in the scheduler spreadsheet and its schedule is not in the export.)
  - If this is not answered: Run once a day at 15:00 UTC with the other employee jobs.

REVIEW-NEEDED: Are trainers currently receiving the "Employee Details Updated" emails? (The email body format used by this job appears incompatible with the notification utility, so the emails may never have been delivered.)
  - If this is not answered: Send the trainer email in the working format to the current trainers list.

REVIEW-NEEDED: Should a preferred name be added to Google as a full email address (for example preferred.lastname@taskus.com) or exactly as entered in HCM? (This job passes the preferred name as entered; the alias service sends it to Google unchanged.)
  - If this is not answered: Pass the preferred name exactly as entered, as today.

