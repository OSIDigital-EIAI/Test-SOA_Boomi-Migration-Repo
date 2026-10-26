# ProcessIncompleteHours\_Sch

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

Report job that calculates, for Philippines employees in certain job categories, how many hours short of 40 each employee was in each of up to four payroll weeks, using an Oracle HCM BI Publisher "SOA Payable Time Report", and writes a CSV (EmployeeId, W1–W4 hours) to the ADP folder on the SOA server.

| Field | Value |
| --- | --- |
| Interface | ProcessIncompleteHours\_Sch |
| Type | Scheduled |
| Complexity | medium |
| Components to build | 4 |
| Mapping rules | 3 |
| Business rules | 3 |

## Interface Summary

This job works out how many hours each eligible Philippines employee was short of a 40-hour week, for up to four payroll weeks. It lists active, payroll-eligible employees in the Philippines whose job is in category 30, or category 20 and not eligible for overtime/night differential, then runs an HCM report of each employee's payable time for each week. Approved regular hours, planned leave, birthday leave and unplanned leave count towards the 40 hours; the shortfall is written to a CSV file on the integration server.  The weeks are taken from a payroll calendar table, and a week is only processed when today's date is exactly one of the pay dates in that table. The table only covers pay dates from February 2018 to January 2019, so today no week is processed and the file contains employee numbers with no hours. Nothing in the export reads the CSV.  A related ADP job (adp-transactional-file-sch) calculates incomplete hours differently (8 hours per time entry). TaskUs needs to confirm whether this report is still needed, which rule is right, and where the payroll calendar should come from. If it is kept, Boomi would run it on a schedule, call the BI Publisher report through the SOAP connector and write the CSV wherever it is needed.

**Complexity:** medium (score 38)

## Functional Analysis

1. **Start** — SOAP process request.
2. **List eligible employees** — OTBI active PH payroll-eligible; filter job 20 not OTND or 30.
3. **Per employee and week** — If today is a pay date in PayrollWeeks with week N dates, run the payable time report.
4. **Decode report** — Base64 decode, normalise G\_2 to G\_1, parse.
5. **Calculate shortfall** — 40 minus approved counted hours per week.
6. **Write CSV** — WeeklyIncompleteHrs\<yyyyMMdd\>.csv to the ADP folder.
7. **Callback** — processResponse to the caller.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP async | processincompletehours\_sch\_client\_ep | none declared |
| outbound | SOAP async (composite) | oramds:/apps/Provider/HCM/processhcmextractprovabcs\_client\_ep.wsdl | none declared |
| outbound | SOAP | oramds:/apps/Provider/HCM/PublicReportService.wsdl | user ID and password in the request body (Credentials.dvm, redacted) |
| outbound | File | /u01/app/oracle/tools/paas/state/homes/oracle/ADP\_OSI\_Taskus/WeeklyIncompleteHrs\<yyyyMMdd\>.csv | n/a |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Payable time report to weekly incomplete hours | W1HrsCount … W4HrsCount | DATA\_DS/G\_1/MEASURE | calculated | Week N processed; STATUS = Approved and ELEMENT in (Regular Hours, Planned Leave, Birthday, Unplanned) | 40 - sum(MEASURE), format \#\#.\#\# | medium | — |
| MR-002 | Employees to CSV | EmployeeId | Employee/EmployeeId (OTBI Column4 Person Number) | passthrough | — | — | low | — |
| MR-003 | Employees to CSV | W1HrsCount … W4HrsCount | Employee/WnHrsCount | conditional | Value not empty | format-number \#\#.\#\# | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Eligible employees | Active - Payroll Eligible, PH, and (job 20 with Not Eligible for OTND) or job 30. | High |
| BR-002 | Run only on pay dates | Week N processed only when today is a PayrollWeeks key with week N dates. | High |
| BR-003 | 40-hour week | Shortfall = 40 - approved Regular Hours, Planned Leave, Birthday, Unplanned. | High |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Start (SOAP request)** — No scheduler adapter
2. **List eligible PH employees** — OTBI + filter
3. **Today a pay date?** — PayrollWeeks.dvm per week
4. **Run payable time report** — BIP runReport, up to 4 weeks
5. **Decode and calculate shortfall**
6. **More employees?**
7. **Write CSV**
8. **Callback**

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| HCM Payable Time Report XML Prof | xml | prof-payable-time-report |
| Weekly Incomplete Hours CSV Prof | flatfile | prof-incomplete-hours-csv |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| Async OTBI provider | Process Call hcm-otbi-sql-query |
| PublicReportService runReport | Web Services SOAP Client connector |
| Base64DecoderClass | Data Process Base64 decode |
| xform\_TimeReportWeekly\_IncompleteHrs | map-time-report-to-incomplete-hours |
| File adapter CSV | Disk/SFTP connector with prof-incomplete-hours-csv |

**Components**

| Component | Role |
| --- | --- |
| prof-payable-time-report | BIP report XML |
| prof-incomplete-hours-csv | Output CSV |
| map-time-report-to-incomplete-hours | Shortfall calculation |
| proc-incomplete-hours-sch | Scheduled process |

## Client questions

- **Q-001** Is the weekly incomplete-hours report still required, who uses it, and where should the file go?
  **If this is not answered:** Not migrated until a consumer is identified.
- **Q-002** Where should payroll-week start and end dates come from in future?
  **If this is not answered:** A Boomi cross-reference table maintained by payroll each year.
- **Q-003** Is an incomplete week measured against 40 hours with regular hours and planned, birthday and unplanned leave counting, or as 8 hours per time entry as in the ADP transactional file?
  **If this is not answered:** 40 hours per week with the four counted types.
- **Q-004** How often should the report run?
  **If this is not answered:** On each pay date (3rd and 18th).
