# Migration Plan — ProcessEmpCampaignExtract_Sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.817Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessEmpCampaignExtract_Sch |
| Identifier | emp-campaign-extract-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | simple (18) |
| Components to build | 6 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/BPEL/ProcessEmpCampaignExtract_Sch.bpel |
| Depends on | newhire-emp-sch |

## 2. End-to-End Process Architecture

BPEL composite started by the XXTS_SCHEDULING servlet. It reads its window with the DB adapter, transforms the input into a submitFlow request with an XSLT, and calls the HCM FlowActionsService through a WSDL held in MDS, passing an HTTP Authorization header taken from Credentials.dvm. It then advances its window and calls back.

Scheduled Boomi process "Employee Campaign Extract Submit Sch" using the shared "Sub: Schedule Window Get and Advance" (newhire-emp-sch), map-campaign-submitflow into prof-hcm-flow-submit-req, and op-hcm-flow-submit on conn-hcm-flowactions-soap. Add a Try/Catch that notifies support (util-send-notification) and a check of the submitFlow response. This interface also owns op-hcm-flow-task-status, used by emp-campaign-change-sch and epms-update-user-sch.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| SchedulerServlet + XXTS DB adapters | Process schedule + proc-sub-schedule-window |  |
| xform_flowActionService.xsl | map-campaign-submitflow |  |
| FlowActionsService reference (MDS WSDL) + Authorization header | conn-hcm-flowactions-soap + op-hcm-flow-submit | Authentication method to verify. |
| catchAll without notification | Try/Catch + util-send-notification | New behaviour. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Start and read window | Direct binding from the scheduler; Select_XXTS_Scheduling for INTERFACE_NAME ProcessEmpCampaignExtract_Sch; start_date_time formatted yyyy-MM-ddTHH:mm:ss; DateTime = date part; .000Z appended to start_date_time. (SOAP client entry uses the caller's DateTime.) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/BPEL/ProcessEmpCampaignExtract_Sch.bpel |
| 2 | Build submitFlow request | xform_flowActionService.xsl: flowName "Camapign", parameter "Effective Date" = DateTime, parameter "Changes only" = "Y", flowInstanceName = "SOA_" + current-date(), legislativeDataGroupName "PH", recurringFlag false. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Transformations/xform_flowActionService.xsl |
| 3 | Submit HCM flow | FlowActionsService.submitFlow (SOAP 1.1, synchronous) with HTTP header Authorization from Credentials.dvm. The response is stored but not inspected. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/BPEL/ProcessEmpCampaignExtract_Sch.bpel |
| 4 | Advance window | Direct binding: Update_XXTS_Schedule (last_scheduled_time = now) then Update_Scheduling_StartDateTime (start + INTERVAL + 1 s), then callback. SOAP client: callback only. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/BPEL/ProcessEmpCampaignExtract_Sch.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| HCM Flow SUBMIT Req XML Prof | xml | submitFlow/flowName | string | yes |
| HCM Flow SUBMIT Req XML Prof | xml | submitFlow/parameterValues/ParameterName | string | no |
| HCM Flow SUBMIT Req XML Prof | xml | submitFlow/parameterValues/ParameterValue | string | no |
| HCM Flow SUBMIT Req XML Prof | xml | submitFlow/flowInstanceName | string | yes |
| HCM Flow SUBMIT Req XML Prof | xml | submitFlow/legislativeDataGroupName | string | no |
| HCM Flow SUBMIT Req XML Prof | xml | submitFlow/recurringFlag | boolean | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | submitFlow/flowName | (none) | hardcoded |  | "Camapign" | low | Spelling must match the HCM flow. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Transformations/xform_flowActionService.xsl | Schedule window to HCM submitFlow |
| MR-002 | parameterValues[1] ParameterName / ParameterValue | process/DateTime | transformed |  | name "Effective Date"; value = window start date (yyyy-MM-dd, from substring-before(start_date_time,"T")) | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/BPEL/ProcessEmpCampaignExtract_Sch.bpel | Schedule window to HCM submitFlow |
| MR-003 | parameterValues[2] ParameterName / ParameterValue | (none) | hardcoded |  | "Changes only" = "Y" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Transformations/xform_flowActionService.xsl | Schedule window to HCM submitFlow |
| MR-004 | submitFlow/flowInstanceName | (current date) | calculated |  | concat("SOA_", xp20:current-date()) | low | xp20:current-date may carry a timezone suffix depending on server settings; the exact string must equal what emp-campaign-change-sch builds. Input element flowInstanceName is ignored. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Transformations/xform_flowActionService.xsl | Schedule window to HCM submitFlow |
| MR-005 | submitFlow/legislativeDataGroupName | (none) | hardcoded |  | "PH" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Transformations/xform_flowActionService.xsl | Schedule window to HCM submitFlow |
| MR-006 | submitFlow/recurringFlag | (none) | hardcoded |  | "false" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Transformations/xform_flowActionService.xsl | Schedule window to HCM submitFlow |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Daily campaign extract submission | Submit flow "Camapign" for LDG PH once per run with Effective Date = window start date and Changes only = Y. | High | Scheduled process + map-campaign-submitflow + op-hcm-flow-submit. |
| BR-002 | Instance naming contract | Flow instance is named "SOA_" + current date so that emp-campaign-change-sch can find it later the same day. | High | Build the name with a shared date format (DPP) used by both processes. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding / SOAP async | processempcampaignextract_sch_DirectBinding / processempcampaignextract_sch_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/composite.xml |
| outbound SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmProcFlowCoreController/FlowActionsService | HTTP Authorization header (value from Credentials.dvm); service WSDL advertises wss11 SAML-or-username-token policies | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/FlowActionsService_wsdl.wsdl |
| outbound Database | eis/DB/SOADemoLocalTx, table XXTS_SCHEDULING | WebLogic data source | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |
| DB adapter Select_XXTS_Scheduling | Pure SQL select of start_date_time and start+INTERVAL for INTERFACE_NAME; retry 4/1/2/120. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |
| DB adapter Update_Scheduling_StartDateTime | start_date_time = start + (INTERVAL+1)/86400 for INTERFACE_NAME. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Adapters/Update_Scheduling_StartDateTime_db.jca |
| DB adapter Update_XXTS_Schedule | last_scheduled_time = now for interface_name. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca |
| SOAP reference FlowActionsService | binding.ws to oramds:[redacted].wsdl, SOAP 1.1, property oracle.webservices.http.headers=Authorization, WSDLDriven transaction flow. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/composite.xml |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| newhire-emp-sch | Sub: Schedule Window Get and Advance (shared) | synchronous |
| util-send-notification | Sub: Send Notification (new error handling) | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Transformations/xform_flowActionService.xsl | Builds the submitFlow request with hardcoded flow name, parameters and LDG. | map-campaign-submitflow | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Process catchAll advances the window and calls back; no notification, no rethrow. |
| Retries | DB adapter retries only (4, backoff 2, max 120 s). |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/HCM/FlowActionsService_wsdl.wsdl | mds | yes |
| FlowActionsService types XSD (imported from the live HCM URL ?XSD=...FlowActionsService.xsd) | other | no |
| Credentials.dvm (Authorization) | dvm | yes |
| XXTS_SCHEDULING row for ProcessEmpCampaignExtract_Sch | other | no |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud FlowActionsService | SOAP 1.1 / HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmProcFlowCoreController/FlowActionsService | Oracle (TaskUs tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | FlowActionsService | https://eetm.hcm.us6.oraclecloud.com:[redacted] | [redacted].wsdl |

## 14. Boomi Migration Readiness Summary

This small job starts a payroll flow in Oracle HCM once a day. The flow, named "Camapign" in HCM (the spelling is in the source and must match HCM exactly), runs for the Philippines legislative data group with two parameters: the effective date (the start date of the job's time window) and "Changes only = Y". The flow instance is named "SOA_" followed by the current date, which is how the follow-on campaign-change job finds it an hour later.

The job does not check whether HCM accepted the request, and if anything fails it sends no notification: it simply moves its time window on. In Boomi it becomes a scheduled process with one SOAP call to HCM, a simple map, and added error handling.

One point needs checking during the build: the HCM web service advertises WS-Security (SAML or username token) policies, while the legacy job authenticates with an HTTP Authorization header. The Boomi connection must use whichever method HCM accepts today.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | HCM Flow SUBMIT Req XML Prof | prof-hcm-flow-submit-req | profile.xml | — |
| 2 | HCM FlowActionsService SOAP Conn | conn-hcm-flowactions-soap | connector-settings | — |
| 3 | HCM Flow SUBMIT Op | op-hcm-flow-submit | connector-action | conn-hcm-flowactions-soap |
| 4 | HCM Flow Task Status GET Op | op-hcm-flow-task-status | connector-action | conn-hcm-flowactions-soap |
| 5 | Map \| Schedule Window to HCM Campaign SubmitFlow | map-campaign-submitflow | transform.map | prof-hcm-flow-submit-req |
| 6 | Employee Campaign Extract Submit Sch | proc-emp-campaign-extract-sch | process | proc-sub-schedule-window, op-hcm-flow-submit, map-campaign-submitflow |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/BPEL/ProcessEmpCampaignExtract_Sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Transformations/xform_flowActionService.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Schemas/ProcessEmpCampaignExtract_Sch.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Adapters/Update_Scheduling_StartDateTime_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/FlowActionsService_wsdl.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Credentials.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/composite.xml

## Open questions

### Still outstanding

REVIEW-NEEDED: Is "Camapign" (spelt this way) still the exact name of the HCM payroll flow that produces the campaign-change extract, and is "PH" the right legislative data group? (The flow name and data group are hardcoded and must match HCM exactly or the submission fails.)
  - If this is not answered: Keep flow name "Camapign" and legislative data group "PH" as in the current code.

REVIEW-NEEDED: If HCM rejects the daily campaign flow submission, who should be notified? (Today a failure is silent and the follow-on AD campaign job simply finds no completed flow.)
  - If this is not answered: Email the integration "notification" distribution list.

REVIEW-NEEDED: Which HCM integration account should the migrated job use to submit HCM flows? (The current account details are held in a shared lookup table and must be configured on the Boomi connection.)
  - If this is not answered: Use the existing HCM integration account with a newly rotated password.

REVIEW-NEEDED: Should the campaign extract keep running only once per day, and only for Philippine workers? (The flow instance name is date-based, so more frequent runs would collide; the data group is PH only.)
  - If this is not answered: Once per day, Philippines only.

