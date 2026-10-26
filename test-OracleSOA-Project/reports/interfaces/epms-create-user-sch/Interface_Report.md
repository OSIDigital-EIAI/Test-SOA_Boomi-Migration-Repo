# ProcessCreateEPMSUser\_sch

Test-OracleSOA-Project · Scheduled · medium complexity

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

Scheduled job that builds the EPMS new-hire profile file: it finds HCM persons created in the scheduler window, reads each person's assignment (64 columns), work relationship and person type from OTBI, maps them to the 68-field pipe-delimited EPMS profile layout (including US ADP company code, file number and position ID) and writes NewHireEPMS\_\<timestamp\>.csv to the EPMS folder on the SOA server. The FTP upload step that would send the file to EPMS is switched off.

| Field | Value |
| --- | --- |
| Interface | ProcessCreateEPMSUser\_sch |
| Type | Scheduled |
| Complexity | medium |
| Components to build | 5 |
| Mapping rules | 11 |
| Business rules | 3 |

## Interface Summary

Twice a day this job collects the people newly created in HCM since the last run and writes a profile line for each into a file for EPMS. For each person it runs three HCM queries (assignment, work relationship, person type) and fills about 70 fields: names, person and assignment IDs, business unit, project, location, job, grade, department, manager, dates, job management level and, for US staff, the ADP company code (MX4, 6PZ or AK0), file number and position ID. If a person's job name or job level is empty it warns the EPMS job team.  The file is written to a folder on the integration server; the step that uploads it to EPMS by FTP is switched off. The export does not show whether something else picks the file up, so TaskUs must confirm how EPMS receives new hires today. The map also writes four fields (country code, country name, legal entity, creative title) that exist only in the development version of the file layout.  In Boomi this becomes a scheduled process using the shared window and OTBI sub-processes, a flat-file profile and an FTP/SFTP send if delivery is confirmed.

**Complexity:** medium (score 48)

## Functional Analysis

1. **Scheduled start** — Direct binding; read XXTS window (start, end).
2. **Find new persons** — OTBI Person Real Time where Creation Date between window start and end.
3. **Per person: three queries** — Assignment (64 columns), work relationship, person type by person number.
4. **Map to EPMS record** — xform\_HCM\_EPMS; accumulate rows.
5. **Warn on missing job data** — Job Name or Job Manager Level empty -\> EmailDL EPMSJOB.
6. **Write file** — NewHireEPMS\_\<ts\>.csv in the EPMS folder.
7. **FTP (disabled)** — Java lftp put to /HCM/PROD skipped.
8. **Advance window** — Update start date and last run; callback.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding / SOAP async | processcreateepmsuser\_DirectBinding / \_sch\_client\_ep | none declared |
| outbound | SOAP async (composite) | oramds:/apps/Provider/HCM/processhcmextractprovabcs\_client\_ep.wsdl | none declared |
| outbound | File (SOA server) | /u01/data/domains/SOACS-Pr\_domain/EPMSFolder/NewHireEPMS\_\<ts\>.csv | n/a |
| outbound | FTP via shell lftp (disabled) | IP-addressed host \[redacted\], directory /HCM/PROD | user/password hardcoded in Java \[redacted\] |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | OTBI assignment to EPMS profile | FirstName, MiddleName, LastName, PreviousLastName, NameSuffix, DisplayName, WorkEmail | Assignment Column29, 31, 30, 33, 32, 27, 28 | passthrough | Assignment status != INACTIVE | — | low | — |
| MR-002 | OTBI assignment to EPMS profile | PersonNumber, PersonId, UserName, AssignmentId, AssignmentNumber | Column38, 37, 40, 39/17, 25 | passthrough | — | — | low | — |
| MR-003 | OTBI assignment to EPMS profile | PersonTypeId / PersonTypeEffectiveDate / PersonTypeModifyDate | Person type query Column5, 1, 2 | passthrough | — | — | low | — |
| MR-004 | OTBI assignment to EPMS profile | BusinessUnit, Project, Location, Job, Grade, Department, Position fields with ids and dates | Assignment columns 1-15, 41-58 | passthrough | — | — | medium | GradeName and GradeCode both from Column4. |
| MR-005 | OTBI assignment to EPMS profile | AssignmentCategory, StartTime, EndTime, ActionCode, ActionReasonCode, AssignmentStatus, assignment dates | Column24, 23, 22, 16, 21, 18, 20, 19 | passthrough | — | — | low | — |
| MR-006 | OTBI assignment to EPMS profile | ManagerId, ManagerPersonNumber, ManagerAssignmentId, OriginalHireDate, LegalEntityId, CreationDate, LastUpdateDate | Column36, 47, 35, 50, 8, 26, 34 | passthrough | — | — | low | — |
| MR-007 | OTBI assignment to EPMS profile | PeriodOfServiceId | Work relationship Column1 | passthrough | — | — | low | — |
| MR-008 | OTBI assignment to EPMS profile | JobLevel, JobManagementLevel ID/Name/dates | Column55, 61, 60, 54, 53 | passthrough | — | — | low | — |
| MR-009 | OTBI assignment to EPMS profile | ADPCompanyCode | Column62 (legal employer), Column11 (location), Column63 (country) | conditional | Country US | TaskUs USA LLC + Santa Monica -\> MX4; TaskUs USA LLC -\> 6PZ; TaskUs Inc. -\> AK0 | medium | Same rule as hcm-external-identifier-sch. |
| MR-010 | OTBI assignment to EPMS profile | FileNumber / PositionID | Column38 | calculated | Country US | substring(PersonNumber, 2) / ADP code + FileNumber | low | — |
| MR-011 | OTBI assignment to EPMS profile | CountryCode, CountryName, LegalEntityName, CreativeTitle | Column63, 64, 62, 7 | passthrough | — | — | low | Not in the PROD\_MDS EPMSWrite.xsd; present in the DEV\_MDS copy. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | New persons in window | Persons whose Creation Date falls in the XXTS window. | High |
| BR-002 | ADP codes for US | ADP company code, file number and position ID only for US workers by legal employer/location. | High |
| BR-003 | Job data completeness | Warn when Job Name or Job Manager Level is empty. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Scheduled start** — 23:00 / 11:00 UTC
2. **Find new persons in window** — OTBI
3. **Any persons?**
4. **Query assignment, relationship, type** — Per person
5. **Map to EPMS row**
6. **Job data missing?**
7. **Warn EPMS job team** — EmailDL EPMSJOB
8. **More persons?**
9. **Write EPMS file** — FTP step disabled
10. **Advance window**

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| EPMS Employee Profile Flat File Prof | flatfile | prof-epms-profile |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| XXTS window | Sub: Schedule Window Get and Advance (newhire-emp-sch) |
| Async OTBI provider x4 | Process Call hcm-otbi-sql-query |
| xform\_HCM\_EPMS | map-hcm-to-epms |
| File adapter + disabled Java lftp | conn-epms-ftp + op-epms-put |

**Components**

| Component | Role |
| --- | --- |
| conn-epms-ftp | EPMS FTP/SFTP connection |
| op-epms-put | Send profile file |
| prof-epms-profile | Profile file layout |
| map-hcm-to-epms | Profile map |
| proc-epms-create-user-sch | Scheduled process |

## Client questions

- **Q-001** How does EPMS receive the new-hire file today, given that the upload step in the integration is switched off?
  **If this is not answered:** Boomi uploads the file to the EPMS server by SFTP.
- **Q-002** Should the EPMS profile file include country code, country name, legal entity and creative title?
  **If this is not answered:** Use the PROD layout (with JobCode, without the four extra fields), as the Update job does.
- **Q-003** Which EPMS server, account and protocol should Boomi use to deliver files?
  **If this is not answered:** SFTP to the existing server with a new service account.
