# Migration Plan — ProcessIncompleteHours_Sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.605Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessIncompleteHours_Sch |
| Identifier | incomplete-hours-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | medium (38) |
| Components to build | 4 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/BPEL/ProcessIncompleteHours_Sch.bpel |
| Depends on | hcm-get-hdl-errors, hcm-otbi-sql-query |

## 2. End-to-End Process Architecture

SOAP-started BPEL: async OTBI employee list, sequential forEach with four week scopes guarded by DVM skip conditions, BIP runReport, Java Base64 decode, XSLT calculation, File adapter CSV.

Boomi scheduled process "Weekly Incomplete Hours Report Sch" (proc-incomplete-hours-sch): hcm-otbi-sql-query for the employee list, per employee and week a SOAP Client call to BI Publisher runReport (account configured in the connection), Base64 decode, profile prof-payable-time-report, map-time-report-to-incomplete-hours, combine into prof-incomplete-hours-csv and send to the agreed destination.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| Async OTBI provider | Process Call hcm-otbi-sql-query |  |
| PublicReportService runReport | Web Services SOAP Client connector | Credentials in connection, not in the map. |
| Base64DecoderClass | Data Process Base64 decode |  |
| xform_TimeReportWeekly_IncompleteHrs | map-time-report-to-incomplete-hours |  |
| File adapter CSV | Disk/SFTP connector with prof-incomplete-hours-csv | Destination to confirm. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Start | SOAP process request. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/composite.xml |
| 2 | List eligible employees | OTBI active PH payroll-eligible; filter job 20 not OTND or 30. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Transformations/Xform_employeelist.xsl |
| 3 | Per employee and week | If today is a pay date in PayrollWeeks with week N dates, run the payable time report. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/BPEL/ProcessIncompleteHours_Sch.bpel |
| 4 | Decode report | Base64 decode, normalise G_2 to G_1, parse. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/SCA-INF/src/com/osi/Base64DecoderClass.java |
| 5 | Calculate shortfall | 40 minus approved counted hours per week. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Transformations/xform_TimeReportWeekly_IncompleteHrs.xsl |
| 6 | Write CSV | WeeklyIncompleteHrs<yyyyMMdd>.csv to the ADP folder. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Adapters/WriteIncompleteHrs_file.jca |
| 7 | Callback | processResponse to the caller. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/BPEL/ProcessIncompleteHours_Sch.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| HCM Payable Time Report XML Prof | xml | DATA_DS/P_FROM_DATE | dateTime | yes |
| HCM Payable Time Report XML Prof | xml | DATA_DS/P_TO_DATE | dateTime | yes |
| HCM Payable Time Report XML Prof | xml | DATA_DS/G_1/PERSON_NUMBER | integer | yes |
| HCM Payable Time Report XML Prof | xml | DATA_DS/G_1/FULL_NAME | string | yes |
| HCM Payable Time Report XML Prof | xml | DATA_DS/G_1/LEGAL_ENTITY | string | yes |
| HCM Payable Time Report XML Prof | xml | DATA_DS/G_1/BUSINESSUNIT | string | yes |
| HCM Payable Time Report XML Prof | xml | DATA_DS/G_1/LOCATION_NAME | string | yes |
| HCM Payable Time Report XML Prof | xml | DATA_DS/G_1/MEASURE | integer | yes |
| HCM Payable Time Report XML Prof | xml | DATA_DS/G_1/TIME_CARD_DATE | date | yes |
| HCM Payable Time Report XML Prof | xml | DATA_DS/G_1/PUNCH_IN | dateTime | yes |
| HCM Payable Time Report XML Prof | xml | DATA_DS/G_1/PUNCH_OUT | dateTime | yes |
| HCM Payable Time Report XML Prof | xml | DATA_DS/G_1/STATUS | string | yes |
| HCM Payable Time Report XML Prof | xml | DATA_DS/G_1/ELEMENT | string | yes |
| Weekly Incomplete Hours CSV Prof | flatfile | EmployeeId | string | no |
| Weekly Incomplete Hours CSV Prof | flatfile | W1HrsCount | string | no |
| Weekly Incomplete Hours CSV Prof | flatfile | W2HrsCount | string | no |
| Weekly Incomplete Hours CSV Prof | flatfile | W3HrsCount | string | no |
| Weekly Incomplete Hours CSV Prof | flatfile | W4HrsCount | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | W1HrsCount … W4HrsCount | DATA_DS/G_1/MEASURE | calculated | Week N processed; STATUS = Approved and ELEMENT in (Regular Hours, Planned Leave, Birthday, Unplanned) | 40 - sum(MEASURE), format ##.## | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Transformations/xform_TimeReportWeekly_IncompleteHrs.xsl | Payable time report to weekly incomplete hours |
| MR-002 | EmployeeId | Employee/EmployeeId (OTBI Column4 Person Number) | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Transformations/Xform_employeelist.xsl | Employees to CSV |
| MR-003 | W1HrsCount … W4HrsCount | Employee/WnHrsCount | conditional | Value not empty | format-number ##.## | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Transformations/xform_Write_IncompleteHrs.xsl | Employees to CSV |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Eligible employees | Active - Payroll Eligible, PH, and (job 20 with Not Eligible for OTND) or job 30. | High | OTBI query + filter. |
| BR-002 | Run only on pay dates | Week N processed only when today is a PayrollWeeks key with week N dates. | High | Cross reference lookup; decide new calendar source. |
| BR-003 | 40-hour week | Shortfall = 40 - approved Regular Hours, Planned Leave, Birthday, Unplanned. | High | Map function. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP async | processincompletehours_sch_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/composite.xml |
| outbound SOAP async (composite) | oramds:/apps/Provider/HCM/processhcmextractprovabcs_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/composite.xml |
| outbound SOAP | oramds:/apps/Provider/HCM/PublicReportService.wsdl | user ID and password in the request body (Credentials.dvm, redacted) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Transformations/xform_PublicReportService.xsl |
| outbound File | /u01/app/oracle/tools/paas/state/homes/oracle/ADP_OSI_Taskus/WeeklyIncompleteHrs<yyyyMMdd>.csv | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Adapters/WriteIncompleteHrs_file.jca |
| File WriteIncompleteHrs | [redacted], FileNamingConvention WeeklyIncompleteHrs%yyyyMMdd%.csv, Append false, NXSD Write_IncompleteHrs.xsd. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Adapters/WriteIncompleteHrs_file.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| hcm-get-hdl-errors | BI Publisher runReport pattern (PublicReportService) | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/SCA-INF/src/com/osi/Base64DecoderClass.java | Base64 decode of BIP reportBytes. | Base64 decode in a Data Process shape. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Transformations/xform_TimeReportWeekly_IncompleteHrs.xsl | 40 - sum of counted approved hours per week. | map-time-report-to-incomplete-hours (map function or script). | Medium |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | None. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/HCM/PublicReportService.wsdl | mds | yes |
| PayrollWeeks.dvm | dvm | yes |
| Credentials.dvm | dvm | yes |
| BIP report SOA Payable Time RPT.xdo | other | no |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM BI Publisher | SOAP | PublicReportService (MDS WSDL) | TaskUs HCM |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Output folder | [redacted] | [redacted].jca |

## 14. Boomi Migration Readiness Summary

This job works out how many hours each eligible Philippines employee was short of a 40-hour week, for up to four payroll weeks. It lists active, payroll-eligible employees in the Philippines whose job is in category 30, or category 20 and not eligible for overtime/night differential, then runs an HCM report of each employee's payable time for each week. Approved regular hours, planned leave, birthday leave and unplanned leave count towards the 40 hours; the shortfall is written to a CSV file on the integration server.

The weeks are taken from a payroll calendar table, and a week is only processed when today's date is exactly one of the pay dates in that table. The table only covers pay dates from February 2018 to January 2019, so today no week is processed and the file contains employee numbers with no hours. Nothing in the export reads the CSV.

A related ADP job (adp-transactional-file-sch) calculates incomplete hours differently (8 hours per time entry). TaskUs needs to confirm whether this report is still needed, which rule is right, and where the payroll calendar should come from. If it is kept, Boomi would run it on a schedule, call the BI Publisher report through the SOAP connector and write the CSV wherever it is needed.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | HCM Payable Time Report XML Prof | prof-payable-time-report | profile.xml | — |
| 2 | Weekly Incomplete Hours CSV Prof | prof-incomplete-hours-csv | profile.flatfile | — |
| 3 | Map \| Payable Time Report to Incomplete Hours | map-time-report-to-incomplete-hours | transform.map | prof-payable-time-report, prof-incomplete-hours-csv |
| 4 | Weekly Incomplete Hours Report Sch | proc-incomplete-hours-sch | process | proc-sub-otbi-query, op-bip-run-report, map-time-report-to-incomplete-hours |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/BPEL/ProcessIncompleteHours_Sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Adapters/WriteIncompleteHrs_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/SCA-INF/src/com/osi/Base64DecoderClass.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Transformations/Xform_employeelist.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Transformations/xform_PublicReportService.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Transformations/xform_TimeReportWeekly_IncompleteHrs.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Transformations/xform_Write_IncompleteHrs.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Schemas/PayableTimeReport.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessIncompleteHours_Sch/SOA/Schemas/Write_IncompleteHrs.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/ADP/PayrollWeeks.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Credentials.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/Taskus_Querys.sql
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx

## Open questions

### Still outstanding

REVIEW-NEEDED: Is the weekly incomplete-hours report still required, who uses it, and where should the file go? (PayrollWeeks.dvm covers only pay dates 2018-02-03 to 2019-01-03, so the process does nothing today. No consumer of the output CSV exists in the export. Confirm whether this report is still required, its schedule, where the file should go, and the source of payroll week dates.)
  - If this is not answered: Not migrated until a consumer is identified.

REVIEW-NEEDED: Where should payroll-week start and end dates come from in future? (The calendar is a fixed table that ended in January 2019 and is matched only on exact pay dates.)
  - If this is not answered: A Boomi cross-reference table maintained by payroll each year.

REVIEW-NEEDED: Is an incomplete week measured against 40 hours with regular hours and planned, birthday and unplanned leave counting, or as 8 hours per time entry as in the ADP transactional file? (The two legacy jobs calculate incomplete hours differently.)
  - If this is not answered: 40 hours per week with the four counted types.

REVIEW-NEEDED: How often should the report run? (No schedule is in the export.)
  - If this is not answered: On each pay date (3rd and 18th).

