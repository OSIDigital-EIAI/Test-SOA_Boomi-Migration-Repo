# Migration Plan — ProcessJobviteFileGen_Sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.862Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessJobviteFileGen_Sch |
| Identifier | jobvite-filegen-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | simple (22) |
| Components to build | 5 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/BPEL/ProcessJobviteFileGen_Sch.bpel |
| Depends on | hcm-otbi-sql-query, util-send-notification |

## 2. End-to-End Process Architecture

BPEL composite with SOAP and direct-binding entry points. It calls the asynchronous ProcessHCMExtractProvABCS composite for OTBI, transforms the rowset with XSLT (Countries DVM lookup) and writes the CSV with the FTP adapter. A local file write and an older XSL are skipped; the XXTS scheduling adapters are wired but unused.

Scheduled Boomi process "Jobvite Employee File Generation Sch": Process Call to the shared OTBI query sub-process, map-rowset-to-jobvite-csv (Countries cross reference), Set Properties for the dated file name, then op-jobvite-ftp-put. Wrapped in Try/Catch with util-send-notification on failure.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| ProcessHCMExtractProvABCS (async) | Process Call hcm-otbi-sql-query | Synchronous in Boomi. |
| xform_hcm_to_Jobvite_put.xsl + Countries.dvm | map-rowset-to-jobvite-csv + Countries cross reference table |  |
| PutJobvite FTP adapter | conn-jobvite-ftp + op-jobvite-ftp-put | Protocol to confirm. |
| (no error handling) | Try/Catch + util-send-notification | New behaviour. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Start | pick Pick1 with two onMessage branches on operation process: the SOAP client endpoint (sets BindingType = Client) and the direct binding (no action). The pick has no createInstance attribute set in the source. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/BPEL/ProcessJobviteFileGen_Sch.bpel |
| 2 | Build the OTBI SQL | SELECT 0 s_0, Person Number s_1, Employee First Name s_2, Employee Last Name s_3, Employee Email Address s_4, Department Name s_5, Worker Location Name s_6, Manager E-Mail Address s_7, Address Line 1 s_8, Address Line 2 s_9, City s_10, Employee Mailing County Code s_11, Employee Mailing Region1 s_12, Worker Location Postal Code s_13, Primary Phones Number s_14, Job Name s_15 FROM "Workforce Management - Worker Assignment Real Time" WHERE "Worker"."Assignment Status" = 'Active - Payroll Eligible'. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/BPEL/ProcessJobviteFileGen_Sch.bpel |
| 3 | Run the query | Invoke ProcessHCMExtractProv.process (async provider, PROD MDS endpoint on the SOA host), then receive the processResponse callback; no timeout. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/processhcmextractprovabcs_client_ep.wsdl |
| 4 | Parse the rowset | RowVar = oraext:parseXML(return/rowset). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/BPEL/ProcessJobviteFileGen_Sch.bpel |
| 5 | Map to Jobvite CSV | xform_hcm_to_Jobvite_put.xsl: one Employee per Row (see MAP-001). The older xform_hcm_to_jobvite.xsl and the local file write (SKIPWriteJobvite) are skipped with skipCondition 1=1. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl |
| 6 | Upload to Jobvite | FTP adapter Put: eis/Ftp/Jobvite, uploads/outbound/PAN_IMPORT_EMPLOYEE_HEADER_%yyyyMMdd%.csv, ascii, Append false, one file per run. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Adapters/PutJobvite_ftp.jca |
| 7 | Callback | callbackClient invoke (processResponse) to the client partner link, executed on both entry paths. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/BPEL/ProcessJobviteFileGen_Sch.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/EmployeeId | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/FirstName | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/LastName | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/EmailAddress | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/DepartmentName | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/LocationName | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/RegionName | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/SubsidiaryName | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/ChangeRole | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/Role | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/Title | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/ReportsTo | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/Address | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/Address2 | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/City | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/State | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/PostalCode | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/Country | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/HomePhone | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/WorkPhone | string | no |
| Jobvite Employee CSV Prof | flatfile | Employees/Employee/Mobile | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | EmployeeId | Row/Column1 | passthrough |  |  | low | Person Number | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-002 | FirstName | Row/Column2 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-003 | LastName | Row/Column3 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-004 | EmailAddress | Row/Column4 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-005 | DepartmentName | (none) | hardcoded |  | empty | low | Column5 (Department Name) is queried but not mapped. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-006 | LocationName | (none) | hardcoded |  | empty | low | Column6 (Worker Location Name) is queried but not mapped. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-007 | RegionName | Row/Column12 | passthrough |  |  | low | Employee Mailing Region1. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-008 | SubsidiaryName | (none) | hardcoded |  | empty | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-009 | ChangeRole | (none) | hardcoded |  | "FALSE" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-010 | Role | (none) | hardcoded |  | "Employee" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-011 | Title | Row/Column15 | passthrough |  |  | low | Job Name. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-012 | ReportsTo | Row/Column7 | passthrough |  |  | low | Manager E-Mail Address. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-013 | Address | Row/Column8 | passthrough |  |  | low | Location Address Line 1 (work location, not home). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-014 | Address2 | Row/Column9 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-015 | City | Row/Column10 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-016 | State | (none) | hardcoded |  | empty | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-017 | PostalCode | Row/Column13 | passthrough |  |  | low | Worker Location Postal Code. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-018 | Country | Row/Column11 | transformed |  | Countries cross reference: Code to ISOCode, default empty | medium | Only PH, TW and US are defined. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Countries.dvm | OTBI rowset to Jobvite employee CSV |
| MR-019 | HomePhone | (none) | hardcoded |  | empty | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-020 | WorkPhone | Row/Column14 | passthrough |  |  | low | Primary Phones Number. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |
| MR-021 | Mobile | (none) | hardcoded |  | empty | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee CSV |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Employee population | All worker assignments with Assignment Status = Active - Payroll Eligible, on every run (full file). | High | OTBI SQL passed to hcm-otbi-sql-query. |
| BR-002 | Country code conversion | Mailing country code is converted to the ISO three-letter code via Countries (PH to PHL, TW to TWN, US to USA); unknown codes become empty. | Medium | Cross Reference Table "Countries" lookup in the map. |
| BR-003 | Fixed Jobvite role | Every employee is sent with Role = Employee and ChangeRole = FALSE. | Medium | Default values in the map. |
| BR-004 | Dated file name | File name PAN_IMPORT_EMPLOYEE_HEADER_<yyyyMMdd>.csv in uploads/outbound; same-day reruns overwrite. | Medium | Set Properties file name with the current date; FTP/SFTP SEND operation. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP async / direct binding | processjobvitefilegen_sch_client_ep / processjobvitefilegen_sch_direct_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/composite.xml |
| outbound SOAP 1.1 over HTTP (async composite) | http://129.144.156.28:80/soa-infra/services/default/ProcessHCMExtractProvABCS/processhcmextractprovabcs_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/processhcmextractprovabcs_client_ep.wsdl |
| outbound FTP (protocol and host not in export) | JNDI eis/Ftp/Jobvite, directory uploads/outbound | configured in WebLogic | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Adapters/PutJobvite_ftp.jca |
| FTP adapter PutJobvite | JNDI eis/Ftp/Jobvite; PhysicalDirectory uploads/outbound; FileNamingConvention PAN_IMPORT_EMPLOYEE_HEADER_%yyyyMMdd%.csv; FileType ascii; Append false; NumberMessages 1; NXSD Jobvite_Write.xsd. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Adapters/PutJobvite_ftp.jca |
| File adapter WriteJobvite (skipped) | JNDI eis/FileAdapter; local SOA-server directory; same file name; invoke skipped by skipCondition 1=1. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Adapters/WriteJobvite_file.jca |
| DB adapters (XXTS_SCHEDULING) | Select_XXTS_Scheduling, Update_Scheduling_StartDateTime and Update_XXTS_Schedule are declared as partner links and wired, but never invoked. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/composite.xml |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl | OTBI rowset to Jobvite employee records, including the Countries DVM lookup. | map-rowset-to-jobvite-csv with a Cross Reference Table lookup for Countries. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | None: no fault handlers or catchAll in the BPEL. |
| Retries | Adapter defaults only. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| Countries.dvm | dvm | yes |
| oramds:/apps/Provider/HCM/processhcmextractprovabcs_client_ep.wsdl | mds | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Jobvite FTP | FTP (or SFTP, to confirm) | eis/Ftp/Jobvite (host not in export) | Jobvite |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | FTP directory | uploads/outbound | [redacted].jca |

## 14. Boomi Migration Readiness Summary

This job sends Jobvite a complete list of TaskUs's active employees. When it runs it asks Oracle HCM for every worker whose assignment is "Active - Payroll Eligible", turns each one into a line of the Jobvite employee import file (ID, name, email, manager email, job title, work address and phone, region and country), and uploads the file to Jobvite's FTP folder with the date in its name.

The mapping is simple, but some details matter for Jobvite. Department and location names are queried but left blank in the file. Country is converted to a three-letter ISO code using a lookup table that has only three countries (Philippines, Taiwan, United States), so any other country goes out blank. Every employee gets the role "Employee".

The legacy job has no error handling at all, so a failed query or upload goes unnoticed. It also never pages through the HCM results, so a large workforce could be cut short without warning. In Boomi it becomes a small scheduled process that reuses the shared HCM query sub-process and adds failure notifications.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Jobvite Employee CSV Prof | prof-jobvite-employee-csv | profile.flatfile | — |
| 2 | Jobvite FTP Conn | conn-jobvite-ftp | connector-settings | — |
| 3 | Jobvite Employee File SEND Op | op-jobvite-ftp-put | connector-action | conn-jobvite-ftp |
| 4 | Map \| OTBI Rowset to Jobvite Employee CSV | map-rowset-to-jobvite-csv | transform.map | prof-otbi-rowset, prof-jobvite-employee-csv |
| 5 | Jobvite Employee File Generation Sch | proc-jobvite-filegen-sch | process | proc-sub-otbi-query, map-rowset-to-jobvite-csv, op-jobvite-ftp-put, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/BPEL/ProcessJobviteFileGen_Sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Adapters/PutJobvite_ftp.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Adapters/WriteJobvite_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_Jobvite_put.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Transformations/xform_hcm_to_jobvite.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessJobviteFileGen_Sch/SOA/Schemas/Jobvite_Write.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Countries.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/processhcmextractprovabcs_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx

## Open questions

### Still outstanding

REVIEW-NEEDED: How often, and at what time, should the Jobvite employee file be sent? (The schedule is held in a production table that is not in the export, and the job is not in the scheduler spreadsheet.)
  - If this is not answered: Once a day at 23:00 UTC, before the Jobvite consolidated job at 23:30.

REVIEW-NEEDED: Does Jobvite receive this file over FTP or SFTP, and which TaskUs team can configure the Jobvite server connection in Boomi? (The server details are configured on the legacy server, not in the export; the file contains personal data.)
  - If this is not answered: SFTP, configured by TaskUs IT directly in Boomi.

REVIEW-NEEDED: Should the file now fill in DepartmentName, LocationName and State, which are currently sent empty? (The department and location are already queried from HCM but not written to the file.)
  - If this is not answered: Keep today's behaviour (empty).

REVIEW-NEEDED: Which countries do TaskUs employees live in today, and should any country beyond Philippines, Taiwan and the United States be sent? (The country conversion table has only three entries; employees in other countries get an empty country.)
  - If this is not answered: Keep the three existing entries and report rows with no country match.

REVIEW-NEEDED: Should Jobvite keep receiving the full employee list on each run, or only changes since the last file? (Today every run sends all active employees; a large file risks being cut short by the HCM query row limit.)
  - If this is not answered: Full list, with paging so no rows are lost.

REVIEW-NEEDED: Who should be told when the file cannot be produced or uploaded? (The legacy job has no error handling, so failures go unnoticed.)
  - If this is not answered: Send to the "notification" distribution list used by the other HR jobs.

