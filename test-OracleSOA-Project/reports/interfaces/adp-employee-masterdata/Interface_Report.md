# ProcessEmployeeADPProvABCS

Test-OracleSOA-Project · Listener · complex complexity

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

Asynchronous web service that receives a batch of employee events (new hire, rehire, termination, company transfer, cost-centre change, name change, tax/dependants change, suspension, return to work, work-schedule change, basic pay, previous employer), enriches each employee from Oracle HCM with up to eleven OTBI queries, and produces the ADP GlobalView G2 HR master-data (HRMD) file for the Philippines payroll: SAP-infotype records P0000–P0709 plus a company-transfer set, between a header and a trailer. The file is written to the SOA server and PGP-encrypted by a shell script; the step that uploads it to ADP is switched off.

| Field | Value |
| --- | --- |
| Interface | ProcessEmployeeADPProvABCS |
| Type | Listener |
| Complexity | complex |
| Components to build | 6 |
| Mapping rules | 11 |
| Business rules | 5 |

## Interface Summary

This service builds the employee master-data file that ADP needs to run TaskUs's Philippines payroll. It is given a list of employees and what happened to each (hired, rehired, terminated, transferred, changed name and so on). For each employee it collects personal, job, pay, bank, tax and government-ID details from HCM and turns them into the record types ADP's G2 interface expects, translating TaskUs values into ADP codes through fourteen lookup tables (legal entity, personnel area, cost centre, employee group and others).  The finished file is saved on the integration server and encrypted with ADP's PGP key by a script. The upload to ADP is switched off in the code, so nothing in the export shows how ADP receives the file today.  Several defects in the mapping would put wrong data into payroll if copied as-is: with more than one employee in a file, title, gender and marital status come from the first employee; the "no TIN" value becomes 0 instead of nine zeros; dependants' birth date and gender are taken from the employee; and one query filter is missing for the first employee. Payroll must decide whether each is fixed, and the result should be validated against ADP's Philippines specification. In Boomi this becomes a sub-process with a large flat-file map, cross-reference tables, a PGP-encrypt step and an SFTP send.

**Complexity:** complex (score 95)

## Functional Analysis

1. **Receive employee batch** — Async SOAP process(EmployeeRequest).
2. **Per employee: read HCM data** — Up to eleven OTBI queries by person number; status filter depends on terminate vs other actions.
3. **Derive codes** — Employee group/subgroup (EmpGrpSGrp), location sub-area (LocationNameCode), rehire same-employer flag (CompCode).
4. **Build HRMD records** — xFORM\_NEWHIRE\_HCM\_TO\_FILE by action.
5. **Set trailer count** — RECCT = number of records + 2.
6. **Write file** — PP3208\_\<ts\>\_PHOHCM\_HRMD01\_DUT8G2I.SAP to the ADP folder.
7. **PGP encrypt** — FileEncryption runs a shell script producing .pgp.
8. **Upload to ADP (disabled)** — Read .pgp and FTP put to /put — skipped.
9. **Callback** — processResponse with the script output.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP async | processemployeeadpprovabcs\_client\_ep | none declared |
| outbound | SOAP async (composite) | oramds:/apps/Provider/HCM/processhcmextractprovabcs\_client\_ep.wsdl | none declared |
| outbound | File | /u01/app/oracle/tools/paas/state/homes/oracle/ADP\_OSI\_Taskus/PP3208\_\<ts\>\_PHOHCM\_HRMD01\_DUT8G2I.SAP | n/a |
| outbound | FTP (disabled) | eis/Ftp/ADP, /put | defined in the server connection factory (not in export) |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Employee EBM to ADP HRMD records | HEADER | (constants), current date/time | hardcoded | First employee | RECTY HEADR, USRID GVETASKUS, CLNAM TaskUs, CNAME DMEKA, FNAME MD, RUNDT, RUNTM, RUNTY P, VERNO 1 | low | Confirm with ADP. |
| MR-002 | Employee EBM to ADP HRMD records | BEGDA (B1 records) | EmploymentAssignments/EffectiveStartDate | transformed | — | yyyyMMdd; if earlier than 20170926 use 20170926 | low | Hardcoded floor. |
| MR-003 | Employee EBM to ADP HRMD records | P0000 MASSN | CustomAttributes/Description | conditional | Action | newhire 01, rehire 12, terminate 10 (BEGDA termination+1), companytransfer/costcenterchange 02 | medium | — |
| MR-004 | Employee EBM to ADP HRMD records | P0001 BUKRS, WERKS, PERSG, PERSK, BTRTL, KOSTL, ANSVH, ABKRS | Legal employer, location, employee group text, project id | transformed | newhire, rehire, costcenterchange | DVM LegalEntity, PersonnelArea, EmployeeGroup, PersonnelSubArea, CCenter, ProjectID; ABKRS 7M | high | — |
| MR-005 | Employee EBM to ADP HRMD records | P0002 ANRED, GESCH, FAMST | /EmployeeRequest/Employee/... (absolute path) | transformed | newhire, rehire, namechange, taxdependentschange | DVM Title, Gender, MaritalStatus | medium | Defect: absolute path returns the first employee's value for every employee. |
| MR-006 | Employee EBM to ADP HRMD records | P0021 FGBDT, FASEX | ../../ContactDetails (employee level) | looped | Per dependant (Child / non-Child sets) | DVM FamilyType, Gender | medium | Defect: values are employee-level, not per dependant. |
| MR-007 | Employee EBM to ADP HRMD records | P0411 TAXFN | Person/NationalID\_Type/TINNumber | conditional | TIN present | remove dashes; else select='000000000' | low | Defect: numeric literal yields 0. |
| MR-008 | Employee EBM to ADP HRMD records | P0411 NODEP / TXRDO | count(Person/Dependent); Organization name | calculated | — | count; DVM LegalEntity rdc | low | — |
| MR-009 | Employee EBM to ADP HRMD records | P0008 wage lines | Person/Visa (wage type rows) | looped | newhire, rehire, basicpay | Wage codes 1000, 3001–3004 into LGA/BET pairs | medium | Wage data carried in the Visa structure. |
| MR-010 | Employee EBM to ADP HRMD records | P0634 | Person/PreEmployerDetail | looped | newhire, rehire, previousemployer | — | medium | Defect: for previousemployer the CLIID path ../../Employee/EmployeeID resolves to nothing, so CLIID is empty. |
| MR-011 | Employee EBM to ADP HRMD records | TRAILER RECCT | generated records | calculated | Last employee | XSL writes 15\*employees+2; BPEL overwrites with actual record count + 2 | low | fn-adp-trailer-count. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Action drives record set | Each action produces a defined set of infotype records (see MigrationManifest 8.1 table). | High |
| BR-002 | Payroll-eligible status filter | Terminations read "Inactive - Payroll Eligible" assignments; all other actions "Active - Payroll Eligible". | High |
| BR-003 | Rehire to same employer | If the last two legal-employer history rows are the same company on rehire, P0709 is sent. | Medium |
| BR-004 | Code translation | TaskUs values are translated to ADP codes through 14 DVMs (e.g. LegalEntity 6 rows, CCenter 290 rows). | High |
| BR-005 | Encrypt before send | File is PGP-encrypted for ADP. | High |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive employee batch** — Async SOAP
2. **Next employee** — Sequential forEach
3. **Query HCM details** — Up to 11 OTBI queries
4. **Employee error?**
5. **Email error, continue** — EmailDL notification
6. **More employees?**
7. **Build HRMD file** — XSLT + trailer count
8. **Write and PGP encrypt** — Shell script
9. **Upload to ADP (disabled)** — skipCondition 1=1
10. **Callback**

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| ADP G2 HRMD Master Data Flat File Prof | flatfile | prof-adp-md-flatfile |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| SOAP async service | Process Call target (sub-process) or Web Services Server listener |
| HCMExtract async provider | Process Call hcm-otbi-sql-query |
| xFORM\_NEWHIRE\_HCM\_TO\_FILE + 14 DVMs | map-ebm-to-adp-md + Cross Reference Tables |
| RECCT assign | fn-adp-trailer-count |
| FileEncryption shell | PGP Encrypt shape |
| eis/Ftp/ADP put (disabled) | op-adp-put on conn-adp-sftp |

**Components**

| Component | Role |
| --- | --- |
| conn-adp-sftp | ADP GVS connection |
| op-adp-put | Send HRMD file |
| prof-adp-md-flatfile | HRMD layout |
| map-ebm-to-adp-md | Record mapping |
| fn-adp-trailer-count | Trailer count |
| proc-adp-employee-masterdata | Process |

## Client questions

- **Q-001** How does the employee master-data file reach ADP today, and how should Boomi deliver it?
  **If this is not answered:** Boomi PGP-encrypts the file and sends it by SFTP to the ADP GVS endpoint ADP specifies.
- **Q-002** For each mapping defect found (first employee's title, gender and marital status reused; TIN default 0 instead of nine zeros; dependants' birth date and gender taken from the employee; empty previous-employer client ID; missing status filter on the first employee), should Boomi fix it or reproduce today's output?
  **If this is not answered:** Fix all of them and validate the file with ADP before go-live.
- **Q-003** Are the ADP identifiers still current: client user GVETASKUS, company DMEKA, payroll area 7M, earliest start date 26 Sep 2017 and the file name PP3208\_…\_PHOHCM\_HRMD01\_DUT8G2I.SAP?
  **If this is not answered:** Keep the current values as environment-configurable properties.
- **Q-004** Who should be told when an employee cannot be included in the ADP file?
  **If this is not answered:** Send a clear per-employee error summary to the payroll team list.
