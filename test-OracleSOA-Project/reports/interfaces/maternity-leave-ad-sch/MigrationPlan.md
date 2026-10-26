# Migration Plan — ProcessEmpMaternityLeave_Sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.617Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessEmpMaternityLeave_Sch |
| Identifier | maternity-leave-ad-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | simple (20) |
| Components to build | 1 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/BPEL/ProcessEmpMaternityLeave.bpel |
| Depends on | hcm-otbi-sql-query, ad-update-ou, newhire-emp-sch, util-send-notification |

## 2. End-to-End Process Architecture

Pick-started BPEL with two sync OTBI calls, two sequential forEach loops of one-way calls to ProcessUpdateOUADUser, and an XXTS last-run update.

Boomi scheduled process "AD Maternity Leave Toggle Sch" (proc-maternity-leave-ad-sch) at 00:30 UTC: hcm-otbi-sql-query for starts and for ends, Process Call ad-update-ou per person (Inactive/Temporary at start, Active at end), errors via util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| ProcessHCMExtractSyncProv | Process Call hcm-otbi-sql-query |  |
| ProcessUpdateOUADUser one-way | Process Call ad-update-ou | Use the end-of-leave result for the return branch. |
| Update_XXTS_Schedule | Boomi schedule (no table needed) or window sub-process of newhire-emp-sch |  |
| Empty catchAll | Try/Catch + util-send-notification |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Scheduled start | 00:30 UTC via direct binding (or SOAP client). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/BPEL/ProcessEmpMaternityLeave.bpel |
| 2 | Find leave starting today | OTBI maternity absences with Date Start = today. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/BPEL/ProcessEmpMaternityLeave.bpel |
| 3 | Disable AD account | Per person ad-update-ou Status Temporary, Action Inactive. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/BPEL/ProcessEmpMaternityLeave.bpel |
| 4 | Find leave ended | OTBI maternity absences with Date End between yesterday and today (result not used). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/BPEL/ProcessEmpMaternityLeave.bpel |
| 5 | Enable AD account | Per person of the START list: ad-update-ou Action Active. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/BPEL/ProcessEmpMaternityLeave.bpel |
| 6 | Record run | Update last_scheduled_time (direct) or callback (client). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca |

## 4. Payload Profiles

> **Not available.** No profile component is owned by this interface; it uses the OTBI rowset of hcm-otbi-sql-query and the request of ad-update-ou.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | CN | Column8, Column9 (First Name, Last Name) | transformed |  | concat with a space | low | ad-update-ou expects its own CN form (see that interface); format to confirm. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/BPEL/ProcessEmpMaternityLeave.bpel | Absence rows to AD OU update request |
| MR-002 | Employee_ID | Column1 (Person Number) | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/BPEL/ProcessEmpMaternityLeave.bpel | Absence rows to AD OU update request |
| MR-003 | Status | (constant) | hardcoded | Start of leave | "Temporary"; not set for end of leave | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/BPEL/ProcessEmpMaternityLeave.bpel | Absence rows to AD OU update request |
| MR-004 | Action | (constant) | hardcoded | Start / end | "Inactive" at start, "Active" at end | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/BPEL/ProcessEmpMaternityLeave.bpel | Absence rows to AD OU update request |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Maternity absence types | Only Maternity Caesarean, Miscarriage and Normal absences. | High | OTBI SQL. |
| BR-002 | Disable on start | Absence Date Start = today -> AD Inactive, Status Temporary. | High | Process Call ad-update-ou. |
| BR-003 | Enable on return | Absence Date End between yesterday and today -> AD Active (intended; currently applied to the start list). | High | Second query and Process Call ad-update-ou. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding / SOAP async | processempmaternityleave_direct_ep / processempmaternityleave_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/composite.xml |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/composite.xml |
| outbound SOAP async one-way (composite) | oramds:/apps/Provider/AD/processupdateouaduser_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/composite.xml |
| DB Update_XXTS_Schedule | update XXTS_SCHEDULING set last_scheduled_time where interface_name = ProcessEmpMaternityLeave_Sch (eis/DB/SOADemoLocalTx). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| ad-update-ou | AD OU / status update | fire-and-forget |
| newhire-emp-sch | Scheduler table pattern (last run) | synchronous |
| util-send-notification | Sub: Send Notification (recommended; not used in legacy) | synchronous |

## 9. Custom Code

> **Not available.** No custom code was found in this interface.

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Per-person empty catchAll; no process-level handler; no notifications. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/AD/processupdateouaduser_client_ep.wsdl | mds | yes |

## 12. External Services

> **Not available.** No external services were recorded for this interface.

## 13. Environment Configuration Matrix

> **Not available.** No environment-specific configuration was recorded.

## 14. Boomi Migration Readiness Summary

Every day just after midnight UTC this job looks in HCM for employees whose maternity leave (caesarean, miscarriage or normal) starts that day and asks the Active Directory interface to disable their account as "Temporary". It then looks for employees whose maternity leave ended since yesterday so it can re-enable their accounts.

The second half has a defect: it re-uses the list of people starting leave instead of the people returning, so someone going on leave is disabled and then immediately re-enabled in the same run, while people coming back are never re-enabled automatically. The account name is also passed as "First Last", which is not the format the AD interface expects. Errors for individual people are silently ignored.

In Boomi this is a small scheduled process that runs two HCM queries and calls the AD update sub-process for each person; TaskUs must confirm the intended behaviour and the account-name format before it is built.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | AD Maternity Leave Toggle Sch | proc-maternity-leave-ad-sch | process | proc-sub-otbi-query, proc-ad-update-ou, op-xxts-set-lastrun, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/BPEL/ProcessEmpMaternityLeave.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/Taskus_Querys.sql

## Open questions

### Still outstanding

REVIEW-NEEDED: Should AD accounts be disabled on the first day of maternity leave and re-enabled the day after it ends? (Two defects: the end-of-leave branch iterates over the start-of-leave result, so people starting leave are immediately reactivated and people returning are never reactivated; and CN is passed as "First Last", while ad-update-ou expects the initials+EID form. Confirm the intended behaviour and the CN format.)
  - If this is not answered: Disable on the start date, re-enable the day after the end date, using the CN format ad-update-ou expects.

REVIEW-NEEDED: Are Maternity Caesarean, Maternity Miscarriage and Maternity Normal still the complete list of absence types that should trigger this? (The list is hardcoded.)
  - If this is not answered: Keep these three, configurable.

REVIEW-NEEDED: Who should be told when an account cannot be disabled or re-enabled? (Failures are silently ignored today.)
  - If this is not answered: Email the IT service desk list.

