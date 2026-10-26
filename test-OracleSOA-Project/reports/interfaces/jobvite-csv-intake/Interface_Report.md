# ProcessCallJobviteRestService

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

Polls the Jobvite folder for candidate CSV files (113 quoted, comma-separated columns with a header), converts each row into the nested Employees JSON (three previous-employment groups, one education group) and posts it, one row at a time, to the Jobvite create-employee REST service (jobvite-create-employee-rest) with a fixed Basic \[redacted\] header.

| Field | Value |
| --- | --- |
| Interface | ProcessCallJobviteRestService |
| Type | Scheduled |
| Complexity | simple |
| Components to build | 4 |
| Mapping rules | 5 |
| Business rules | 1 |

## Interface Summary

This integration watches a folder for candidate files exported from Jobvite. For each file it reads every candidate row (113 columns: personal details, IDs, address, job, schedule, salary, allowances, up to three previous employers and one education record), turns it into the format of the TaskUs create-employee service and calls that service once per candidate. The file is deleted after reading.  The call goes over plain HTTP to the integration server with a username and password written into the code. There is no error handling: if a call fails the instance stops and later rows in the file are not sent, and nobody is told. The location is cut to its first word before sending, while the salary-basis lookup later uses the part after the first space, so Jobvite hires may get no salary basis. This needs confirming with real data.  In Boomi this becomes a scheduled process reading the folder (Disk or SFTP), with a flat-file profile and one map, calling the shared validation sub-process directly instead of making an HTTP call to another integration.

**Complexity:** simple (score 18)

## Functional Analysis

1. **Poll Jobvite folder** — Read \*.csv, delete after read.
2. **Parse rows** — ReadFile.xsd (header, quoted CSV).
3. **Convert row** — xform\_Temp\_RestService to Employees JSON (one employee).
4. **Create employee** — POST to ProcessINTReadFileABCSImpl /create with Basic auth.
5. **Finish** — Wait 10 seconds after the last row.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | File (SOA server) | /u01/data/domains/SOACS-Pr\_domain/JobviteFolder/\*.csv | n/a |
| outbound | REST over plain HTTP | http://129.144.156.139:9073/soa-infra/resources/default/ProcessINTReadFileABCSImpl!2.0/CreateEmployee/create | HTTP Basic (hardcoded header) |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Jobvite flat CSV row to Employees JSON | Employee scalar fields (\~75) | same-named CSV columns | passthrough | — | — | low | — |
| MR-002 | Jobvite flat CSV row to Employees JSON | Employee/TaxableAllowance, NonTaxableAllowance | AllowanceType, Allowance | passthrough | — | — | low | Renamed. |
| MR-003 | Jobvite flat CSV row to Employees JSON | Employee/Location | Location | transformed | — | substring-before(Location, " ") | low | First word only. |
| MR-004 | Jobvite flat CSV row to Employees JSON | Employee/PreviousEmploymentDetails\[1..3\] | PreviousEmployer1..3, PreviousPosition1..3, PreviousSalary1..3, dates, address, city, postcode, reason | conditional | start date of that employer present | one group per employer | medium | — |
| MR-005 | Jobvite flat CSV row to Employees JSON | Employee/EducationDetails | Educational attainment columns | passthrough | — | one group | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | One request per candidate | Each CSV row is sent separately to the create-employee service. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Poll Jobvite folder** — Every 60 s
2. **Next candidate row**
3. **Convert to Employees JSON**
4. **POST create employee** — jobvite-create-employee-rest
5. **More rows?**
6. **Unhandled fault** — Remaining rows lost
7. **Done** — Wait 10 s

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| Jobvite Flat Candidate CSV Prof | flatfile | prof-jobvite-flat-csv |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| File adapter ReadINTFile | op-jobvite-folder-get |
| xform\_Temp\_RestService | map-jobvite-flat-to-employees |
| REST call to /create | Process Call proc-sub-int-validate-create |
| (no error handling) | Try/Catch per row + util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| op-jobvite-folder-get | Get \*.csv |
| prof-jobvite-flat-csv | Flat CSV profile |
| map-jobvite-flat-to-employees | Row to Employees map |
| proc-jobvite-csv-intake | Scheduled process |

## Client questions

- **Q-001** Which system or person puts the Jobvite candidate files in the folder, and where should Boomi pick them up (a folder on the Boomi server or an SFTP location)?
  **If this is not answered:** An SFTP folder managed by TaskUs IT.
- **Q-002** What does the "Location" value in the Jobvite file look like, and which part identifies the salary basis?
  **If this is not answered:** Send the full location and let the salary map choose the part it needs.
- **Q-003** If one candidate in a file fails, should the remaining candidates still be processed?
  **If this is not answered:** Yes; process the rest and report the failed rows.
