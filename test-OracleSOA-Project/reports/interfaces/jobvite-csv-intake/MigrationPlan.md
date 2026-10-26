# Migration Plan — ProcessCallJobviteRestService

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.474Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessCallJobviteRestService |
| Identifier | jobvite-csv-intake |
| Trigger | Scheduled |
| Starts with | Disk |
| Complexity | simple (16) |
| Components to build | 4 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/BPEL/ProcessCallJobiveRestService.bpel |
| Depends on | jobvite-create-employee-rest, int-file-intake, dat-profile-generation-rest, util-send-notification |

## 2. End-to-End Process Architecture

BPEL composite started by an inbound File adapter; sequential loop with XSLT and a REST call to another composite (ProcessINTReadFileABCSImpl) using a hardcoded Basic header.

Boomi scheduled process "Jobvite Candidate CSV Intake": op-jobvite-folder-get (Disk/SFTP, get and delete, archive), prof-jobvite-flat-csv, split rows, map-jobvite-flat-to-employees, Process Call proc-sub-int-validate-create (int-file-intake) per row inside Try/Catch; failures reported via util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| File adapter ReadINTFile | op-jobvite-folder-get |  |
| xform_Temp_RestService | map-jobvite-flat-to-employees |  |
| REST call to /create | Process Call proc-sub-int-validate-create | No HTTP self-call, no stored password. |
| (no error handling) | Try/Catch per row + util-send-notification |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Poll Jobvite folder | Read *.csv, delete after read. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Adapters/ReadINTFile_file.jca |
| 2 | Parse rows | ReadFile.xsd (header, quoted CSV). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Schemas/ReadFile.xsd |
| 3 | Convert row | xform_Temp_RestService to Employees JSON (one employee). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Transformations/xform_Temp_RestService.xsl |
| 4 | Create employee | POST to ProcessINTReadFileABCSImpl /create with Basic auth. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/WADLs/JobviteRestService.wadl |
| 5 | Finish | Wait 10 seconds after the last row. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/BPEL/ProcessCallJobiveRestService.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| Jobvite Flat Candidate CSV Prof | flatfile | Employee (one per line after header) | record | yes |
| Jobvite Flat Candidate CSV Prof | flatfile | ID, FirstName, MiddleName, LastName, Email, PhoneNumber, SSSNo, TINno, Philhealth, HDMFPagibig, ... | string | no |
| Jobvite Flat Candidate CSV Prof | flatfile | AllowanceType, Allowance, Salary, Currency, SalaryGrade, Location, JobCode, Campaign, ... | string | no |
| Jobvite Flat Candidate CSV Prof | flatfile | PreviousEmployer1..3 and related columns; education columns | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | Employee scalar fields (~75) | same-named CSV columns | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Transformations/xform_Temp_RestService.xsl | Jobvite flat CSV row to Employees JSON |
| MR-002 | Employee/TaxableAllowance, NonTaxableAllowance | AllowanceType, Allowance | passthrough |  |  | low | Renamed. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Transformations/xform_Temp_RestService.xsl | Jobvite flat CSV row to Employees JSON |
| MR-003 | Employee/Location | Location | transformed |  | substring-before(Location, " ") | low | First word only. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Transformations/xform_Temp_RestService.xsl | Jobvite flat CSV row to Employees JSON |
| MR-004 | Employee/PreviousEmploymentDetails[1..3] | PreviousEmployer1..3, PreviousPosition1..3, PreviousSalary1..3, dates, address, city, postcode, reason | conditional | start date of that employer present | one group per employer | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Transformations/xform_Temp_RestService.xsl | Jobvite flat CSV row to Employees JSON |
| MR-005 | Employee/EducationDetails | Educational attainment columns | passthrough |  | one group | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Transformations/xform_Temp_RestService.xsl | Jobvite flat CSV row to Employees JSON |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | One request per candidate | Each CSV row is sent separately to the create-employee service. | Medium | Split rows; Process Call proc-sub-int-validate-create per row. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound File (SOA server) | /u01/data/domains/SOACS-Pr_domain/JobviteFolder/*.csv | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/composite.xml |
| outbound REST over plain HTTP | http://129.144.156.139:9073/soa-infra/resources/default/ProcessINTReadFileABCSImpl!2.0/CreateEmployee/create | HTTP Basic (hardcoded header) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/composite.xml |
| File adapter ReadINTFile (inbound) | LogicalDirectory FileDirectory ([redacted]); IncludeFiles .*\.csv; Recursive true; PollingFrequency 60; MinimumAge 0; DeleteFile true; UseHeaders false. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Adapters/ReadINTFile_file.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| jobvite-create-employee-rest | Legacy target of the REST call | synchronous |
| int-file-intake | Sub: INT Employee Validate and Create | synchronous |
| dat-profile-generation-rest | Employees JSON profile | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Transformations/xform_Temp_RestService.xsl | Flat 113-column row to nested Employees record. | map-jobvite-flat-to-employees. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | None. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

> **Not available.** No shared artefacts were recorded for this interface.

## 12. External Services

> **Not available.** No external services were recorded for this interface.

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Input folder | [redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

This integration watches a folder for candidate files exported from Jobvite. For each file it reads every candidate row (113 columns: personal details, IDs, address, job, schedule, salary, allowances, up to three previous employers and one education record), turns it into the format of the TaskUs create-employee service and calls that service once per candidate. The file is deleted after reading.

The call goes over plain HTTP to the integration server with a username and password written into the code. There is no error handling: if a call fails the instance stops and later rows in the file are not sent, and nobody is told. The location is cut to its first word before sending, while the salary-basis lookup later uses the part after the first space, so Jobvite hires may get no salary basis. This needs confirming with real data.

In Boomi this becomes a scheduled process reading the folder (Disk or SFTP), with a flat-file profile and one map, calling the shared validation sub-process directly instead of making an HTTP call to another integration.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Jobvite Flat Candidate CSV Prof | prof-jobvite-flat-csv | profile.flatfile | — |
| 2 | Jobvite Folder CSV GET n DEL Op | op-jobvite-folder-get | connector-action | conn-int-file-share |
| 3 | Map \| Jobvite Flat CSV to Employees JSON | map-jobvite-flat-to-employees | transform.map | prof-jobvite-flat-csv, prof-jobvite-employees-json |
| 4 | Jobvite Candidate CSV Intake | proc-jobvite-csv-intake | process | op-jobvite-folder-get, map-jobvite-flat-to-employees, proc-jobvite-create-employee-rest, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/BPEL/ProcessCallJobiveRestService.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Adapters/ReadINTFile_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Schemas/ReadFile.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Schemas/ReadFileService.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/Transformations/xform_Temp_RestService.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/WADLs/JobviteRestService.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_IntReq_to_WriteSalatyDat.xsl

## Open questions

### Still outstanding

REVIEW-NEEDED: Which system or person puts the Jobvite candidate files in the folder, and where should Boomi pick them up (a folder on the Boomi server or an SFTP location)? (The file producer and the Boomi drop location are not defined in the export.)
  - If this is not answered: An SFTP folder managed by TaskUs IT.

REVIEW-NEEDED: What does the "Location" value in the Jobvite file look like, and which part identifies the salary basis? (The location is cut to its first word here but the salary-basis lookup later uses the text after the first space, so Jobvite hires may get no salary basis.)
  - If this is not answered: Send the full location and let the salary map choose the part it needs.

REVIEW-NEEDED: If one candidate in a file fails, should the remaining candidates still be processed? (Today a failure stops the rest of the file silently.)
  - If this is not answered: Yes; process the rest and report the failed rows.

