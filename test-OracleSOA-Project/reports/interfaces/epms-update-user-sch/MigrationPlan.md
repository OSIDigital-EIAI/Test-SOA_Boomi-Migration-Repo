# Migration Plan — ProcessUpdateEPMSUser_sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.542Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessUpdateEPMSUser_sch |
| Identifier | epms-update-user-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | complex (72) |
| Components to build | 3 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/BPEL/ProcessUpdateEPMSUser_sch.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/ProcessUpdateEPMSUser_sch_cfgplan_Dev.xml |
| Depends on | epms-create-user-sch, newhire-emp-sch, emp-campaign-extract-sch, emp-campaign-change-sch, hcm-otbi-sql-query, util-send-notification |

## 2. End-to-End Process Architecture

Scheduler-started BPEL composite: XXTS window, two REST Atom references, HCM FlowActions/PayrollProcessing SOAP services, async OTBI provider, XSLT change detection and mapping, File adapter and embedded Java lftp upload, XXSendNotification.

Boomi scheduled process "EPMS Employee Update Sch" (proc-epms-update-user-sch): window sub-process, HTTP Client GETs on the HCM Atom feeds, optional HCM flow status/extract calls, fn-epms-build-change-list, per-person hcm-otbi-sql-query, map-hcm-to-epms-gt / map-hcm-to-epms into prof-epms-profile, combine, op-epms-put on conn-epms-ftp (SFTP), notifications through util-send-notification. Per-person errors should be collected and reported rather than swallowed.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| XXTS window | Sub: Schedule Window Get and Advance (newhire-emp-sch) |  |
| REST Atom references | HTTP Client connector to HCM atomservlet | Connection shared with other feed-based interfaces. |
| FlowActions / PayrollProcessing services | As in emp-campaign-change-sch |  |
| xform_EMPList + remove_duplicates | fn-epms-build-change-list |  |
| xform_HCM_EPMS(_GT) | map-hcm-to-epms-gt / map-hcm-to-epms |  |
| File adapter + Java lftp | op-epms-put on conn-epms-ftp |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Scheduled start and window | Direct binding; read start/end from XXTS_SCHEDULING; format as ISO with .000Z. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/BPEL/ProcessUpdateEPMSUser_sch.bpel |
| 2 | Read assignment feed | GET empassignment?updated-min&updated-max (Atom). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/WADLs/EmpAssignmentFeed.wadl |
| 3 | Read employee update feed | GET empupdate with the same window via HTTP properties. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/WADLs/EmployeeUpdateFeed.wadl |
| 4 | Campaign changes (first runs of the day) | If window start date = today: flow task status for SOA_<date>/Camapign/PH; if COMPLETED fetch extract output. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_FlowActionsService.xsl |
| 5 | Build change list | xform_EMPList conditions then remove duplicates (keep last). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_EMPList.xsl |
| 6 | Per person: query HCM | Assignment (63 columns), work relationship, person type by person number. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/BPEL/ProcessUpdateEPMSUser_sch.bpel |
| 7 | Map record | Global Transfer -> GT map, else standard map; Active rows only. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl |
| 8 | Write and upload file | UpdateEmployeeEPMS_<ts>.csv to EPMS folder, then lftp upload to /HCM/PROD; email on failure. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/SCA-INF/src/com/osi/FTPFileTransfer_Class.java |
| 9 | Advance window | Update start date and last run; callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/BPEL/ProcessUpdateEPMSUser_sch.bpel |

## 4. Payload Profiles

> **Not available.** No profile components are owned by this interface; it uses prof-epms-profile from epms-create-user-sch.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | List/PersonNumber, content="Employee Assignment Feed" | empassignment entry content PersonNumber | conditional | "Person Assignment Updated" with AssignmentStatusType new INACTIVE or ACTIVE and ActionCode new != TERMINATION; Updated/Corrected with OrganizationId change; Updated/Corrected with ActionCode LOCATION_CHANGE, MANAGER_CHANGE or JOB_CHANGE; "Person Assignment Created" with ActionCode REHIRE; "Person Manager Updated" with ManagerId change | substring parsing of the JSON text in Atom content | high |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_EMPList.xsl | Atom feeds and campaign result to change list |
| MR-002 | List/PersonNumber, content="Employee Update Feed" | empupdate entry content PersonNumber | conditional | "Person Name Corrected" or "Person Name Updated" |  | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_EMPList.xsl | Atom feeds and campaign result to change list |
| MR-003 | List/PersonNumber, content="Campaign Change" | New_Campaign/Assignment_Info/Worker_Information/Person_Number | passthrough | Campaign flow COMPLETED |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_EMPList.xsl | Atom feeds and campaign result to change list |
| MR-004 | List/PersonNumber, content="Global Transfer" | empassignment entry | conditional | "Person Assignment Created" with ActionCode GLB_TRANSFER |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_EMPList.xsl | Atom feeds and campaign result to change list |
| MR-005 | List (unique) | List | calculated |  | Drop an entry if a later entry has the same PersonNumber | low | Global Transfer entries are emitted last, so they win. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_remove_duplicates.xsl | Atom feeds and campaign result to change list |
| MR-006 | All EPMS profile fields except the four DEV-only fields | Assignment, work relationship, person type rowsets | passthrough | Column18 Assignment Status Type = Active | Same column mapping as epms-create-user-sch MAP-001 | medium | Referenced, not re-derived. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI rows to EPMS profile (update) |
| MR-007 | JobCode | Assignment Column59 | passthrough |  |  | low | Update only. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI rows to EPMS profile (update) |
| MR-008 | ADPCompanyCode / FileNumber / ADPPositionID | Column62, Column11, Column63, Column38 | conditional | Column63 = US | MX4 / 6PZ / AK0 rule; substring(PersonNumber,2); code + FileNumber | medium | Same rule as epms-create-user-sch. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI rows to EPMS profile (update) |
| MR-009 | (Global Transfer variant) | same | passthrough | List content = Global Transfer | xform_HCM_EPMS_GT; identical field mapping, Employees root always emitted | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS_GT.xsl | OTBI rows to EPMS profile (update) |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Qualifying changes | Only the listed assignment, manager, name, rehire, global-transfer and campaign events send a person to EPMS. | High | fn-epms-build-change-list. |
| BR-002 | One record per person | A person appears once per file, current data re-read from HCM. | High | Dedup in script; then per-person OTBI. |
| BR-003 | Campaign changes after flow completion | Campaign changes are included only when the SOA_<date> campaign flow is COMPLETED and the window starts today. | Medium | Call HCM flow status; consider tracking whether the day's output was already sent. |
| BR-004 | Active only | Persons whose assignment is not Active produce no record. | Medium | Decision before map. |
| BR-005 | ADP codes for US | As epms-create-user-sch BR-002. | High | Shared cross reference. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding / SOAP async | processupdateepmsuser_sch_DirectBinding / _sch_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/composite.xml |
| outbound REST GET (Atom) | https://eetm.hcm.us6.oraclecloud.com/hcmCoreApi/atomservlet/employee/empassignment?updated-min={}&updated-max={} | Authorization header from Credentials.dvm [redacted] | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/WADLs/EmpAssignmentFeed.wadl |
| outbound REST GET (Atom) | https://eetm.hcm.us6.oraclecloud.com/hcmCoreApi/atomservlet/employee/empupdate?updated-min={}&updated-max={} | Authorization header from Credentials.dvm [redacted] | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/WADLs/EmployeeUpdateFeed.wadl |
| outbound SOAP | oramds:/apps/Provider/HCM/FlowActionsService_wsdl.wsdl | Authorization header from Credentials.dvm [redacted] | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/composite.xml |
| outbound SOAP | oramds:/apps/Provider/HCM/PayrollProcessingActionService.wsdl | Authorization header from Credentials.dvm [redacted] | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/composite.xml |
| outbound SOAP async (composite) | oramds:/apps/Provider/HCM/processhcmextractprovabcs_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/composite.xml |
| outbound File (SOA server) | /u01/data/domains/SOACS-Pr_domain/EPMSFolder/UpdateEmployeeEPMS_<ts>.csv | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Domain.dvm |
| outbound FTP via shell lftp | IP-addressed host [redacted], directory /HCM/PROD | user/password hardcoded in Java [redacted] | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/SCA-INF/src/com/osi/FTPFileTransfer_Class.java |
| File EPMS_FileWrite | Design-time PhysicalDirectory is the Te_domain EPMSFolder and name epmsupdate.csv; both overridden at runtime (Domain.dvm EPMS, UpdateEmployeeEPMS_<ts>.csv); Append false; NXSD EPMSWrite.xsd. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Adapters/EPMS_FileWrite_file.jca |
| REST references EmpAssignmentFeed / EmployeeUpdateFeed | Content-Type and Accept application/atom+xml; Connection close. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/BPEL/ProcessUpdateEPMSUser_sch.bpel |
| DB adapters XXTS_SCHEDULING | Select, update start, update last run; jca.retry.count 4. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/composite.xml |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| epms-create-user-sch | EPMS connection, operation, profile and map | synchronous |
| newhire-emp-sch | Sub: Schedule Window Get and Advance | synchronous |
| emp-campaign-extract-sch | Campaign flow submission (SOA_<date>) | asynchronous |
| emp-campaign-change-sch | Campaign flow status / extract pattern | synchronous |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/SCA-INF/src/com/osi/FTPFileTransfer_Class.java | lftp put -E (removes local file on success) with hardcoded account; returns success if the local file no longer exists. | SFTP connector send (conn-epms-ftp / op-epms-put from epms-create-user-sch). | High |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_EMPList.xsl | Change detection by string-parsing Atom entry JSON content. | fn-epms-build-change-list (Groovy or JSON profile + map). | High |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_remove_duplicates.xsl | Keep last entry per PersonNumber. | Part of fn-epms-build-change-list. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS_GT.xsl | Same mapping as xform_HCM_EPMS; only the placement of the Active test differs. | map-hcm-to-epms-gt (or reuse map-hcm-to-epms). | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Process catchAll with EmailDL notification; FTP result check with email; per-person and campaign scopes swallow errors. |
| Retries | DB adapter retries only. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Utilities/XSD/EPMSWrite.xsd (PROD_MDS) | mds | yes |
| Domain.dvm | dvm | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud Atom feeds | HTTPS REST | https://eetm.hcm.us6.oraclecloud.com/hcmCoreApi/atomservlet/employee | TaskUs HCM |
| EPMS FTP server | FTP (lftp) | IP-addressed host [redacted], /HCM/PROD | TaskUs EPMS |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | HCM feed host | eetm.hcm.us6.oraclecloud.com | [redacted].xml |
| DEV | HCM feed host | ebly-dev1.hcm.us2.oraclecloud.com | [redacted].xml |
| PROD | EPMS folder | [redacted] | [redacted].dvm |

## 14. Boomi Migration Readiness Summary

This job keeps EPMS up to date with employee changes made in HCM. On each run it asks HCM which employees changed since the last run (status, department, location, manager, job, name, rehire or global transfer) and, once the day's campaign-change process has completed in HCM, also picks up everyone whose campaign changed. Each person appears once in the list. For each one it re-reads their current details from HCM and fills the same EPMS profile record as the new-hire job, including the US ADP company code, file number and position ID.

The resulting file is uploaded to the EPMS server by FTP using a shell command with a hardcoded account; if the upload fails two named individuals are emailed. Errors for an individual person are silently skipped, so a person can be missed without anyone being told.

Two things need confirming before the Boomi build: how Boomi should reach EPMS (the current server is addressed by IP with plain FTP; SFTP is preferred) and how often the job really runs (every 30 minutes per the scheduling table, or twice a day per the spreadsheet). In Boomi this becomes a scheduled process that reads the HCM feeds, builds the change list in a script or map step, reuses the EPMS profile and map from the Create interface and sends the file over SFTP.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | EPMS Change List Builder Script | fn-epms-build-change-list | transform.function | — |
| 2 | Map \| OTBI Assignment to EPMS Profile Global Transfer | map-hcm-to-epms-gt | transform.map | prof-otbi-rowset, prof-epms-profile |
| 3 | EPMS Employee Update Delta Sch | proc-epms-update-user-sch | process | proc-sub-schedule-window, op-hcm-atom-feed-get, fn-atom-content-parse, op-hcm-flow-task-status, op-hcm-fetch-extract, fn-epms-build-change-list, proc-sub-otbi-query, map-hcm-to-epms, map-hcm-to-epms-gt, op-epms-put, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/BPEL/ProcessUpdateEPMSUser_sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/ProcessUpdateEPMSUser_sch_cfgplan_Dev.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/SCA-INF/src/com/osi/FTPFileTransfer_Class.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Adapters/EPMS_FileWrite_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_EMPList.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_remove_duplicates.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS_GT.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_FlowActionsService.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/Transformations/xform_payrollProcessingService.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/WADLs/EmpAssignmentFeed.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessUpdateEPMSUser_sch/SOA/WADLs/EmployeeUpdateFeed.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/XSD/EPMSWrite.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Domain.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/Taskus_Querys.sql
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/BPEL/ProcessCreateEPMSUser_sch.bpel

## Open questions

### Still outstanding

REVIEW-NEEDED: Which EPMS server and transfer protocol should Boomi use to deliver the employee update file? (The EPMS FTP host is an IP address and its credentials are hardcoded in FTPFileTransfer_Class.java (redacted). Confirm the EPMS endpoint and protocol (SFTP preferred) for Boomi; the account itself will be configured directly in Boomi, not through this questionnaire.)
  - If this is not answered: SFTP to the existing EPMS server, directory /HCM/PROD.

REVIEW-NEEDED: How often should the EPMS update job run? (Confirm the run frequency: the spreadsheet says 23:00/11:00 UTC, while the scheduling table row says every 30 minutes between 14:00 and 23:50.)
  - If this is not answered: Every 30 minutes, 14:00–23:50, as in the scheduling table.

REVIEW-NEEDED: Should employees from the daily campaign-change process be sent to EPMS only once per day? (Today every run whose window starts on the current date re-reads the completed campaign output, so the same people can be sent several times a day.)
  - If this is not answered: Send campaign changes once per day, on the first run after the campaign flow completes.

REVIEW-NEEDED: Who should be told when an individual employee cannot be sent to EPMS or the upload fails? (Per-person errors are silently skipped today and upload failures go to two named individuals.)
  - If this is not answered: Report both to the EmailDL EPMSJOB list.

