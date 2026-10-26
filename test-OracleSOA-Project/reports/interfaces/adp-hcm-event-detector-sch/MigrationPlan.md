# Migration Plan — ProcessNewHireEmpBIReq_Sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.591Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessNewHireEmpBIReq_Sch |
| Identifier | adp-hcm-event-detector-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | medium (40) |
| Components to build | 2 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/BPEL/ProcessNewHireEmpBIReq_Sch.bpel |
| Depends on | adp-employee-masterdata, newhire-emp-sch, hcm-otbi-sql-query, hcm-dat-generation |

## 2. End-to-End Process Architecture

Scheduler-started BPEL with XXTS DB adapters, skipped REST Atom feeds and new-hire query, one async OTBI termination query, XSLT to EmployeeEBM and an async call to ProcessEmployeeADPProvABCS.

Boomi scheduled process "ADP HCM Event Detector Sch" (proc-adp-hcm-event-detector-sch): window sub-process (newhire-emp-sch), hcm-otbi-sql-query for the agreed events (plus HCM Atom feeds if re-enabled), map-events-to-adp-request, Decision on count, Process Call proc-adp-employee-masterdata, then advance the window.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| XXTS window (update skipped) | Sub: Schedule Window Get and Advance | Advance after success. |
| Async OTBI termination query | Process Call hcm-otbi-sql-query |  |
| xform_hcm_to_adp | map-events-to-adp-request | Fix AdjustedServiceDate. |
| Async call to ProcessEmployeeADPProvABCS | Process Call proc-adp-employee-masterdata |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Start and read window | SOAP or direct binding; XXTS window to updatedMin/updatedMax. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/BPEL/ProcessNewHireEmpBIReq_Sch.bpel |
| 2 | Read HCM feeds (disabled) | empupdate and empassignment Atom feeds — skipped. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/BPEL/ProcessNewHireEmpBIReq_Sch.bpel |
| 3 | New-hire query (disabled) | Skipped; contains a termination-date query. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/BPEL/ProcessNewHireEmpBIReq_Sch.bpel |
| 4 | Find terminations | OTBI Actual Termination Date between window dates. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/BPEL/ProcessNewHireEmpBIReq_Sch.bpel |
| 5 | Build employee request | xform_hcm_to_adp. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/Transformations/xform_hcm_to_adp.xsl |
| 6 | Call ADP master data | If at least one employee. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/BPEL/ProcessNewHireEmpBIReq_Sch.bpel |
| 7 | Advance window (disabled) | Scheduler update skipped; client callback sent. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/BPEL/ProcessNewHireEmpBIReq_Sch.bpel |

## 4. Payload Profiles

> **Not available.** No profile component is owned by this interface; it uses the OTBI rowset profile of hcm-otbi-sql-query and the employee request of adp-employee-masterdata.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | EmployeeID | Terminate rowset Column5 (Person Number) | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/Transformations/xform_hcm_to_adp.xsl | HCM events to ADP employee request |
| MR-002 | EffectiveStartDate | Column6 (Termination Date) | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/Transformations/xform_hcm_to_adp.xsl | HCM events to ADP employee request |
| MR-003 | LastReviewedDate | Column3 (Last Update Date) | calculated |  | replace / with -, add 1 day, date part | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/Transformations/xform_hcm_to_adp.xsl | HCM events to ADP employee request |
| MR-004 | AdjustedServiceDate | $Terminate_RowSetVar/rowset/Row/Column6 (absolute) | passthrough |  |  | low | Defect: first row's value for every employee. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/Transformations/xform_hcm_to_adp.xsl | HCM events to ADP employee request |
| MR-005 | CustomAttributes/Description | (constant per source) | hardcoded | terminations | "terminate"; disabled sources: newhire, namechange, taxdependentschange, rehire | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/Transformations/xform_hcm_to_adp.xsl | HCM events to ADP employee request |
| MR-006 | EmployeeID / EffectiveStartDate (disabled feeds) | Atom content PersonNumber / EffectiveDate | conditional | "Person Name Corrected/Updated" -> namechange; "Person Legislative Information Updated" with MaritalStatus -> taxdependentschange; "Person Assignment Created" with ActionCode REHIRE -> rehire | substring parsing | medium | Inactive while the feed scopes are skipped. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/Transformations/xform_hcm_to_adp.xsl | HCM events to ADP employee request |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Terminations in window | Work relationships with Actual Termination Date between window start and end dates. | High | OTBI query via sub-process. |
| BR-002 | Call only when needed | ADP master-data service called only when at least one employee was found. | Medium | Decision on document count. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding / SOAP async | ProcessNewHireEmpBIReqDirectBinding / processnewhireempbireq_sch_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/composite.xml |
| outbound SOAP async (composite) | oramds:/apps/Provider/HCM/processhcmextractprovabcs_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/composite.xml |
| outbound SOAP async (composite) | oramds:/apps/Provider/ADP/processemployeeadpprovabcs_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/composite.xml |
| outbound REST GET Atom (disabled) | <HCM host>/hcmCoreApi/atomservlet/employee (empupdate, empassignment) | not determinable from the disabled scopes | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/ProcessNewHireEmpBIReq_Sch_Prod_cfgplan.xml |
| DB XXTS_SCHEDULING select/update | Interface ProcessNewHireEmpBIReq_Sch; update calls skipped. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/BPEL/ProcessNewHireEmpBIReq_Sch.bpel |
| Read_HCMExtractFile (FTP Get_ptt) service | Declared on the component only; no composite-level binding, so it is not an active entry point. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/composite.xml |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| adp-employee-masterdata | ADP Employee Master Data File | asynchronous |
| newhire-emp-sch | Sub: Schedule Window Get and Advance | synchronous |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| hcm-dat-generation | Shared EmployeeEBM request contract | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/Transformations/xform_hcm_to_adp.xsl | OTBI rows and Atom entries to EmployeeRequest with action codes. | map-events-to-adp-request. | Medium |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | None — no catch or catchAll. |
| Retries | DB adapter retries only. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/ADP/processemployeeadpprovabcs_client_ep.wsdl | mds | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud Atom feeds (disabled) | HTTPS REST | eetm.hcm.us6.oraclecloud.com (PROD plan) | TaskUs HCM |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| DEV | HCM feed host | ebly-dev1.hcm.us2.oraclecloud.com | [redacted].xml |
| PROD | HCM feed host | eetm.hcm.us6.oraclecloud.com | [redacted].xml |

## 14. Boomi Migration Readiness Summary

This job is the trigger that decides which employees go into the ADP master-data file. It was designed to pick up new hires, rehires, terminations, and name and marital-status changes from HCM. In the exported version only the termination check is switched on: each run asks HCM for employees whose termination date falls in the scheduler window and, if there are any, sends them to the ADP file builder as terminations.

The step that moves the scheduler window forward is also switched off, so every run looks at the same dates and would send the same terminations again. The new-hire check that is switched off actually contains a termination query too, so it could not simply be re-enabled.

Before the ADP interfaces can be scoped TaskUs must decide which employee events ADP needs and how they should be detected. In Boomi this becomes a scheduled process using the shared window and OTBI sub-processes, a small map to the employee request and a call to the ADP master-data sub-process.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Map \| HCM Events to ADP Employee Request | map-events-to-adp-request | transform.map | prof-otbi-rowset, prof-employee-ebm-xml |
| 2 | ADP HCM Event Detector Sch | proc-adp-hcm-event-detector-sch | process | proc-sub-schedule-window, proc-sub-otbi-query, op-hcm-atom-feed-get, map-events-to-adp-request, proc-adp-employee-masterdata |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/BPEL/ProcessNewHireEmpBIReq_Sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/Transformations/xform_hcm_to_adp.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/ProcessNewHireEmpBIReq_Sch_Dev_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/ProcessNewHireEmpBIReq_Sch_Prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/Taskus_Querys.sql
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/WSDLs/ProcessEmployeeADPProvABCS.wsdl

## Open questions

### Still outstanding

REVIEW-NEEDED: Which employee events must be sent to ADP — new hires, rehires, terminations, name changes, marital-status changes, transfers — and how should each be detected? (Only terminations reach ADP from this detector; new hires, rehires, name and marital-status changes are disabled. The window is never advanced, so every run re-reads the same window. Confirm which events must be sent to ADP and by which trigger. Without that, adp-employee-masterdata cannot be scoped.)
  - If this is not answered: Send new hires, rehires, terminations, name and marital-status changes, detected from HCM each run, with the window advanced after each successful run.

REVIEW-NEEDED: How often should employee events be sent to ADP? (No schedule for this job is in the export.)
  - If this is not answered: Once a day.

