# Migration Plan — ProcessJobviteConsolidatedNotification_Sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.491Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessJobviteConsolidatedNotification_Sch |
| Identifier | jobvite-consolidated-notification-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | simple (18) |
| Components to build | 3 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/BPEL/ProcessJobviteConsolidatedNotification.bpel |
| Depends on | dat-profile-generation-rest, util-file-notification, newhire-emp-sch |

## 2. End-to-End Process Architecture

Scheduler-started BPEL composite reading day files written by ProcessDatProfileGeneration, writing two reports and calling FileNotification; XXTS adapters partly skipped.

Boomi scheduled process "Jobvite Consolidated Notification Sch" (23:30 UTC): read the day's rows from the staging store, map-consolidated-to-summary per country, Process Call util-file-notification per country (skip or send "no hires" when empty), mark rows reported; Try/Catch with util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| XXTS scheduler direct binding | Boomi schedule 23:30 UTC | Window not used. |
| SyncFileRead of day files | Read from staging store (decision in dat-profile-generation-rest) |  |
| xfrom_PH/US_consolidated + WriteNotifications | map-consolidated-to-summary into prof-newhire-summary-csv (in memory) |  |
| FileNotification | Process Call util-file-notification |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Scheduled start | Direct binding from the scheduler (or SOAP client). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/BPEL/ProcessJobviteConsolidatedNotification.bpel |
| 2 | Read PH day file | Sync read JobviteConsolidate_PH.csv and delete; errors ignored. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Adapters/SyncFileRead_file.jca |
| 3 | Write PH report | xfrom_PH_consolidated -> NewHire_PH_<date>.csv. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Transformations/xfrom_PH_consolidated.xsl |
| 4 | Read US day file | Same for JobviteConsolidate_US.csv. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Adapters/SyncFileRead_file.jca |
| 5 | Write US report | xform_US_consolidated -> NewHire_US_<date>.csv. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Transformations/xform_US_consolidated.xsl |
| 6 | Email reports | FileNotification PH to HCMListEmail, US to USHCMListEmail. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/BPEL/ProcessJobviteConsolidatedNotification.bpel |
| 7 | Record run | Update_XXTS_Schedule last_scheduled_time (direct path); callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| NewHire Daily Summary CSV Prof | flatfile | Employee/Sno | number | yes |
| NewHire Daily Summary CSV Prof | flatfile | Employee/EmployeeId | string | no |
| NewHire Daily Summary CSV Prof | flatfile | Employee/CandidateId | string | no |
| NewHire Daily Summary CSV Prof | flatfile | Employee/FirstName, MiddleName, LastName | string | no |
| NewHire Daily Summary CSV Prof | flatfile | Employee/Status | string | no |
| NewHire Daily Summary CSV Prof | flatfile | Employee/Message | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | Sno | (row position) | calculated |  | position() | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Transformations/xfrom_PH_consolidated.xsl | Consolidated rows to daily summary |
| MR-002 | EmployeeId, CandidateId, FirstName, MiddleName, LastName, Status, Message | same fields | looped | each row |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Transformations/xfrom_PH_consolidated.xsl | Consolidated rows to daily summary |
| MR-003 | Header line | (none) | hardcoded |  | "SL_No, Person Number, Candidate ID, First Name, Middle Name, Last Name, Status, Message" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Transformations/xform_US_consolidated.xsl | Consolidated rows to daily summary |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Country split | PH report to HCMListEmail, US report to USHCMListEmail. | Medium | Two branches by country in the staging data. |
| BR-002 | Daily reset | Day files are deleted when read. | Medium | Mark staged rows as reported or delete them. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding / SOAP async | processjobviteconsolidatednotification_direct_ep / _client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/composite.xml |
| outbound File (SOA server) | /u01/data/domains/SOACS-Pr_domain/osi_file/EmailAttachment/JobviteConsolidate_PH.csv, _US.csv (read+delete); NewHire_PH/US_<date>.csv (write) | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Domain.dvm |
| outbound SOAP (composite) | oramds:/apps/Utilities/Notification/filenotification_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/composite.xml |
| outbound DB (SOA schema) | XXTS_SCHEDULING | data source | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca |
| File SyncFileRead (outbound read) | FileReadInteractionSpec; default directory [redacted], file Test; DeleteFile true; directory and file set at runtime. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Adapters/SyncFileRead_file.jca |
| File WriteNotifications | Append false; directory/file set at runtime; NXSD WriteNotification.xsd. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Adapters/WriteNotifications_file.jca |
| DB adapters XXTS_SCHEDULING | Select window and advance start (both skipped); update last_scheduled_time; jca.retry.count 4, interval 1, backoff 2, maxInterval 120. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| dat-profile-generation-rest | Consolidated row profile and staging | synchronous |
| util-file-notification | Sub: File Notification | synchronous |
| newhire-emp-sch | Schedule-window sub-process (XXTS, if retained) | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Transformations/xfrom_PH_consolidated.xsl | Add serial number and header. | map-consolidated-to-summary. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Empty catchAll around each file read; no process handler. |
| Retries | DB adapter retries only. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| Domain.dvm, EmailDL.dvm | dvm | yes |

## 12. External Services

> **Not available.** No external services were recorded for this interface.

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Folder | [redacted] | [redacted].dvm |

## 14. Boomi Migration Readiness Summary

Every night at 23:30 this job collects the day's results of the Jobvite hiring flow. During the day, each hire processed by the follow-up service adds a line (person number, candidate ID, name, success or failure and the message) to a Philippine or US file on the integration server. This job reads both files, numbers the lines, saves them as NewHire_PH_<date>.csv and NewHire_US_<date>.csv, and emails them to the PH and US HCM list recipients. The day files are deleted as they are read, so the next day starts empty.

If a country had no hires its file does not exist; the read error is silently ignored, but the email step still tries to attach a report that was never written, which most likely makes the job fail at that point without telling anyone. Because rows are appended to a file that has a header line, the day file may also contain repeated header lines. The job does not use the scheduler window; it only records its last run time.

In Boomi this becomes a scheduled process that reads the day's rows from the staging store agreed for dat-profile-generation-rest, builds the two reports and sends them through the file-notification sub-process, sending a "no hires today" notice or nothing when a country has no rows.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | NewHire Daily Summary CSV Prof | prof-newhire-summary-csv | profile.flatfile | — |
| 2 | Map \| Consolidated Rows to Daily Summary | map-consolidated-to-summary | transform.map | prof-jobvite-consolidated-csv, prof-newhire-summary-csv |
| 3 | Jobvite Consolidated Notification Sch | proc-jobvite-consolidated-notification-sch | process | map-consolidated-to-summary, proc-sub-file-notification, op-xxts-set-lastrun |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/BPEL/ProcessJobviteConsolidatedNotification.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Adapters/SyncFileRead_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Adapters/WriteNotifications_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Adapters/Update_Scheduling_StartDateTime_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Transformations/xfrom_PH_consolidated.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Transformations/xform_US_consolidated.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Schemas/WriteNotification.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessJobviteConsolidatedNotification_Sch/SOA/Schemas/SyncFileRead.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Domain.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Adapters/Write_Consolidated_File_file.jca

## Open questions

### Still outstanding

REVIEW-NEEDED: On a day with no hires for a country, should that country's team receive a "no hires today" email or nothing? (Today a missing day file probably makes the job fail without notice.)
  - If this is not answered: Send nothing for that country.

REVIEW-NEEDED: Is 23:30 UTC still the right time for the daily Jobvite hire summary? (The schedule comes from the spreadsheet; the window table is not used.)
  - If this is not answered: 23:30 UTC daily.

