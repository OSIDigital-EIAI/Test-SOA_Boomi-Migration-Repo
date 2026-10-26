# Migration Plan — ProcessEmpAssignmentReq_Sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.804Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessEmpAssignmentReq_Sch |
| Identifier | emp-assignment-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | complex (68) |
| Components to build | 6 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/BPEL/ProcessEmpAssignmentReq_Sch.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/ProcessEmpAssignmentReq_Sch_cfgplan_prod.xml |
| Depends on | newhire-emp-sch, hcm-otbi-sql-query, gsuite-user-lifecycle, ad-update-ou, ad-update-user, util-send-notification |

## 2. End-to-End Process Architecture

BPEL composite started by the XXTS scheduler servlet; reads the HCM empassignment ATOM feed, calls ProcessHCMExtractSyncProv for the country, the emps REST resource for details, and invokes ProcessUpdateGSuitUser, ProcessUpdateOUADUser and ProcessUpdateADUser; TeleOpti create/update references are wired but every call is skipped.

Scheduled Boomi process "Employee Assignment Change Sch": shared window sub-process and Atom/emps components from newhire-emp-sch, fn-atom-content-parse for the event JSON, a corrected OTBI country query via hcm-otbi-sql-query, a Route on event type, the five maps of this interface, and synchronous Process Calls to gsuite-user-lifecycle, ad-update-ou and ad-update-user, each event in its own Try/Catch.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| EmpAssignmentFeed REST | op-hcm-atom-feed-get (collection = empassignment) | Shared. |
| Substring parsing of content | fn-atom-content-parse |  |
| ProcessHCMExtractSyncProv country lookup (defective SQL) | hcm-otbi-sql-query with corrected SQL |  |
| CheckAssignmentStatus if/elseif chain | Route shape |  |
| Five XSLs | Five maps of this interface |  |
| One-way invokes of Google / AD composites | Synchronous Process Calls with result checks |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Start and read window | Direct binding; Select_XXTS_Scheduling for ProcessEmpAssignmentReq_Sch; format, append .000Z/.999Z. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/BPEL/ProcessEmpAssignmentReq_Sch.bpel |
| 2 | Read empassignment feed | REST GET /empassignment?updated-min&updated-max with Atom accept headers and Authorization from Credentials.dvm. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/WADLs/EmpAssignmentFeed.wadl |
| 3 | Any entries? | If count(entry) > 0 loop over entries (forEach parallel="no"). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/BPEL/ProcessEmpAssignmentReq_Sch.bpel |
| 4 | Look up country | OTBI via ProcessHCMExtractSyncProv. Intended: SELECT Employee Mailing County Code FROM Worker Assignment Real Time WHERE Person Number = <pn>; actual: the second copy overwrites the SQL with "'<pn>'" only. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/BPEL/ProcessEmpAssignmentReq_Sch.bpel |
| 5 | Philippines only | Continue only if Row[1]/Column1 = "PH". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/BPEL/ProcessEmpAssignmentReq_Sch.bpel |
| 6 | Rehire | summary "Person Assignment Created" and ActionCode "REHIRE": GET emps; xform_hcm_to_gsuite (Operation Rehire) -> ProcessUpdateGSuitUser (not awaited); TeleOpti create skipped. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl |
| 7 | Assignment inactive | summary "Person Assignment Updated", AssignmentStatusType new = INACTIVE, ActionCode new != TERMINATION: GET emps; Google suspend (TemporarySuspension); AD Inactive/Temporary via ProcessUpdateOUADUser. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_inactive.xsl |
| 8 | Assignment active | Same with new = ACTIVE: Google suspended=false; AD Active via ProcessUpdateOUADUser. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_active.xsl |
| 9 | Organisation change | summary "Person Assignment Updated" and content contains OrganizationId change: GET emps; ProcessUpdateADUser with Employee_ID and DN "CN=" + initials + person number; TeleOpti update skipped. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/BPEL/ProcessEmpAssignmentReq_Sch.bpel |
| 10 | Location / manager / job change | Matched but the only actions (TeleOpti updates) are skipped: no effect. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/BPEL/ProcessEmpAssignmentReq_Sch.bpel |
| 11 | Advance window | Update_Scheduling_StartDateTime and Update_XXTS_Schedule, then callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/BPEL/ProcessEmpAssignmentReq_Sch.bpel |

## 4. Payload Profiles

> **Not available.** No profile components are owned by this interface; it uses the shared Atom, emps and OTBI rowset profiles and the called services' request profiles.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-101 | Email, User/primaryEmail, User/emails/address | items/WorkEmail | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google rehire request |
| MR-102 | Operation | (none) | hardcoded |  | "Rehire" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google rehire request |
| MR-103 | EID | items/PersonNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google rehire request |
| MR-104 | User/name/givenName, familyName | items/FirstName, LastName | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google rehire request |
| MR-105 | Alias | items/PreferredName | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google rehire request |
| MR-106 | User/password, hashFunction, organizations name/title | (none) | hardcoded |  | literal "", "false", "taskus.com", "SWE" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google rehire request |
| MR-107 | User/addresses (streetAddress, region, postalCode), phones/value | items/AddressLine1, Region, PostalCode, WorkMobilePhoneNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | HCM worker to Google rehire request |
| MR-201 | Email, User/primaryEmail, User/emails/address | items/WorkEmail | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_active.xsl | HCM worker to Google unsuspend request |
| MR-202 | Operation | (none) | hardcoded |  | "Suspend" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_active.xsl | HCM worker to Google unsuspend request |
| MR-203 | EID | items/PersonNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_active.xsl | HCM worker to Google unsuspend request |
| MR-204 | User/name/givenName, familyName | items/FirstName, LastName | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_active.xsl | HCM worker to Google unsuspend request |
| MR-205 | User/suspended | (none) | hardcoded |  | "false" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_active.xsl | HCM worker to Google unsuspend request |
| MR-206 | User/password, hashFunction, organizations name/title | (none) | hardcoded |  | literal "", "false", "taskus.com", "SWE" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_active.xsl | HCM worker to Google unsuspend request |
| MR-207 | User/addresses (streetAddress, region, postalCode), phones/value | items/AddressLine1, Region, PostalCode, WorkMobilePhoneNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_active.xsl | HCM worker to Google unsuspend request |
| MR-301 | Email, User/primaryEmail, User/emails/address | items/WorkEmail | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_inactive.xsl | HCM worker to Google temporary suspend request |
| MR-302 | Operation | (none) | hardcoded |  | "Suspend" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_inactive.xsl | HCM worker to Google temporary suspend request |
| MR-303 | EID | items/PersonNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_inactive.xsl | HCM worker to Google temporary suspend request |
| MR-304 | User/name/givenName, familyName | items/FirstName, LastName | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_inactive.xsl | HCM worker to Google temporary suspend request |
| MR-305 | User/suspended | (none) | hardcoded |  | "true" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_inactive.xsl | HCM worker to Google temporary suspend request |
| MR-306 | User/password, hashFunction, organizations name/title | (none) | hardcoded |  | literal "", "false", "taskus.com", "SWE" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_inactive.xsl | HCM worker to Google temporary suspend request |
| MR-307 | User/addresses (streetAddress, region, postalCode), phones/value | items/AddressLine1, Region, PostalCode, WorkMobilePhoneNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_inactive.xsl | HCM worker to Google temporary suspend request |
| MR-308 | Status | (none) | hardcoded |  | "TemporarySuspension" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_inactive.xsl | HCM worker to Google temporary suspend request |
| MR-401 | CN | items/FirstName, LastName, PersonNumber | calculated |  | lower(first char FirstName)+lower(first char LastName)+PersonNumber (substring-before(x,substring(x,2)) idiom) | medium | Idiom fails when the 2nd character equals the 1st. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_ad_active.xsl | HCM worker to AD active request |
| MR-402 | Employee_ID / Employee_Email | items/PersonNumber / items/WorkEmail | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_ad_active.xsl | HCM worker to AD active request |
| MR-403 | Action | (none) | hardcoded |  | "Active" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_ad_active.xsl | HCM worker to AD active request |
| MR-501 | CN | items/FirstName, LastName, PersonNumber | calculated |  | lower(first char FirstName)+lower(first char LastName)+PersonNumber (substring-before(x,substring(x,2)) idiom) | medium | Idiom fails when the 2nd character equals the 1st. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_ad_inactive.xsl | HCM worker to AD inactive request |
| MR-502 | Employee_ID / Employee_Email | items/PersonNumber / items/WorkEmail | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_ad_inactive.xsl | HCM worker to AD inactive request |
| MR-503 | Action, Status | (none) | hardcoded |  | "Inactive", "Temporary" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_ad_inactive.xsl | HCM worker to AD inactive request |
| MR-601 | Employee_ID | entry/content PersonNumber | transformed |  | substring parse of content | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/BPEL/ProcessEmpAssignmentReq_Sch.bpel | Organisation change to AD update request |
| MR-602 | DN | items/FirstName, LastName, PersonNumber | calculated |  | "CN=" + lower(initial First) + lower(initial Last) + PersonNumber | medium | Department/DisplayName are not sent; ad-update-user derives department from OTBI. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/BPEL/ProcessEmpAssignmentReq_Sch.bpel | Organisation change to AD update request |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Philippines only | Only act on employees whose Employee Mailing County Code is PH. | High | Correct OTBI country query + Decision (or use the Atom content / emps country if available). |
| BR-002 | Inactive but not terminated | AssignmentStatusType new INACTIVE and ActionCode new not TERMINATION -> temporary suspension in Google and Pending Termination in AD. | High | Route + map-asg-to-gsuite-inactive + map-asg-to-ad-inactive. |
| BR-003 | Reactivation | AssignmentStatusType new ACTIVE and ActionCode new not TERMINATION -> unsuspend Google, AD Active. | High | Route + map-asg-to-gsuite-active + map-asg-to-ad-active. |
| BR-004 | Rehire | Person Assignment Created with ActionCode REHIRE -> Google Operation Rehire. | High | Route + map-asg-to-gsuite-rehire. |
| BR-005 | Organisation change | OrganizationId change -> AD update (department derived downstream). | Medium | Route + Process Call ad-update-user. |
| BR-006 | First match wins | Branches are evaluated in order: rehire, inactive, active, organisation, location, manager, job; only the first match runs. | Medium | Route shape with ordered conditions, or handle combined changes explicitly (decision). |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding / SOAP async | processempassignmentreq_DirectBinding / processempassignmentreq_sch_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/composite.xml |
| outbound REST GET | https://ebly-dev1.hcm.us2.oraclecloud.com:443/hcmCoreApi/atomservlet/employee/empassignment (prod plan eetm.hcm.us6) | HTTP Authorization header from Credentials.dvm | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/WADLs/EmpAssignmentFeed.wadl |
| outbound REST GET | <HCMURL>:443/hcmCoreApi/resources/latest/emps?q=PersonNumber=<pn> | HTTP Authorization header from Credentials.dvm | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/WADLs/GetEmployee.wadl |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/composite.xml |
| outbound SOAP async (composite) | oramds:/apps/Provider/GSuit/processupdategsuituser_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/composite.xml |
| outbound SOAP async (composite) | oramds:/apps/Provider/AD/processupdateouaduser_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/composite.xml |
| outbound SOAP (composite) | oramds:/apps/Provider/AD/processupdateaduser_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/composite.xml |
| outbound SOAP (composite, disabled) | oramds:/apps/Provider/TeleOpti/processcreateteleoptiuser_client_ep.wsdl; processupdateteleoptuser_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/composite.xml |
| outbound Database | eis/DB/SOADemoLocalTx, XXTS_SCHEDULING | WebLogic data source | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |
| DB adapters (XXTS_SCHEDULING) | Select window; advance start by INTERVAL+1 s; set last_scheduled_time. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| newhire-emp-sch | Shared window and HCM REST/ATOM components | synchronous |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| gsuite-user-lifecycle | GSuite User Lifecycle | fire-and-forget |
| ad-update-ou | AD User OU MOVE | fire-and-forget |
| ad-update-user | AD User Attributes UPDATE | fire-and-forget |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl | Rehire request (Operation Rehire, Alias). | map-asg-to-gsuite-rehire | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_active.xsl | Unsuspend request. | map-asg-to-gsuite-active | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_inactive.xsl | Temporary suspend request. | map-asg-to-gsuite-inactive | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_ad_active.xsl | AD Active request. | map-asg-to-ad-active | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_ad_inactive.xsl | AD Inactive/Temporary request. | map-asg-to-ad-inactive | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Process catchAll only: notification (format likely unparseable), window advanced, callback. |
| Retries | DB adapter retries only. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| EmailDL.dvm (notification) | dvm | yes |
| Credentials.dvm | dvm | yes |
| XXTS_SCHEDULING row for ProcessEmpAssignmentReq_Sch | other | no |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud ATOM / REST | HTTPS REST | <HCM host>/hcmCoreApi/atomservlet/employee/empassignment | Oracle (TaskUs tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | EmpAssignmentFeed | https://eetm.hcm.us6.oraclecloud.com:[redacted] | [redacted].xml |
| DEV | EmpAssignmentFeed | https://ebly-dev1.hcm.us2.oraclecloud.com:[redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

This job keeps Google and Active Directory access in step with changes to an employee's assignment in HCM. Each day it reads HCM's assignment events for its time window, checks that the person is in the Philippines, and then acts depending on the event: a rehire restores the Google account; an assignment made inactive (other than a termination) temporarily suspends Google and moves the AD account to "Pending Termination"; an assignment made active again reverses that; and a change of organisation updates the AD account.

There is a confirmed defect in the country check: the query text is overwritten before it is sent, so HCM receives only the person number instead of a full query. Whether this has been failing or simply returning nothing in production cannot be told from the code, but either way the actions above may not be happening today. This needs to be confirmed with TaskUs before the job is rebuilt.

Location, job and manager changes are recognised but do nothing, because the TeleOpti updates behind them are switched off. In Boomi the job becomes a scheduled process that parses the HCM events properly (as JSON rather than by searching text), routes them, and calls the shared Google and AD sub-processes.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Map \| HCM Emp to AD OU Active | map-asg-to-ad-active | transform.map | prof-hcm-emps-json |
| 2 | Map \| HCM Emp to AD OU Inactive | map-asg-to-ad-inactive | transform.map | prof-hcm-emps-json |
| 3 | Map \| HCM Emp to GSuite Rehire | map-asg-to-gsuite-rehire | transform.map | prof-hcm-emps-json |
| 4 | Map \| HCM Emp to GSuite Suspend Temporary | map-asg-to-gsuite-inactive | transform.map | prof-hcm-emps-json |
| 5 | Map \| HCM Emp to GSuite Unsuspend | map-asg-to-gsuite-active | transform.map | prof-hcm-emps-json |
| 6 | Employee Assignment Change Sch | proc-emp-assignment-sch | process | proc-sub-schedule-window, op-hcm-atom-feed-get, fn-atom-content-parse, op-hcm-emps-get, proc-sub-otbi-query, map-asg-to-gsuite-rehire, map-asg-to-gsuite-active, map-asg-to-gsuite-inactive, map-asg-to-ad-active, map-asg-to-ad-inactive, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/BPEL/ProcessEmpAssignmentReq_Sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/ProcessEmpAssignmentReq_Sch_cfgplan_prod.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Adapters/Update_Scheduling_StartDateTime_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_active.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_gsuite_inactive.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_ad_active.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/Transformations/xform_hcm_to_ad_inactive.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/WADLs/EmpAssignmentFeed.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/WADLs/GetEmployee.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/FormatEmailContent.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Credentials.dvm

## Open questions

### Still outstanding

REVIEW-NEEDED: Are Google suspensions and AD moves currently happening when an employee's assignment is made inactive, reactivated or rehired in HCM? (A defect in the current country check means these actions may not have been running; the answer sets the baseline the new build is tested against.)
  - If this is not answered: Build the intended behaviour (PH employees: rehire, inactive, active and organisation changes actioned) with the country check fixed.

REVIEW-NEEDED: Should location, job and manager changes in HCM trigger any updates (for example in TeleOpti or AD)? (These events are recognised but their only actions (TeleOpti updates) are switched off.)
  - If this is not answered: No action for location, job or manager changes.

REVIEW-NEEDED: Should these access changes apply only to Philippine employees, or to all countries? (The job filters on country PH.)
  - If this is not answered: Philippines only.

REVIEW-NEEDED: When one HCM change includes both a status change and an organisation change, should both be applied? (Today only the first matching action runs.)
  - If this is not answered: Apply only the status change, as today.

REVIEW-NEEDED: Should a rehired employee also get a TeleOpti account again? (The TeleOpti creation on rehire is switched off.)
  - If this is not answered: No TeleOpti account on rehire.

