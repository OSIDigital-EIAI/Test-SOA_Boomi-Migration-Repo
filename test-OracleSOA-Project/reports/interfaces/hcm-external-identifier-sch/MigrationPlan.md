# Migration Plan — ProcessUpdateExternalIdentifier

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.510Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessUpdateExternalIdentifier |
| Identifier | hcm-external-identifier-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | medium (38) |
| Components to build | 3 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/BPEL/ProcessUpdateExternalIdentifier.bpel |
| Depends on | newhire-emp-sch, hcm-otbi-sql-query, hcm-hdl-loader, hcm-get-hdl-errors, util-send-notification |

## 2. End-to-End Process Architecture

Scheduler-started BPEL composite: OTBI via ProcessHCMExtractSyncProv, XSLT to DAT, File adapter + Java zip, UCM and HDL SOAP calls, 5-minute wait, getHDLErrors, XXSendNotification, XXTS adapters.

Boomi scheduled process "HCM External Identifier Load Sch": schedule (or window sub-process from newhire-emp-sch), Process Call hcm-otbi-sql-query, map-rowset-to-extid-dat, Process Call hcm-hdl-loader with status polling, hcm-get-hdl-errors, notification to a distribution list.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| XXTS window | Schedule-window sub-process (newhire-emp-sch) or Boomi schedule | Use the window for selection to catch missed days. |
| ProcessHCMExtractSyncProv | Process Call hcm-otbi-sql-query |  |
| xform_Int_ExtIdDAT | map-rowset-to-extid-dat |  |
| File + Java + UCM + HDL | Process Call hcm-hdl-loader |  |
| Wait 5 min + getHDLErrors | Load status polling + hcm-get-hdl-errors |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Scheduled start | Direct binding; XXTS window read (InputDate = current date). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/BPEL/ProcessUpdateExternalIdentifier.bpel |
| 2 | Find today's US starters | OTBI query (see key points). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/BPEL/ProcessUpdateExternalIdentifier.bpel |
| 3 | Any rows? | count(Row) > 0, else nothing. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/BPEL/ProcessUpdateExternalIdentifier.bpel |
| 4 | Build ExternalIdentifier.dat | xform_Int_ExtIdDAT; File adapter writes Worker.dat. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Transformations/xform_Int_ExtIdDAT.xsl |
| 5 | Zip and load | Java ZIPtoBASE64 on /u01/data/domains/SOACS-Pr_domain/ExternalIdentifier; UCM check-in; HDL importAndLoadData. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java |
| 6 | Check errors | Wait 5 minutes; getHDLErrors (ContentId = dDocName); first MSG_TEXT. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Transformations/xfrom_gethdlError.xsl |
| 7 | Notify and finish | Error or success email; advance XXTS window and last run. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/BPEL/ProcessUpdateExternalIdentifier.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| HCM HDL ExternalIdentifier DAT Prof | flatfile | ExtIdentifier/METADATA, ExternalIdentifier | string | yes |
| HCM HDL ExternalIdentifier DAT Prof | flatfile | ExtIdentifier/PersonNumber, PersonId, AssignmentNumber, AssignmentId | string | no |
| HCM HDL ExternalIdentifier DAT Prof | flatfile | ExtIdentifier/ExternalIdentifierNumber, ExternalIdentifierType, ExternalIdentifierSequence | string | yes |
| HCM HDL ExternalIdentifier DAT Prof | flatfile | ExtIdentifier/DateFrom, DateTo | string | no |
| HCM HDL ExternalIdentifier DAT Prof | flatfile | ExtIdentifier/SourceSystemOwner, SourceSystemId, GUID | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | METADATA / ExternalIdentifier / ExternalIdentifierSequence | (none) | hardcoded |  | "MERGE" / "ExternalIdentifier" / "1" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Transformations/xform_Int_ExtIdDAT.xsl | OTBI rowset to ExternalIdentifier DAT |
| MR-002 | PersonNumber | Column3 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Transformations/xform_Int_ExtIdDAT.xsl | OTBI rowset to ExternalIdentifier DAT |
| MR-003 | ExternalIdentifierNumber | Column1 (legal employer), Column2 (location), Column3 | conditional | TaskUs USA LLC + Santa Monica / TaskUs USA LLC / TaskUs Inc. | "MX4" / "6PZ" / "AK0" + substring(PersonNumber, 2) | medium | Empty for other employers. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Transformations/xform_Int_ExtIdDAT.xsl | OTBI rowset to ExternalIdentifier DAT |
| MR-004 | ExternalIdentifierType | (none) | hardcoded |  | "ORA_3RD_PARTY_PAY_ID" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Transformations/xform_Int_ExtIdDAT.xsl | OTBI rowset to ExternalIdentifier DAT |
| MR-005 | AssignmentNumber | Column4 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Transformations/xform_Int_ExtIdDAT.xsl | OTBI rowset to ExternalIdentifier DAT |
| MR-006 | DateFrom / DateTo | (current time) | calculated |  | now (yyyy/MM/dd HH:mm:ss) / "4712/12/31 11:59:00" | low | Not the assignment start date. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Transformations/xform_Int_ExtIdDAT.xsl | OTBI rowset to ExternalIdentifier DAT |
| MR-007 | SourceSystemId / SourceSystemOwner | Column4 | calculated |  | "EXT_" + substring-after(Column4, "ASSIGNMENT_") / "TASKUS" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Transformations/xform_Int_ExtIdDAT.xsl | OTBI rowset to ExternalIdentifier DAT |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Selection | US legislation, Active - Payroll Eligible, assignment starting today. | High | OTBI SQL via hcm-otbi-sql-query. |
| BR-002 | Payroll ID prefix | Prefix by legal employer and Santa Monica location. | High | Cross reference table (employer, location -> prefix). |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding / SOAP async | processupdateexternalidentifier_direct_ep / _client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/composite.xml |
| outbound File (SOA server) | /u01/data/domains/SOACS-Pr_domain/ExternalIdentifier/Worker.dat | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/composite.xml |
| outbound SOAP over HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/GenericSoapPort.wsdl |
| outbound SOAP over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCommonDataLoader/HCMDataLoader | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/HCMDataLoader.wsdl |
| outbound SOAP (composites) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl, gethdlerrorsprovabcsimpl_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/composite.xml |
| File WriteExtIdentifierDAT | LogicalDirectory ServerPath ([redacted]); Worker.dat; Append false; NXSD ExternalIdentifierDAT.xsd (pipe-delimited, header). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Adapters/WriteExtIdentifierDAT_file.jca |
| DB adapters XXTS_SCHEDULING | Select window, advance start, update last run. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Adapters/Select_XXTS_Scheduling_db.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| newhire-emp-sch | Sub: Schedule Window Get and Advance | synchronous |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| hcm-hdl-loader | Sub: HCM HDL Zip Checkin and Load | synchronous |
| hcm-get-hdl-errors | Sub: HCM Get HDL Errors | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java | Zip .dat files of the folder, Base64, clean folder. | hcm-hdl-loader (fn-zip-base64). | Medium |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Transformations/xform_Int_ExtIdDAT.xsl | Rowset to ExternalIdentifier DAT with prefix rules. | map-rowset-to-extid-dat + cross reference of prefixes. | Medium |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll notify only; HDL errors reported by email. |
| Retries | DB adapter retries only. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| Credentials.dvm | dvm | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud HDL | SOAP / HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCommonDataLoader/HCMDataLoader | Oracle (TaskUs tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Folder | [redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

Each day this job looks in HCM for new US hires whose assignment starts that day and gives each one a third-party payroll ID. The ID is built from a three-character code that depends on the legal employer and, for TaskUs USA LLC, on whether the location is Santa Monica, followed by the person number without its first character. The IDs are loaded into HCM with HCM Data Loader. After five minutes the job checks the load for errors and emails the outcome.

Only hires starting on the day the job runs are picked up, so a missed run means those hires never get the ID. Workers of any other legal employer get an empty ID. The result emails go to one hardcoded person in a format the notification service cannot read, so they are probably not delivered, and the five-minute wait is a guess at how long HCM needs.

In Boomi this becomes a scheduled process using the shared OTBI and HDL loading sub-processes, with the prefix rules held in a cross-reference table and the load status checked instead of a fixed wait.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | HCM HDL ExternalIdentifier DAT Prof | prof-hdl-extid-dat | profile.flatfile | — |
| 2 | Map \| OTBI Rowset to ExternalIdentifier DAT | map-rowset-to-extid-dat | transform.map | prof-otbi-rowset, prof-hdl-extid-dat |
| 3 | HCM External Identifier Load Sch | proc-hcm-external-identifier-sch | process | proc-sub-schedule-window, proc-sub-otbi-query, map-rowset-to-extid-dat, proc-sub-hdl-load, proc-sub-hcm-get-hdl-errors, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/BPEL/ProcessUpdateExternalIdentifier.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Adapters/WriteExtIdentifierDAT_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Adapters/Select_XXTS_Scheduling_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Adapters/Update_Scheduling_StartDateTime_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Schemas/ExternalIdentifierDAT.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Transformations/xform_Int_ExtIdDAT.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Transformations/xform_dat_to_ucm.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/Transformations/xfrom_gethdlError.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/GenericSoapPort.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/HCMDataLoader.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx

## Open questions

### Still outstanding

REVIEW-NEEDED: Is the US third-party payroll ID job running in production today? (The client WSDL exists only in Utils/DEV_MDS_JobVite, the composite is not in the scheduler spreadsheet, and its XXTS_SCHEDULING row is not in the export. Confirm whether this interface is live in production.)
  - If this is not answered: It is live and will be rebuilt.

REVIEW-NEEDED: When should the US third-party payroll ID job run each day? (The run time is not in the scheduler spreadsheet or the export.)
  - If this is not answered: Daily at 23:00 UTC, picking up all starters since the last successful run.

REVIEW-NEEDED: Are the payroll ID prefixes still MX4 (TaskUs USA LLC, Santa Monica), 6PZ (other TaskUs USA LLC) and AK0 (TaskUs Inc.), and what should happen for legal employers that match no prefix rule? (The rules are fixed in code; workers of other legal employers get no ID.)
  - If this is not answered: Keep the three rules in a maintainable table owned by US payroll.

REVIEW-NEEDED: Who should receive the success and error notices of this job? (They go to one named person today, in a format that is probably not delivered.)
  - If this is not answered: The US HRIS distribution list.

