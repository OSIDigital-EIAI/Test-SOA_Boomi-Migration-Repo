# Migration Plan — ProcessTerminateEmpReq_Sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.767Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessTerminateEmpReq_Sch |
| Identifier | terminate-emp-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | medium (52) |
| Components to build | 3 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/ProcessTerminateEmpReq_Sch_cfgplan_prod.xml |
| Depends on | newhire-emp-sch, hcm-otbi-sql-query, gsuite-user-lifecycle, ad-update-ou, util-send-notification |

## 2. End-to-End Process Architecture

BPEL composite started by the XXTS_SCHEDULING servlet. Reads its window with DB adapters, queries OTBI through ProcessHCMExtractSyncProv, looks up each worker with an HCM REST reference whose URI is built at runtime from Credentials.dvm, and invokes the ProcessUpdateGSuitUser and ProcessUpdateOUADUser composites one-way. ATOM termination feed, Interviewed and TeleOpti references are wired but disabled.

Scheduled Boomi process "Employee Termination Deprovisioning Sch" using the shared "Sub: Schedule Window Get and Advance" and "Sub: HCM OTBI Execute SQL"; per leaver (split documents, Try/Catch each) call op-hcm-emps-get (owned by newhire-emp-sch), map with the two maps of this interface, call gsuite-user-lifecycle and ad-update-ou synchronously, and notify via util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| Scheduler servlet + XXTS DB adapters | Process schedule + proc-sub-schedule-window | Owned by newhire-emp-sch. |
| ProcessHCMExtractSyncProv (OTBI) | Process Call hcm-otbi-sql-query |  |
| GetEmployee REST reference | op-hcm-emps-get on conn-hcm-rest | Owned by newhire-emp-sch. |
| xform_hcm_to_gsuite.xsl + ProcessUpdateGSuitUser (one-way) | map-term-hcm-to-gsuite-suspend + Process Call gsuite-user-lifecycle | Synchronous; result checked. |
| xform_hcm_to_AD.xsl + ProcessUpdateOUADUser (one-way) | map-term-hcm-to-ad-disable + Process Call ad-update-ou |  |
| Process catchAll | Per-leaver Try/Catch + global Try/Catch |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Start and read window | Direct binding start from the scheduler; Select_XXTS_Scheduling for INTERFACE_NAME ProcessTerminateEmpReq_Sch; format yyyy-MM-ddTHH:mm:ss and append .000Z/.999Z. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel |
| 2 | Query terminated workers | OTBI: SELECT 0 s_0, Actual Termination Date s_1, Worker Creation Date s_2, Employee Hire Date s_3, Last Update Date s_4, Person ID s_5, Person Number s_6 FROM "Workforce Management - Work Relationship Real Time" WHERE ("Location"."Country Code" = 'PH') AND ("Work Relationship Details"."Actual Termination Date" BETWEEN '<min>' AND '<max>'). Rowset parsed with oraext:parseXML. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel |
| 3 | Any leavers? | If count(Row) > 0 (the ATOM alternative is always empty) process each row; otherwise go to the window update. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel |
| 4 | Look up worker in HCM | Per row (forEach parallel="no"): PersonNumber = Column6; HCM REST GET <Credentials.dvm HCMURL>:443/hcmCoreApi/resources/latest/emps?q=PersonNumber=<pn>, Content-Type application/vnd.oracle.adf.resourceitem+json, Authorization header from Credentials.dvm. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel |
| 5 | Suspend Google account | xform_hcm_to_gsuite builds an Operation=Suspend request (suspended=true, Status "Permanently Suspended"); ProcessUpdateGSuitUser invoked without waiting for its callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl |
| 6 | Disable AD account | xform_hcm_to_AD builds CN = initials + person number, Action Inactive, Status Permanent; ProcessUpdateOUADUser invoked without waiting for its callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_AD.xsl |
| 7 | Notify suspended list | XXSendNotification to EmailDL "suspended": subject "InstanceID#<id>EmployeeTerminated# PersonNumber:<pn>", content Date / Instance ID / PersonID / Termination Date / "Employee Terminated". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel |
| 8 | Advance window | Update_Scheduling_StartDateTime (start + INTERVAL + 1 s) and Update_XXTS_Schedule (last_scheduled_time = now), then direct-binding callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel |

## 4. Payload Profiles

> **Not available.** This interface owns no profile component: it uses the shared OTBI rowset (hcm-otbi-sql-query) and HCM emps profile (newhire-emp-sch) and the request profiles of gsuite-user-lifecycle and ad-update-ou.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | Email, User/primaryEmail, User/emails/address | items/WorkEmail | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google suspend request |
| MR-002 | Operation | (none) | hardcoded |  | "Suspend" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google suspend request |
| MR-003 | EID | items/PersonNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google suspend request |
| MR-004 | User/name/givenName, familyName | items/FirstName, LastName | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google suspend request |
| MR-005 | User/suspended | (none) | hardcoded |  | "true" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google suspend request |
| MR-006 | User/password, User/hashFunction | (none) | hardcoded |  | literal "" and "false" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google suspend request |
| MR-007 | User/addresses/streetAddress, region, postalCode | items/AddressLine1, Region, PostalCode | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google suspend request |
| MR-008 | User/organizations/name, title | (none) | hardcoded |  | "taskus.com", "SWE" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google suspend request |
| MR-009 | User/phones/value | items/WorkMobilePhoneNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google suspend request |
| MR-010 | Status | (none) | hardcoded |  | "Permanently Suspended" | low | Used by gsuite-user-lifecycle in its notification. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google suspend request |
| MR-011 | CN | items/FirstName, LastName, PersonNumber | calculated |  | lower(first character of FirstName) + lower(first character of LastName) + PersonNumber (legacy idiom substring-before(x, substring(x,2))) | medium | Legacy idiom returns an empty initial when the 2nd character equals the 1st; Boomi should use the first character directly. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_AD.xsl | HCM worker to AD disable request |
| MR-012 | Employee_ID | items/PersonNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_AD.xsl | HCM worker to AD disable request |
| MR-013 | Employee_Email | items/WorkEmail | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_AD.xsl | HCM worker to AD disable request |
| MR-014 | Action, Status | (none) | hardcoded |  | "Inactive", "Permanent" | low | ad-update-ou moves Permanent to OU=Disabled Accounts. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_AD.xsl | HCM worker to AD disable request |
| MR-015 | Email | EmailDL key "suspended" | transformed |  | cross reference lookup | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel | Leaver notification content |
| MR-016 | Subject | flow id, Column6 | calculated |  | "InstanceID#"+id+"EmployeeTerminated# PersonNumber:"+Column6 | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel | Leaver notification content |
| MR-017 | Content | now, flow id, Column6, Column1 | calculated |  | Date @!- Instance ID @!- PersonID @!- Termination Date @!- Message "Employee Terminated" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel | Leaver notification content |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Leaver selection | Workers whose location country code is PH and whose Actual Termination Date is within the window. | High | OTBI SQL via proc-sub-otbi-query with window dates. |
| BR-002 | Permanent suspension | Google account suspended with status "Permanently Suspended"; AD account Action Inactive, Status Permanent (Disabled Accounts). | High | Two maps + Process Calls to gsuite-user-lifecycle and ad-update-ou. |
| BR-003 | Leaver notification | Email the "suspended" distribution list for each leaver with person number and termination date. | Medium | util-send-notification sub-process. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding / SOAP async | processterminateempreq_DirectBinding / processterminateempreq_sch_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/composite.xml |
| outbound Database | eis/DB/SOADemoLocalTx, XXTS_SCHEDULING | WebLogic data source | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/composite.xml |
| outbound REST GET over HTTPS | <HCMURL>:443/hcmCoreApi/resources/latest/emps?q=PersonNumber=<pn> (composite default https://ebly-dev1.hcm.us2.oraclecloud.com:443/hcmCoreApi/resources/latest/emps) | HTTP Authorization header (value from Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/WADLs/GetEmployee.wadl |
| outbound SOAP async (composite) | oramds:/apps/Provider/GSuit/processupdategsuituser_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/composite.xml |
| outbound SOAP async (composite) | oramds:/apps/Provider/AD/processupdateouaduser_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/composite.xml |
| outbound SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/composite.xml |
| outbound REST GET (disabled) | https://ebly-dev1.hcm.us2.oraclecloud.com:443/hcmCoreApi/atomservlet/employee/termination (prod plan: eetm.hcm.us6) | HTTP Authorization header | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/WADLs/TerminationFeed.wadl |
| outbound REST POST (disabled) | https://taskus.interviewed.com/integrations/taskus/update_employee?api_key=[redacted] | API key in query string | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/WADLs/interviewed.wadl |
| DB adapter Select_XXTS_Scheduling | SELECT TO_CHAR(start_date_time,...) start_date_time, TO_CHAR(start_date_time + ((interval)/86400),...) end_date_time FROM XXTS_SCHEDULING where INTERFACE_NAME=#Interface_Name (UI connection SoaCS_TaskUS) | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |
| DB adapter Update_Scheduling_StartDateTime | UPDATE xxts_scheduling SET start_date_time = start + (interval+1)/86400 WHERE INTERFACE_NAME=#Interface_Name | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Adapters/Update_Scheduling_StartDateTime_db.jca |
| DB adapter Update_XXTS_Schedule | update XXTS_SCHEDULING set last_scheduled_time =#last_scheduled_time where interface_name=#interface_name | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| newhire-emp-sch | Sub: Schedule Window Get and Advance (shared components) | synchronous |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| gsuite-user-lifecycle | GSuite User Lifecycle (Suspend) | fire-and-forget |
| ad-update-ou | AD User OU MOVE (Inactive/Permanent) | fire-and-forget |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google suspend request. | map-term-hcm-to-gsuite-suspend | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_AD.xsl | HCM worker to AD OU request with CN built from initials + person number. | map-term-hcm-to-ad-disable | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Single process-level catchAll: notify EmailDL "notification", advance the window, call back. No per-leaver scope. |
| Retries | DB adapter retries only; no business retry. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| EmailDL.dvm (suspended, notification) | dvm | yes |
| Credentials.dvm (HCMURL, Authorization) | dvm | yes |
| XXTS_SCHEDULING row for ProcessTerminateEmpReq_Sch | other | no |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud REST | HTTPS REST | <HCMURL>:443/hcmCoreApi/resources/latest/emps | Oracle (TaskUs tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| DEV | HCM REST / ATOM base | https://ebly-dev1.hcm.us2.oraclecloud.com:443 | [redacted].xml |
| PROD | TerminationFeed (disabled) | https://eetm.hcm.us6.oraclecloud.com:[redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

This job removes access for people who have left TaskUs in the Philippines. Each day it asks HCM reporting for workers whose actual termination date falls in the latest time window, looks each one up in HCM to get their work email and names, then suspends their Google account, moves their Active Directory account to the disabled area, and emails the "suspended" distribution list.

Although the job is named after HCM's termination event feed, that feed is switched off in the code; the live logic uses an HCM reporting query restricted to country PH. Two other steps written into the job are also switched off: telling the "Interviewed" recruiting platform about the separation, and ending the person in TeleOpti.

In Boomi this becomes a scheduled process that calls the shared HCM query, Google and AD sub-processes. Two behaviours should be decided rather than copied: an error with one leaver stops processing of all remaining leavers in that run, and the time window is moved on even if the run fails, so those leavers would not be retried automatically.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Map \| HCM Emp to AD Disable | map-term-hcm-to-ad-disable | transform.map | prof-hcm-emps-json |
| 2 | Map \| HCM Emp to GSuite Suspend | map-term-hcm-to-gsuite-suspend | transform.map | prof-hcm-emps-json |
| 3 | Employee Termination Deprovisioning Sch | proc-terminate-emp-sch | process | proc-sub-schedule-window, proc-sub-otbi-query, op-hcm-emps-get, map-term-hcm-to-gsuite-suspend, map-term-hcm-to-ad-disable, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/ProcessTerminateEmpReq_Sch_cfgplan_prod.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Adapters/Update_Scheduling_StartDateTime_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/Transformations/xform_hcm_to_AD.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/WADLs/TerminationFeed.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/WADLs/GetEmployee.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/WADLs/interviewed.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Credentials.dvm

## Open questions

### Still outstanding

REVIEW-NEEDED: Should leaver de-provisioning cover only employees located in the Philippines, as today, or also other countries such as the US? (The current query only selects country PH; other leavers are not de-provisioned by this job.)
  - If this is not answered: Philippines only, as today.

REVIEW-NEEDED: Should the recruiting platform (Interviewed) and TeleOpti still be told when an employee leaves, as the original design intended? (Both steps are switched off in the current code; rebuilding them needs current endpoints and credentials.)
  - If this is not answered: Do not notify Interviewed or TeleOpti.

REVIEW-NEEDED: If de-provisioning fails for one leaver, should the job continue with the others and report the failure, and should the failed time window be retried? (Today one failure stops the remaining leavers and the window is still moved on, so they are never retried automatically.)
  - If this is not answered: Continue with the remaining leavers, report each failure, and do not advance the window if the run itself fails.

REVIEW-NEEDED: Is the actual termination date the right trigger, or should access be removed on the last working day or a set time before? (Defines when accounts are suspended relative to the employee's last day.)
  - If this is not answered: Keep using the actual termination date in HCM.

REVIEW-NEEDED: Who should receive the per-leaver "Employee Terminated" emails? (Recipients come from the "suspended" distribution list in the lookup table.)
  - If this is not answered: Use the current "suspended" distribution list from production.

