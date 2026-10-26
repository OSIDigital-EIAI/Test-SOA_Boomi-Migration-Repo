# Migration Plan — ProcessEmpCampaignChange_Sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.831Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessEmpCampaignChange_Sch |
| Identifier | emp-campaign-change-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | medium (50) |
| Components to build | 7 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel |
| Depends on | emp-campaign-extract-sch, hcm-otbi-sql-query, ad-update-ou, util-file-notification, util-send-notification, newhire-emp-sch |

## 2. End-to-End Process Architecture

BPEL composite started by the XXTS_SCHEDULING servlet. Calls HCM FlowActionsService and PayrollProcessingActionService through MDS WSDLs with an HTTP Authorization header, parses the escaped extract result, loops over workers calling ProcessHCMExtractSyncProv and ProcessUpdateOUADUser (waiting for the callback), appends CSV rows with the File adapter to the EmailAttachment folder and calls FileNotification.

Scheduled Boomi process "Employee Campaign Change AD Sch": op-hcm-flow-task-status (owned by emp-campaign-extract-sch) then a Decision on COMPLETED; op-hcm-fetch-extract on conn-hcm-payroll-soap; a Groovy/XML step to unescape the result into prof-campaign-extract-result; split by Assignment_Info with Flow Control one at a time; per worker in a Try/Catch call hcm-otbi-sql-query and ad-update-ou synchronously; map rows into prof-campaign-report-csv; Combine and send with util-file-notification. The window is handled by the shared schedule sub-process.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| FlowActionsService.getFlowTaskInstanceStatus | op-hcm-flow-task-status (emp-campaign-extract-sch) |  |
| PayrollProcessingActionService.fetchExtractOutput + oraext:parseXML | op-hcm-fetch-extract + unescape step + prof-campaign-extract-result |  |
| ProcessUpdateOUADUser invoke + receive | map-campaign-worker-to-ouad + synchronous Process Call ad-update-ou | Add a time limit. |
| File adapter append + FileNotification | map-campaign-result-to-report + Combine + util-file-notification | In memory. |
| Per-worker catchAll | Try/Catch per document |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Start and read window | Direct binding from the scheduler; Select_XXTS_Scheduling for ProcessEmpCampaignChange_Sch; start_date_time formatted and .000Z appended (used only for the window update). The SOAP client path skips this. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel |
| 2 | Check today's flow status | FlowActionsService.getFlowTaskInstanceStatus with flowInstanceName "SOA_"+current-date(), flowTaskInstanceName "Camapign", legislativeDataGroupName "PH"; HTTP Authorization header from Credentials.dvm. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Transformations/xform_FlowActionsService.xsl |
| 3 | Completed? | Continue only if result = "COMPLETED"; otherwise go to the window update. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel |
| 4 | Fetch campaign extract | PayrollProcessingActionService.fetchExtractOutput (instanceName SOA_<date>, taskName "Camapign", mode "FLOW"); result string parsed with oraext:parseXML into New_Campaign. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Transformations/xform_payrollProcessingService.xsl |
| 5 | Any workers? | If count(Assignment_Info) > 0: FileName = CampaignChange_<yyyy-MM-ddTHH:mm:ss>.csv and header row written to AttachmentPath. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel |
| 6 | For each worker (sequential) | forEach parallel="no"; per-worker catchAll emails EmailDL itservicedesk with the fault and continues. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel |
| 7 | Look up job name | OTBI: SELECT Employee Mailing County Code s_1, Job Name s_2 FROM Worker Assignment Real Time WHERE Person Number = '<Person_Number>' (appended correctly with bpelx:append). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel |
| 8 | Move AD user to new campaign | ProcessUpdateOUADUser: Action "Change", CN = initials + Person_Number, Employee_ID = Person_Number, Status = Project_ID; then waits for the processResponse callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel |
| 9 | Write report row | Status "Success" if result contains "Success", else "Failure"; Message = text after "@" in result; Project Description from the AD response. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel |
| 10 | Email report | FileNotification to EmailDL itservicedesk, subject "Consolidated Notification for Campaign Change", file from Domain.dvm Email path. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel |
| 11 | Advance window | Direct path: Update_Scheduling_StartDateTime and Update_XXTS_Schedule, then callback. Client path: callback with result = first OBJECT_ACTION_ID. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| HCM Campaign Extract Result XML Prof | xml | New_Campaign/REP_CATEGORY_NAME | string | no |
| HCM Campaign Extract Result XML Prof | xml | New_Campaign/parameters (request_id, FLOW_NAME, legislative_data_group_id, effective_date, start_date, report_category_id, action_parameter_group_id) | string | no |
| HCM Campaign Extract Result XML Prof | xml | New_Campaign/Assignment_Info | repeating | no |
| HCM Campaign Extract Result XML Prof | xml | New_Campaign/Assignment_Info/OBJECT_ACTION_ID | string | no |
| HCM Campaign Extract Result XML Prof | xml | New_Campaign/Assignment_Info/Worker_Information/Person_Number | string | no |
| HCM Campaign Extract Result XML Prof | xml | New_Campaign/Assignment_Info/Worker_Information/First_Name | string | no |
| HCM Campaign Extract Result XML Prof | xml | New_Campaign/Assignment_Info/Worker_Information/Last_Name | string | no |
| HCM Campaign Extract Result XML Prof | xml | New_Campaign/Assignment_Info/Worker_Information/Project_ID | string | no |
| Campaign Change Report CSV Prof | flatfile | Sl_No | string | no |
| Campaign Change Report CSV Prof | flatfile | Instance_ID | string | no |
| Campaign Change Report CSV Prof | flatfile | Employee_ID | string | no |
| Campaign Change Report CSV Prof | flatfile | Employee_Name | string | no |
| Campaign Change Report CSV Prof | flatfile | JobTitle | string | no |
| Campaign Change Report CSV Prof | flatfile | New_Campaign | string | no |
| Campaign Change Report CSV Prof | flatfile | Project_Description | string | no |
| Campaign Change Report CSV Prof | flatfile | Status | string | no |
| Campaign Change Report CSV Prof | flatfile | Error | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | Action | (none) | hardcoded |  | "Change" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel | Campaign worker to AD OU change request |
| MR-002 | CN | Worker_Information/First_Name, Last_Name, Person_Number | calculated |  | lower(first char First_Name)+lower(first char Last_Name)+Person_Number | medium | substring-before(x,substring(x,2)) idiom; use first character directly in Boomi. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel | Campaign worker to AD OU change request |
| MR-003 | Employee_ID | Worker_Information/Person_Number | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel | Campaign worker to AD OU change request |
| MR-004 | Status | Worker_Information/Project_ID | passthrough |  |  | low | For Action=Change the AD service treats Status as the new project ID. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel | Campaign worker to AD OU change request |
| MR-005 | Sl_No | loop counter | calculated |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel | AD result to report row |
| MR-006 | Instance_ID | flow id | calculated |  | execution id in Boomi | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel | AD result to report row |
| MR-007 | Employee_ID | Person_Number | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel | AD result to report row |
| MR-008 | Employee_Name | First_Name, Last_Name | calculated |  | First_Name + " " + Last_Name | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel | AD result to report row |
| MR-009 | JobTitle | OTBI Row[1]/Column2 (Job Name) | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel | AD result to report row |
| MR-010 | New_Campaign | Project_ID | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel | AD result to report row |
| MR-011 | Project_Description | AD response Description | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel | AD result to report row |
| MR-012 | Status | AD response result | conditional | contains(result,"Success") | "Success" / "Failure" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel | AD result to report row |
| MR-013 | Error (header "Message") | AD response result | transformed |  | substring-after(result,"@") | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel | AD result to report row |
| MR-014 | flowInstanceName / instanceName | (current date) | calculated |  | concat("SOA_", xp20:current-date()) | low | Must equal the name built by emp-campaign-extract-sch. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Transformations/xform_FlowActionsService.xsl | Flow status and extract requests |
| MR-015 | flowTaskInstanceName / taskName | (none) | hardcoded |  | "Camapign" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Transformations/xform_payrollProcessingService.xsl | Flow status and extract requests |
| MR-016 | legislativeDataGroupName / mode | (none) | hardcoded |  | "PH" / "FLOW" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Transformations/xform_payrollProcessingService.xsl | Flow status and extract requests |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Process only completed extracts | Read the extract only when today's flow instance status is COMPLETED. | High | Decision on getFlowTaskInstanceStatus result. |
| BR-002 | Campaign move in AD | Each worker in the extract is moved with Action "Change" and Status = new Project ID. | High | map-campaign-worker-to-ouad + Process Call ad-update-ou. |
| BR-003 | Success detection | An AD result containing "Success" is reported as Success; otherwise Failure, with the text after "@" as message. | Medium | map-campaign-result-to-report. |
| BR-004 | Report distribution | Email the CSV report to EmailDL itservicedesk when at least one worker was in the extract. | Medium | Combine rows + util-file-notification. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding / SOAP async | processempcampaignchange_sch_DirectBinding / processempcampaignchange_sch_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/composite.xml |
| outbound SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmProcFlowCoreController/FlowActionsService | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/FlowActionsService_wsdl.wsdl |
| outbound SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmBatchProcessCore/PayrollProcessingActionService | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/PayrollProcessingActionService.wsdl |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/composite.xml |
| outbound SOAP async (composite) | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateOUADUser/processupdateouaduser_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processupdateouaduser_client_ep.wsdl |
| outbound File (SOA server) | AttachmentPath /u01/data/domains/SOACS-Pr_domain/osi_file/EmailAttachment (also set from Domain.dvm Email path) | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Adapters/Attachment_File_Write_file.jca |
| outbound SOAP (composite) | oramds:/apps/Utilities/Notification/filenotification_client_ep.wsdl / xxsendnotification_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/composite.xml |
| outbound Database | eis/DB/SOADemoLocalTx, XXTS_SCHEDULING | WebLogic data source | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |
| File adapter Attachment_File_Write | JNDI eis/FileAdapter; LogicalDirectory AttachmentPath; FileNamingConvention "Test" overridden by jca.file.FileName; data rows also override jca.file.Directory from Domain.dvm Email path; Append=true; NXSD WriteFile.xsd (comma, double-quoted, US-ASCII, 1 header line). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Adapters/Attachment_File_Write_file.jca |
| DB adapters (XXTS_SCHEDULING) | Select window; advance start_date_time by INTERVAL+1 s; set last_scheduled_time. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| emp-campaign-extract-sch | HCM Flow Task Status GET Op (shared component) | synchronous |
| newhire-emp-sch | Sub: Schedule Window Get and Advance (shared) | synchronous |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| ad-update-ou | AD User OU MOVE (Action Change) | synchronous |
| util-file-notification | Sub: Send File Notification | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Transformations/xform_FlowActionsService.xsl | getFlowTaskInstanceStatus request (SOA_<date>, Camapign, PH). | Message/map into the FlowActions request (uses op-hcm-flow-task-status from emp-campaign-extract-sch). | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Transformations/xform_payrollProcessingService.xsl | fetchExtractOutput request (SOA_<date>, Camapign, FLOW). | Message/map into op-hcm-fetch-extract request. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Transformations/xform_WriteFileHeaders.xsl | Literal report header row. | Flat file profile header in prof-campaign-report-csv. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Per-worker catchAll (notify itservicedesk, continue) and process-level catchAll (notify itservicedesk, advance window, callback). |
| Retries | DB adapter retries only. |
| Compensation | None; failed AD moves are listed in the report for manual action. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/HCM/PayrollProcessingActionService.wsdl | mds | yes |
| oramds:/apps/Provider/HCM/FlowActionsService_wsdl.wsdl | mds | yes |
| EmailDL.dvm (itservicedesk) | dvm | yes |
| Domain.dvm (Email path) | dvm | yes |
| Credentials.dvm (Authorization) | dvm | yes |
| XXTS_SCHEDULING row for ProcessEmpCampaignChange_Sch | other | no |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud FlowActionsService | SOAP 1.1 / HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmProcFlowCoreController/FlowActionsService | Oracle (TaskUs tenant) |
| Oracle HCM Cloud PayrollProcessingActionService | SOAP 1.1 / HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmBatchProcessCore/PayrollProcessingActionService | Oracle (TaskUs tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | AttachmentPath | [redacted] | [redacted].xml |
| PROD | PayrollProcessingActionService | https://eetm.hcm.us6.oraclecloud.com:[redacted] | [redacted].wsdl |

## 14. Boomi Migration Readiness Summary

This job applies campaign moves to Active Directory. Each day it asks HCM whether today's campaign extract flow (started an hour earlier by the companion job) has completed. If it has, it fetches the list of workers whose campaign changed and, one by one, asks the AD service to move each person to the new campaign's organisational unit and groups. It waits for each answer and records it in a CSV report, which is then emailed to the IT service desk.

The job only looks at "today's" flow, identified by the name SOA_ plus the date. If the flow has not finished by the time the job runs, that day's changes are not picked up later. If the job ran twice, the same workers would be processed again. Errors for an individual worker are emailed to the service desk and the job carries on; any other error is emailed and the time window still moves on.

In Boomi this becomes a scheduled process with two HCM SOAP calls, a loop that calls the AD sub-process synchronously for each worker, and a report built in memory and emailed through the shared file-notification sub-process.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | HCM Campaign Extract Result XML Prof | prof-campaign-extract-result | profile.xml | — |
| 2 | Campaign Change Report CSV Prof | prof-campaign-report-csv | profile.flatfile | — |
| 3 | HCM PayrollProcessingActionService SOAP Conn | conn-hcm-payroll-soap | connector-settings | — |
| 4 | HCM Payroll Extract Output GET Op | op-hcm-fetch-extract | connector-action | conn-hcm-payroll-soap |
| 5 | Map \| AD Result to Campaign Report Row | map-campaign-result-to-report | transform.map | prof-campaign-report-csv |
| 6 | Map \| Campaign Worker to AD OU Change | map-campaign-worker-to-ouad | transform.map | prof-campaign-extract-result |
| 7 | Employee Campaign Change AD Sch | proc-emp-campaign-change-sch | process | proc-sub-schedule-window, op-hcm-flow-task-status, op-hcm-fetch-extract, proc-sub-otbi-query, map-campaign-worker-to-ouad, map-campaign-result-to-report, proc-sub-file-notification, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Transformations/xform_FlowActionsService.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Transformations/xform_payrollProcessingService.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Transformations/xform_WriteFileHeaders.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Schemas/Result.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Schemas/WriteFile.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Adapters/Attachment_File_Write_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Adapters/Update_Scheduling_StartDateTime_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/FlowActionsService_wsdl.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/PayrollProcessingActionService.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processupdateouaduser_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Domain.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Credentials.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignExtract_Sch/SOA/Transformations/xform_flowActionService.xsl

## Open questions

### Still outstanding

REVIEW-NEEDED: If the daily campaign extract in HCM has not finished when this job runs, should the job try again later the same day or pick it up the next day? (Today the job only looks for "today's" flow; if it is not completed at 15:00 UTC those campaign moves are never applied automatically.)
  - If this is not answered: Retry hourly until the end of the day, then report the missed extract to the service desk.

REVIEW-NEEDED: Should a worker who appears in the extract twice in one day (for example if the job is rerun) be moved again? (Rerunning today would repeat the same AD moves and report rows.)
  - If this is not answered: Process each extract only once per day.

REVIEW-NEEDED: Who should receive the daily campaign change report and the error notices? (Both go to the "itservicedesk" distribution list today.)
  - If this is not answered: Keep sending both to the current itservicedesk list.

REVIEW-NEEDED: Roughly how many campaign changes happen on a typical day and on the busiest day? (Each worker is processed one at a time with a wait for the AD result, so volume determines run time.)
  - If this is not answered: Assume up to 500 per day.

