# Migration Plan — ProcessADPTransProvABCS

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.580Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessADPTransProvABCS |
| Identifier | adp-transactional-file-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | complex (76) |
| Components to build | 4 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/BPEL/ProcessADPTransProvABCS.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Transformations/Xform_Hcm_to_TransactionalFile.xsl |
| Depends on | adp-employee-masterdata, newhire-emp-sch, hcm-otbi-sql-query, util-send-notification, hcm-dat-generation |

## 2. End-to-End Process Architecture

Scheduler/SOAP-started BPEL with XXTS DB adapters, async OTBI provider calls per employee, small XSLTs that stage rowsets into EBM fields, a large XSLT to NXSD, File adapter, Java PGP, skipped FTP.

Boomi scheduled process "ADP Transactional File Sch" (proc-adp-transactional-file-sch): window sub-process (newhire-emp-sch), hcm-otbi-sql-query for employees, time cards and element entries, fn-incomplete-hours, map-ebm-to-adp-tr into prof-adp-tr-flatfile, PGP Encrypt and the ADP SFTP send from adp-employee-masterdata; notifications via util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| XXTS DB adapters | Sub: Schedule Window Get and Advance | Fix the missing update values. |
| Async OTBI provider | Process Call hcm-otbi-sql-query |  |
| Incomplete-hours XSL + BPEL arithmetic | fn-incomplete-hours |  |
| Xform_Hcm_to_TransactionalFile | map-ebm-to-adp-tr |  |
| FileEncryption + FTP | PGP Encrypt + op-adp-put (adp-employee-masterdata) |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Start and read window | Direct binding or SOAP client; XXTS window read into StartDate/EndDate. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/BPEL/ProcessADPTransProvABCS.bpel |
| 2 | List employees | OTBI active payroll-eligible PH employees. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/BPEL/ProcessADPTransProvABCS.bpel |
| 3 | Incomplete hours (eligible jobs) | Time-card queries for two payroll weeks; hours shortfall per week. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Transformations/xForm_Timecard_To_Incomplete_Hours.xsl |
| 4 | Wage elements | Element entries per employee. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/BPEL/ProcessADPTransProvABCS.bpel |
| 5 | Build TR file | Header, P0014, P0015, P2010, trailer. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Transformations/Xform_Hcm_to_TransactionalFile.xsl |
| 6 | Write and encrypt | Write .SAP, PGP to .pgp via Java embedding. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/WriteTransData_file.jca |
| 7 | Upload (disabled) | Read .pgp and FTP put — skipped. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/PutTransData_ftp.jca |
| 8 | Update window and reply | Scheduler update (values missing) or client callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/BPEL/ProcessADPTransProvABCS.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| ADP G2 Transactional Flat File Prof | flatfile | HEADER | record | no |
| ADP G2 Transactional Flat File Prof | flatfile | P0014 | record | no |
| ADP G2 Transactional Flat File Prof | flatfile | P0015 | record | no |
| ADP G2 Transactional Flat File Prof | flatfile | P2010 | record | no |
| ADP G2 Transactional Flat File Prof | flatfile | TRAILER | record | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | HEADER | (constants) | hardcoded | First employee | FNAME TR; other header constants as HRMD | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Transformations/Xform_Hcm_to_TransactionalFile.xsl | Employee wage elements to ADP TR records |
| MR-002 | P0014 (SUBTY, LGART, BETRG, WAERS PHP) | Person/Visa VisaPermitNumber (wage code), VisaPermitType (amount) | looped | Code in {3000,3001,3002,3003,3004,3027,3040,3044,3045,3046,3047,3050,3051,3100,3130,3200,3201,7001,9372,9374,9380,9384} and amount not in {N, PRD, "", Semi-Month, Year, Y, 0} |  | medium | 22 blocks. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Transformations/Xform_Hcm_to_TransactionalFile.xsl | Employee wage elements to ADP TR records |
| MR-003 | P0015 | Person/Visa | looped | Code in the P0015 list (42 blocks incl. 3024, 3031, 3034, 3042, 3043, 3060, 3061, 3070, 3101–3104, 3110–3112, 3120, 3121, 3131–3134, 3150, 3151, 5001, 5002) and valid amount |  | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Transformations/Xform_Hcm_to_TransactionalFile.xsl | Employee wage elements to ADP TR records |
| MR-004 | BEGDA | EmploymentAssignments/EffectiveStartDate | transformed |  | yyyyMMdd, floor 20170926 | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Transformations/Xform_Hcm_to_TransactionalFile.xsl | Employee wage elements to ADP TR records |
| MR-005 | P2010 SUBTY 2051, STDAZ | Person/Dependent/PersonalCode (incomplete hours) | conditional | Eligible job and hours > 0 |  | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Transformations/Xform_Hcm_to_TransactionalFile.xsl | Employee wage elements to ADP TR records |
| MR-006 | TRAILER RECCT | records | calculated |  | P0014 + P0015 + P2010 + 2 | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/BPEL/ProcessADPTransProvABCS.bpel | Employee wage elements to ADP TR records |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Population | Active - Payroll Eligible employees with PH mailing country. | High | OTBI query. |
| BR-002 | Incomplete-hours eligibility | (JobCode 20 and Not Eligible for OTND) or JobCode 30. | High | Decision. |
| BR-003 | Incomplete hours | Per payroll week: 8 h x number of time entries minus reported hours; send if > 0. | High | fn-incomplete-hours. |
| BR-004 | Valid amounts only | Skip amounts N, PRD, empty, Semi-Month, Year, Y, 0. | Medium | Map condition. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding / SOAP async | processadptransprovabcs_DirectBinding / _client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/composite.xml |
| outbound SOAP async (composite) | oramds:/apps/Provider/HCM/processhcmextractprovabcs_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/composite.xml |
| outbound File | /u01/app/oracle/tools/paas/state/homes/oracle/ADP_OSI_Taskus/PP3208_<ts>_PHOHCM_TRAN01_DUT8G2I.SAP | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/WriteTransData_file.jca |
| outbound FTP (disabled) | eis/Ftp/FtpAdapter | server connection factory (not in export) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/PutTransData_ftp.jca |
| DB Select_XXTS_Scheduling | Window start and start + interval for interface ProcessADPTransProvABCS. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/Select_XXTS_Scheduling_db.jca |
| DB Update_Scheduling_StartDateTime / Update_XXTS_Schedule | Advance start and set last run; called with only the interface name set. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/Update_Scheduling_StartDateTime_db.jca |
| File WriteTransData | [redacted], name at runtime, NXSD Write_Transactional_Data.xsd. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/WriteTransData_file.jca |
| File SyncReadADPFile / FTP PutTransData | Read .pgp, put (skipped). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/SyncReadADPFile_file.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| adp-employee-masterdata | ADP SFTP connection, put operation, trailer pattern | synchronous |
| newhire-emp-sch | Sub: Schedule Window Get and Advance | synchronous |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Transformations/xForm_Timecard_To_Incomplete_Hours.xsl | Sum reported hours per week. | fn-incomplete-hours. | Medium |
| other | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/BPEL/ProcessADPTransProvABCS.bpel | count(entries) x 8 - sum(hours) per week; RECCT count. | fn-incomplete-hours; trailer count as in adp-employee-masterdata. | Medium |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/BPEL/ProcessADPTransProvABCS.bpel | PGP via FileEncryption shell script. | PGP Encrypt shape. | High |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Transformations/Xform_Hcm_to_TransactionalFile.xsl | EBM to TR records. | map-ebm-to-adp-tr. | High |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Per-employee and process catchAll with notification. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| PayrollWeeks.dvm | dvm | yes |
| Credentials.dvm (PGP keys) | dvm | yes |
| PGP shell script and ADP public key | other | no |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| ADP GlobalView GVS | FTP (legacy, disabled) | eis/Ftp/FtpAdapter | ADP |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Output folder | [redacted] | [redacted].jca |

## 14. Boomi Migration Readiness Summary

This job builds the payroll transactions file that ADP uses alongside the master data to pay TaskUs Philippines staff. Each run it lists every active, payroll-eligible employee in the Philippines, reads their pay elements from HCM (allowances, adjustments and other wage codes) and, for employees in certain job categories, works out how many hours were missing from their time cards over two payroll weeks. These become ADP recurring-payment, one-off payment and incomplete-hours records.

As exported the incomplete-hours part cannot work correctly: the payroll-week calendar it reads is looked up with a column name and a date that do not exist in the lookup table, the second week's end date uses the first week's value, and the run window is never moved forward. The file is encrypted but, as with the master data, the upload to ADP is switched off.

In Boomi this becomes a scheduled process that reuses the OTBI sub-process, a script for incomplete hours, a flat-file map and the ADP SFTP connection from adp-employee-masterdata. The payroll calendar and the incomplete-hours rule must be confirmed first.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | ADP G2 Transactional Flat File Prof | prof-adp-tr-flatfile | profile.flatfile | — |
| 2 | Incomplete Hours Calculation Script | fn-incomplete-hours | transform.function | — |
| 3 | Map \| Employee Wage Elements to ADP TR Records | map-ebm-to-adp-tr | transform.map | prof-employee-ebm-xml, prof-adp-tr-flatfile |
| 4 | ADP Transactional File Sch | proc-adp-transactional-file-sch | process | proc-sub-schedule-window, proc-sub-otbi-query, fn-incomplete-hours, map-ebm-to-adp-tr, fn-adp-trailer-count, op-adp-put, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/BPEL/ProcessADPTransProvABCS.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Transformations/Xform_Hcm_to_TransactionalFile.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Schemas/Write_Transactional_Data.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Transformations/xForm_Timecard_To_Incomplete_Hours.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Transformations/xForm_IncompleteHours_To_TempVar.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/WriteTransData_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/PutTransData_ftp.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/SyncReadADPFile_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/Select_XXTS_Scheduling_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/Update_Scheduling_StartDateTime_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessADPTransProvABCS/SOA/Adapters/Update_XXTS_Schedule_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/ADP/PayrollWeeks.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Credentials.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/Taskus_Querys.sql

## Open questions

### Still outstanding

REVIEW-NEEDED: Where should the payroll-week dates for incomplete hours come from, and which rule applies: 8 hours per time entry or 40 hours per week? (As exported this interface cannot produce correct incomplete-hours records: PayrollWeeks.dvm has no weekOne/weekTwo columns and no key 2018-03-01, the week-2 range end uses weekOne, and the window update passes null values. Confirm the payroll-week calendar source, which incomplete-hours rule applies (8 h per entry here vs 40 h per week in incomplete-hours-sch), and the run schedule.)
  - If this is not answered: Use the PayrollWeeks table keyed by the current payroll period start and 8 hours per time entry, as coded.

REVIEW-NEEDED: How often and at what time should the ADP transactional file be produced? (No schedule for this job is in the export.)
  - If this is not answered: Once per payroll cut-off, triggered manually until a schedule is agreed.

REVIEW-NEEDED: Should the transactional file be delivered to ADP the same way as the master-data file? (The upload is disabled here as well; delivery is decided under adp-employee-masterdata.)
  - If this is not answered: Same PGP + SFTP delivery as the master-data file.

REVIEW-NEEDED: Are the wage codes in the recurring (P0014) and one-off (P0015) lists still complete and correct? (The code lists are hardcoded in the map; any new pay element would be silently omitted.)
  - If this is not answered: Keep the current lists in a configurable cross-reference table.

