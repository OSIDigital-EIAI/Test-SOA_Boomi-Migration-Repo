# Migration Plan — ProcessDatProfileGeneration (archive file poll)

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.459Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessDatProfileGeneration (archive file poll) |
| Identifier | dat-profile-archive-poll |
| Trigger | Scheduled |
| Starts with | Disk |
| Complexity | simple (20) |
| Components to build | 2 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Adapters/ReadINTFile_file.jca |
| Depends on | dat-profile-generation-rest, int-file-intake |

## 2. End-to-End Process Architecture

Inbound File adapter service of ProcessDatProfileGeneration (pick onMessage ReadINTFile) feeding the same BPEL as the REST service.

Boomi scheduled process "DAT Profile Generation Archive Poll": op-int-archive-get on conn-int-file-share (int-file-intake), profile prof-int-newhire-csv, Process Call proc-sub-dat-profile-core (dat-profile-generation-rest); retain an archive copy.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| ReadINTFile File adapter | op-int-archive-get on conn-int-file-share |  |
| BPEL core | Process Call proc-sub-dat-profile-core |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Poll archive folder | Every 10 minutes; files .*\.csv_.* older than 2 hours; delete after read. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Adapters/ReadINTFile_file.jca |
| 2 | Parse file | ReadFileService NXSD (INT new-hire CSV). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Schemas/ReadFileService.xsd |
| 3 | Run post-load core | Same loop and batch steps as dat-profile-generation-rest. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/BPEL/ProcessDatProfileGeneration.bpel |

## 4. Payload Profiles

> **Not available.** No profile components owned here; it uses the INT CSV profile of int-file-intake.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | Employees/Employee[] | CSV rows | passthrough |  | NXSD parse; same element names | low | No XSLT; the BPEL uses the parsed file directly. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/BPEL/ProcessDatProfileGeneration.bpel | Archived INT CSV to Employees |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Settle time | Files are only processed when at least two hours old. | Medium | Schedule plus file-age filter, or trigger after confirmed HDL load. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound File (SOA server) | /u01/data/domains/SOACS-Pr_domain/InterviewedArchive/.*\.csv_.* | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/composite.xml |
| File adapter ReadINTFile (inbound) | LogicalDirectory LogicalPath; IncludeFiles .*\.csv_.*; Recursive true; MinimumAge 7200; PollingFrequency 600; DeleteFile true; UseHeaders false; NXSD ReadFileService.xsd. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Adapters/ReadINTFile_file.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| dat-profile-generation-rest | Sub: DAT Profile Generation Core | synchronous |
| int-file-intake | File connection and INT CSV profile | synchronous |

## 9. Custom Code

> **Not available.** No custom code was found in this interface.

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Composite rejected-message recovery; core errors as in dat-profile-generation-rest. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

> **Not available.** No shared artefacts were recorded for this interface.

## 12. External Services

> **Not available.** No external services were recorded for this interface.

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Archive folder | [redacted] (composite default; plan value empty) | [redacted].xml |
| DEV | Archive folder | [redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

After the new-hire file intake has processed a file from the recruiting system, the file is moved to an archive folder. Two hours later this poll picks the archived file up again and runs the follow-up work for every hire in it: checking that HCM created the person, updating Jobvite with success or the error, emailing HR lists, adding the hires to the nightly summary, and loading salary, talent profile and allowances. The archived file is then deleted, so no archive copy remains.

The two-hour delay gives HCM time to finish loading the workers. In the production configuration plan the folder setting for this poll is blank (the composite itself names InterviewedArchive), so whether the poll is actually active in production cannot be confirmed from the export. In Boomi this becomes a scheduled process that reads the same files (or, better, is triggered once the worker loads are confirmed) and calls the shared core sub-process.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | INT Archive CSV GET n DEL Op | op-int-archive-get | connector-action | conn-int-file-share |
| 2 | DAT Profile Generation Archive Poll | proc-dat-profile-archive-poll | process | op-int-archive-get, prof-int-newhire-csv, proc-sub-dat-profile-core |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Adapters/ReadINTFile_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/WSDLs/ReadINTFile.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Schemas/ReadFileService.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/BPEL/ProcessDatProfileGeneration.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/ProcessDatProfileGeneration_prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/ProcessDatProfileGeneration_dev_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/ProcessINTReadFileABCSImpl_prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/BPEL/ProcessINTReadFileABCSImpl.bpel

## Open questions

### Still outstanding

REVIEW-NEEDED: Do hires that arrive by file currently receive the Jobvite status update and the salary and allowance loads? (This follow-up depends on a delayed re-read of the archive whose production folder setting is blank; if it is not running, those steps are missing today.)
  - If this is not answered: Yes; build the follow-up as a step that runs once the worker load is confirmed.

REVIEW-NEEDED: How long should processed new-hire files be kept, given that they contain personal data? (Today the archive copy is deleted after the follow-up.)
  - If this is not answered: Keep 30 days in a restricted archive, then delete.

