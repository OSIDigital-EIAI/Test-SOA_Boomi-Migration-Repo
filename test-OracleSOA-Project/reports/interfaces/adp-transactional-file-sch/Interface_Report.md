# ProcessADPTransProvABCS

Test-OracleSOA-Project · Scheduled · complex complexity

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

Scheduled job that produces the ADP GlobalView G2 transactional (TR) payroll file for the Philippines: for every active, payroll-eligible employee with a PH mailing country it reads the employee's element entries (wage codes and amounts) and, for certain job types, their reported time cards, then writes P0014 (recurring) and P0015 (one-off) wage records and P2010 incomplete-hours records between a header and trailer. The file is PGP-encrypted on the SOA server; the upload to ADP is switched off.

| Field | Value |
| --- | --- |
| Interface | ProcessADPTransProvABCS |
| Type | Scheduled |
| Complexity | complex |
| Components to build | 4 |
| Mapping rules | 6 |
| Business rules | 4 |

## Interface Summary

This job builds the payroll transactions file that ADP uses alongside the master data to pay TaskUs Philippines staff. Each run it lists every active, payroll-eligible employee in the Philippines, reads their pay elements from HCM (allowances, adjustments and other wage codes) and, for employees in certain job categories, works out how many hours were missing from their time cards over two payroll weeks. These become ADP recurring-payment, one-off payment and incomplete-hours records.  As exported the incomplete-hours part cannot work correctly: the payroll-week calendar it reads is looked up with a column name and a date that do not exist in the lookup table, the second week's end date uses the first week's value, and the run window is never moved forward. The file is encrypted but, as with the master data, the upload to ADP is switched off.  In Boomi this becomes a scheduled process that reuses the OTBI sub-process, a script for incomplete hours, a flat-file map and the ADP SFTP connection from adp-employee-masterdata. The payroll calendar and the incomplete-hours rule must be confirmed first.

**Complexity:** complex (score 76)

## Functional Analysis

1. **Start and read window** — Direct binding or SOAP client; XXTS window read into StartDate/EndDate.
2. **List employees** — OTBI active payroll-eligible PH employees.
3. **Incomplete hours (eligible jobs)** — Time-card queries for two payroll weeks; hours shortfall per week.
4. **Wage elements** — Element entries per employee.
5. **Build TR file** — Header, P0014, P0015, P2010, trailer.
6. **Write and encrypt** — Write .SAP, PGP to .pgp via Java embedding.
7. **Upload (disabled)** — Read .pgp and FTP put — skipped.
8. **Update window and reply** — Scheduler update (values missing) or client callback.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding / SOAP async | processadptransprovabcs\_DirectBinding / \_client\_ep | none declared |
| outbound | SOAP async (composite) | oramds:/apps/Provider/HCM/processhcmextractprovabcs\_client\_ep.wsdl | none declared |
| outbound | File | /u01/app/oracle/tools/paas/state/homes/oracle/ADP\_OSI\_Taskus/PP3208\_\<ts\>\_PHOHCM\_TRAN01\_DUT8G2I.SAP | n/a |
| outbound | FTP (disabled) | eis/Ftp/FtpAdapter | server connection factory (not in export) |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Employee wage elements to ADP TR records | HEADER | (constants) | hardcoded | First employee | FNAME TR; other header constants as HRMD | low | — |
| MR-002 | Employee wage elements to ADP TR records | P0014 (SUBTY, LGART, BETRG, WAERS PHP) | Person/Visa VisaPermitNumber (wage code), VisaPermitType (amount) | looped | Code in {3000,3001,3002,3003,3004,3027,3040,3044,3045,3046,3047,3050,3051,3100,3130,3200,3201,7001,9372,9374,9380,9384} and amount not in {N, PRD, "", Semi-Month, Year, Y, 0} | — | medium | 22 blocks. |
| MR-003 | Employee wage elements to ADP TR records | P0015 | Person/Visa | looped | Code in the P0015 list (42 blocks incl. 3024, 3031, 3034, 3042, 3043, 3060, 3061, 3070, 3101–3104, 3110–3112, 3120, 3121, 3131–3134, 3150, 3151, 5001, 5002) and valid amount | — | medium | — |
| MR-004 | Employee wage elements to ADP TR records | BEGDA | EmploymentAssignments/EffectiveStartDate | transformed | — | yyyyMMdd, floor 20170926 | low | — |
| MR-005 | Employee wage elements to ADP TR records | P2010 SUBTY 2051, STDAZ | Person/Dependent/PersonalCode (incomplete hours) | conditional | Eligible job and hours \> 0 | — | medium | — |
| MR-006 | Employee wage elements to ADP TR records | TRAILER RECCT | records | calculated | — | P0014 + P0015 + P2010 + 2 | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Population | Active - Payroll Eligible employees with PH mailing country. | High |
| BR-002 | Incomplete-hours eligibility | (JobCode 20 and Not Eligible for OTND) or JobCode 30. | High |
| BR-003 | Incomplete hours | Per payroll week: 8 h x number of time entries minus reported hours; send if \> 0. | High |
| BR-004 | Valid amounts only | Skip amounts N, PRD, empty, Semi-Month, Year, Y, 0. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Scheduled start** — Schedule unknown
2. **List PH payroll employees** — OTBI
3. **Incomplete-hours eligible?** — JobCode 20/30
4. **Calculate incomplete hours** — Time cards, 2 weeks
5. **Read wage elements** — OTBI
6. **More employees?**
7. **Build TR file**
8. **Write and PGP encrypt**
9. **Upload to ADP (disabled)**
10. **Update scheduler** — Values not set

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| ADP G2 Transactional Flat File Prof | flatfile | prof-adp-tr-flatfile |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| XXTS DB adapters | Sub: Schedule Window Get and Advance |
| Async OTBI provider | Process Call hcm-otbi-sql-query |
| Incomplete-hours XSL + BPEL arithmetic | fn-incomplete-hours |
| Xform\_Hcm\_to\_TransactionalFile | map-ebm-to-adp-tr |
| FileEncryption + FTP | PGP Encrypt + op-adp-put (adp-employee-masterdata) |

**Components**

| Component | Role |
| --- | --- |
| prof-adp-tr-flatfile | TR layout |
| fn-incomplete-hours | Hours shortfall |
| map-ebm-to-adp-tr | TR mapping |
| proc-adp-transactional-file-sch | Scheduled process |

## Client questions

- **Q-001** Where should the payroll-week dates for incomplete hours come from, and which rule applies: 8 hours per time entry or 40 hours per week?
  **If this is not answered:** Use the PayrollWeeks table keyed by the current payroll period start and 8 hours per time entry, as coded.
- **Q-002** How often and at what time should the ADP transactional file be produced?
  **If this is not answered:** Once per payroll cut-off, triggered manually until a schedule is agreed.
- **Q-003** Should the transactional file be delivered to ADP the same way as the master-data file?
  **If this is not answered:** Same PGP + SFTP delivery as the master-data file.
- **Q-004** Are the wage codes in the recurring (P0014) and one-off (P0015) lists still complete and correct?
  **If this is not answered:** Keep the current lists in a configurable cross-reference table.
