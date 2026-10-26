# ProcessJobviteFileGen\_Sch

Test-OracleSOA-Project · Scheduled · simple complexity

[← All interfaces](../../Extraction_Report.md)

- [Interface Overview](#interface-overview)
- [Interface Summary](#interface-summary)
- [Functional Analysis](#functional-analysis)
- [Technical Analysis](#technical-analysis)
- [Mappings](#mappings)
- [Business Rules](#business-rules)
- [Flow Diagram](#flow-diagram)
- [Data Model](#data-model)
- [Architecture](#architecture)

---

## Interface Overview

Builds a full employee file for Jobvite from Oracle HCM Cloud: every worker whose assignment is Active - Payroll Eligible is extracted through OTBI, mapped to the Jobvite employee import CSV layout and uploaded by FTP to uploads/outbound as PAN\_IMPORT\_EMPLOYEE\_HEADER\_\<yyyyMMdd\>.csv.

| Field | Value |
| --- | --- |
| Interface | ProcessJobviteFileGen\_Sch |
| Type | Scheduled |
| Complexity | simple |
| Components to build | 5 |
| Mapping rules | 21 |
| Business rules | 4 |

## Interface Summary

This job sends Jobvite a complete list of TaskUs's active employees. When it runs it asks Oracle HCM for every worker whose assignment is "Active - Payroll Eligible", turns each one into a line of the Jobvite employee import file (ID, name, email, manager email, job title, work address and phone, region and country), and uploads the file to Jobvite's FTP folder with the date in its name.  The mapping is simple, but some details matter for Jobvite. Department and location names are queried but left blank in the file. Country is converted to a three-letter ISO code using a lookup table that has only three countries (Philippines, Taiwan, United States), so any other country goes out blank. Every employee gets the role "Employee".  The legacy job has no error handling at all, so a failed query or upload goes unnoticed. It also never pages through the HCM results, so a large workforce could be cut short without warning. In Boomi it becomes a small scheduled process that reuses the shared HCM query sub-process and adds failure notifications.

**Complexity:** simple (score 22)

## Functional Analysis

1. **Start** — pick Pick1 with two onMessage branches on operation process: the SOAP client endpoint (sets BindingType = Client) and the direct binding (no action). The pick has no createInstance attribute set in the source.
2. **Build the OTBI SQL** — SELECT 0 s\_0, Person Number s\_1, Employee First Name s\_2, Employee Last Name s\_3, Employee Email Address s\_4, Department Name s\_5, Worker Location Name s\_6, Manager E-Mail Address s\_7, Address Line 1 s\_8, Address Line 2 s\_9, City s\_10, Employee Mailing County Code s\_11, Employee Mailing Region1 s\_12, Worker Location Postal Code s\_13, Primary Phones Number s\_14, Job Name s\_15 FROM "Workforce Management - Worker Assignment Real Time" WHERE "Worker"."Assignment Status" = 'Active - Payroll Eligible'.
3. **Run the query** — Invoke ProcessHCMExtractProv.process (async provider, PROD MDS endpoint on the SOA host), then receive the processResponse callback; no timeout.
4. **Parse the rowset** — RowVar = oraext:parseXML(return/rowset).
5. **Map to Jobvite CSV** — xform\_hcm\_to\_Jobvite\_put.xsl: one Employee per Row (see MAP-001). The older xform\_hcm\_to\_jobvite.xsl and the local file write (SKIPWriteJobvite) are skipped with skipCondition 1=1.
6. **Upload to Jobvite** — FTP adapter Put: eis/Ftp/Jobvite, uploads/outbound/PAN\_IMPORT\_EMPLOYEE\_HEADER\_%yyyyMMdd%.csv, ascii, Append false, one file per run.
7. **Callback** — callbackClient invoke (processResponse) to the client partner link, executed on both entry paths.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP async / direct binding | processjobvitefilegen\_sch\_client\_ep / processjobvitefilegen\_sch\_direct\_ep | none declared |
| outbound | SOAP 1.1 over HTTP (async composite) | http://129.144.156.28:80/soa-infra/services/default/ProcessHCMExtractProvABCS/processhcmextractprovabcs\_client\_ep | none declared |
| outbound | FTP (protocol and host not in export) | JNDI eis/Ftp/Jobvite, directory uploads/outbound | configured in WebLogic |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | OTBI rowset to Jobvite employee CSV | EmployeeId | Row/Column1 | passthrough | — | — | low | Person Number |
| MR-002 | OTBI rowset to Jobvite employee CSV | FirstName | Row/Column2 | passthrough | — | — | low | — |
| MR-003 | OTBI rowset to Jobvite employee CSV | LastName | Row/Column3 | passthrough | — | — | low | — |
| MR-004 | OTBI rowset to Jobvite employee CSV | EmailAddress | Row/Column4 | passthrough | — | — | low | — |
| MR-005 | OTBI rowset to Jobvite employee CSV | DepartmentName | (none) | hardcoded | — | empty | low | Column5 (Department Name) is queried but not mapped. |
| MR-006 | OTBI rowset to Jobvite employee CSV | LocationName | (none) | hardcoded | — | empty | low | Column6 (Worker Location Name) is queried but not mapped. |
| MR-007 | OTBI rowset to Jobvite employee CSV | RegionName | Row/Column12 | passthrough | — | — | low | Employee Mailing Region1. |
| MR-008 | OTBI rowset to Jobvite employee CSV | SubsidiaryName | (none) | hardcoded | — | empty | low | — |
| MR-009 | OTBI rowset to Jobvite employee CSV | ChangeRole | (none) | hardcoded | — | "FALSE" | low | — |
| MR-010 | OTBI rowset to Jobvite employee CSV | Role | (none) | hardcoded | — | "Employee" | low | — |
| MR-011 | OTBI rowset to Jobvite employee CSV | Title | Row/Column15 | passthrough | — | — | low | Job Name. |
| MR-012 | OTBI rowset to Jobvite employee CSV | ReportsTo | Row/Column7 | passthrough | — | — | low | Manager E-Mail Address. |
| MR-013 | OTBI rowset to Jobvite employee CSV | Address | Row/Column8 | passthrough | — | — | low | Location Address Line 1 (work location, not home). |
| MR-014 | OTBI rowset to Jobvite employee CSV | Address2 | Row/Column9 | passthrough | — | — | low | — |
| MR-015 | OTBI rowset to Jobvite employee CSV | City | Row/Column10 | passthrough | — | — | low | — |
| MR-016 | OTBI rowset to Jobvite employee CSV | State | (none) | hardcoded | — | empty | low | — |
| MR-017 | OTBI rowset to Jobvite employee CSV | PostalCode | Row/Column13 | passthrough | — | — | low | Worker Location Postal Code. |
| MR-018 | OTBI rowset to Jobvite employee CSV | Country | Row/Column11 | transformed | — | Countries cross reference: Code to ISOCode, default empty | medium | Only PH, TW and US are defined. |
| MR-019 | OTBI rowset to Jobvite employee CSV | HomePhone | (none) | hardcoded | — | empty | low | — |
| MR-020 | OTBI rowset to Jobvite employee CSV | WorkPhone | Row/Column14 | passthrough | — | — | low | Primary Phones Number. |
| MR-021 | OTBI rowset to Jobvite employee CSV | Mobile | (none) | hardcoded | — | empty | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Employee population | All worker assignments with Assignment Status = Active - Payroll Eligible, on every run (full file). | High |
| BR-002 | Country code conversion | Mailing country code is converted to the ISO three-letter code via Countries (PH to PHL, TW to TWN, US to USA); unknown codes become empty. | Medium |
| BR-003 | Fixed Jobvite role | Every employee is sent with Role = Employee and ChangeRole = FALSE. | Medium |
| BR-004 | Dated file name | File name PAN\_IMPORT\_EMPLOYEE\_HEADER\_\<yyyyMMdd\>.csv in uploads/outbound; same-day reruns overwrite. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Start (schedule unknown)** — SOAP client or direct binding
2. **Query active employees** — OTBI full extract via hcm-otbi-sql-query
3. **Map rows to Jobvite CSV** — Countries lookup for Country
4. **Upload file to Jobvite** — uploads/outbound/PAN\_IMPORT\_EMPLOYEE\_HEADER\_\<yyyyMMdd\>.csv
5. **Callback / end**
6. **Unhandled fault** — No fault handler in legacy; Boomi adds notification

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| Jobvite Employee CSV Prof | flatfile | prof-jobvite-employee-csv |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| ProcessHCMExtractProvABCS (async) | Process Call hcm-otbi-sql-query |
| xform\_hcm\_to\_Jobvite\_put.xsl + Countries.dvm | map-rowset-to-jobvite-csv + Countries cross reference table |
| PutJobvite FTP adapter | conn-jobvite-ftp + op-jobvite-ftp-put |
| (no error handling) | Try/Catch + util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| conn-jobvite-ftp | Jobvite FTP/SFTP connection |
| op-jobvite-ftp-put | SEND operation to uploads/outbound |
| prof-jobvite-employee-csv | Target flat-file profile |
| map-rowset-to-jobvite-csv | Rowset to CSV map |
| proc-jobvite-filegen-sch | Main scheduled process |

## Client questions

- **Q-001** How often, and at what time, should the Jobvite employee file be sent?
  **If this is not answered:** Once a day at 23:00 UTC, before the Jobvite consolidated job at 23:30.
- **Q-002** Does Jobvite receive this file over FTP or SFTP, and which TaskUs team can configure the Jobvite server connection in Boomi?
  **If this is not answered:** SFTP, configured by TaskUs IT directly in Boomi.
- **Q-003** Should the file now fill in DepartmentName, LocationName and State, which are currently sent empty?
  **If this is not answered:** Keep today's behaviour (empty).
- **Q-004** Which countries do TaskUs employees live in today, and should any country beyond Philippines, Taiwan and the United States be sent?
  **If this is not answered:** Keep the three existing entries and report rows with no country match.
- **Q-005** Should Jobvite keep receiving the full employee list on each run, or only changes since the last file?
  **If this is not answered:** Full list, with paging so no rows are lost.
- **Q-006** Who should be told when the file cannot be produced or uploaded?
  **If this is not answered:** Send to the "notification" distribution list used by the other HR jobs.
