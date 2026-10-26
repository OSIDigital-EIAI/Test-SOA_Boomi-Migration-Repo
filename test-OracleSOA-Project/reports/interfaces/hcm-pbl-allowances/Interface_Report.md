# ProcessPBLProvABCSImpl

Test-OracleSOA-Project · Listener · medium complexity

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

Creates payroll element entries for new hires' allowances in Oracle HCM Payroll. From the Employee EBM it builds a Payroll Batch Loader (PBL) file with a "Taxable Allowance" entry and, for Philippine workers, the non-taxable "De Minimis" allowance split into Rice, Christmas, Uniform and Laundry by fixed thresholds; it checks the file into UCM and submits the HCM payroll flow "Load and Transfer Batch" for the PH Legislative Data Group.

| Field | Value |
| --- | --- |
| Interface | ProcessPBLProvABCSImpl |
| Type | Listener |
| Complexity | medium |
| Components to build | 4 |
| Mapping rules | 15 |
| Business rules | 3 |

## Interface Summary

This service sets up a new employee's allowances in payroll. For every employee it receives, it creates a "Taxable Allowance" entry with the allowance amount. For employees in the Philippines it also takes the de minimis (tax-free) allowance and splits it across benefit types in a fixed order: Rice up to 1,500, then Christmas up to 400 more, then Uniform up to 400 more, and anything above 2,300 as Laundry. It writes these entries into a payroll batch file, uploads it to HCM and starts the payroll flow that loads and transfers the batch.  The payroll impact makes accuracy important, and there are defects to settle before rebuilding. The checks that should skip employees without an allowance are written so they are always true, so entries are created even when the amount is empty or zero. The employee number in the file is taken from the SSS number field. The thresholds and benefit names are fixed in the code. The service also never checks whether the payroll flow succeeded, and on error it emails the service desk but never answers the caller.  In Boomi this becomes a sub-process that builds the file in memory, uses the shared HCM upload and flow-submission components, and returns a real result.

**Complexity:** medium (score 42)

## Functional Analysis

1. **Receive employees** — Async SOAP process with EmployeeEBM EmployeeRequest (createInstance).
2. **Build PBL rows** — XForm\_EBO\_To\_DAT: Taxable Allowance row per employee; PH de minimis tier rows.
3. **Any rows?** — count(PBL) \> 0, otherwise nothing is done (in practice always true).
4. **Write PBL file** — File adapter writes PBLFile.dat with header line to the LogicalPath (PROD: /u01/data/domains/SOACS-Pr\_domain/osi\_payroll/put/).
5. **Encode file** — Java FiletoBase64Class: Base64 of PBLFile.dat (US-ASCII), then clean the folder; errors only audited.
6. **Check in to UCM** — GenericSoapOperation CHECKIN\_UNIVERSAL with Authorization header (Credentials.dvm).
7. **Submit payroll flow** — FlowActionsService.submitFlow "Load and Transfer Batch" with the UCM content ID; Authorization header.
8. **Callback** — processResponse with empty result.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessPBLProvABCSImpl/processpblprovabcsimpl\_client\_ep | none declared |
| outbound | File (SOA server) | /u01/data/domains/SOACS-Pr\_domain/osi\_payroll/put/PBLFile.dat | n/a |
| outbound | SOAP 1.1 over HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | HTTP Authorization header (Credentials.dvm) |
| outbound | SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com/hcmProcFlowCoreController/FlowActionsService | HTTP Authorization header (Credentials.dvm) |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Employee EBM to PBL element entries | TaskName / TaskActionName | (none) | hardcoded | — | "Element Entry" / "Create Element Entry" | low | All rows. |
| MR-002 | Employee EBM to PBL element entries | EffectiveStartDate | Employee/EmploymentAssignments/EffectiveStartDate | passthrough | — | — | low | — |
| MR-003 | Employee EBM to PBL element entries | PersonNumber | Employee/Person/NationalID\_Type/SSSNumber | passthrough | — | — | low | SSS number used as person number. |
| MR-004 | Employee EBM to PBL element entries | AssignmentNumber | Employee/Person/NationalID\_Type/SSSNumber | calculated | — | "ASSIGNMENT\_" + SSSNumber | low | — |
| MR-005 | Employee EBM to PBL element entries | PBL row: Reference "Taxable Allowance" / Amount | Employee/EmployeeAllowanceType/AllowanceType | conditional | AllowanceType != "" or != "0" (always true) | AllowanceType | medium | Every employee. |
| MR-006 | Employee EBM to PBL element entries | PBL row: Reference "De Minimis Rice" / Amount | Employee/EmployeeAllowanceType/Allowance | conditional | CountryCode = PH | Allowance if \<= 1500, else 1500 | medium | Guard on Allowance also always true. |
| MR-007 | Employee EBM to PBL element entries | PBL row: Reference "De Minimis Christmas" / Amount | Employee/EmployeeAllowanceType/Allowance | conditional | PH and Allowance \> 1500 | Allowance - 1500 if \<= 1900, else 400 | medium | — |
| MR-008 | Employee EBM to PBL element entries | PBL row: Reference "De Minimis Uniform" / Amount | Employee/EmployeeAllowanceType/Allowance | conditional | PH and Allowance \> 1900 | Allowance - 1900 if \<= 2300, else 400 | medium | — |
| MR-009 | Employee EBM to PBL element entries | PBL row: Reference "De Minimis Laundry" / Amount | Employee/EmployeeAllowanceType/Allowance | conditional | PH and Allowance \> 2300 | Allowance - 2300 | medium | No upper limit. |
| MR-010 | Employee EBM to PBL element entries | EffectiveEndDate, TermsNumber, RelationshipNumber, Payroll, Reason, Subpriority, PayValue, Periodicity, FullTimeEquivalent, OverrideEntry | (none) | hardcoded | — | empty | low | — |
| MR-011 | UCM response to payroll flow submission | flowName | (none) | hardcoded | — | "Load and Transfer Batch" | low | — |
| MR-012 | UCM response to payroll flow submission | parameterValues\[Batch\] | current dateTime | calculated | — | "PFB" + yyyyMMddHHmmss | low | Server time zone. |
| MR-013 | UCM response to payroll flow submission | parameterValues\[Content Id\] | GenericResponse Field\[@name="dDocName"\] | conditional | @name = dDocName | — | low | — |
| MR-014 | UCM response to payroll flow submission | flowInstanceName | current dateTime | calculated | — | "PF" + yyyyMMddHHmmss | low | Two calls in the same second collide. |
| MR-015 | UCM response to payroll flow submission | legislativeDataGroupName / recurringFlag | (none) | hardcoded | — | "PH Legislative Data Group" / false | low | Non-PH Taxable Allowance rows also go to the PH LDG. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Taxable allowance entry | Every employee gets a Taxable Allowance entry with AllowanceType as amount (intended only when non-empty and non-zero). | High |
| BR-002 | PH de minimis split | Rice up to 1500; Christmas up to 400 more; Uniform up to 400 more; remainder above 2300 as Laundry. | High |
| BR-003 | Payroll load flow | Load and Transfer Batch submitted for the PH Legislative Data Group with batch name PFB\<timestamp\>. | High |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive new-hire employees** — Employee EBM
2. **Build allowance rows** — Taxable + PH de minimis tiers
3. **Any rows?** — Always true in practice
4. **Write and encode file** — PBLFile.dat, Base64, folder cleaned
5. **Check in to UCM** — CHECKIN\_UNIVERSAL
6. **Submit payroll flow** — Load and Transfer Batch; not verified
7. **Callback** — Empty result
8. **Notify service desk** — No callback

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| HCM PBL Element Entry DAT Prof | flatfile | prof-pbl-dat |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback | Sub-process start + Return Documents |
| XForm\_EBO\_To\_DAT + File adapter | map-ebm-to-pbl + prof-pbl-dat (in memory) |
| Java FiletoBase64Class | Base64 encode step |
| CheckIN\_UCM | op-ucm-checkin (hcm-hdl-loader) |
| PBLSubmitFlowService submitFlow | map-ucm-to-pbl-submitflow + op-hcm-flow-submit (emp-campaign-extract-sch) |
| catchAll | Try/Catch + util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| prof-pbl-dat | PBL flat-file profile |
| map-ebm-to-pbl | Allowance calculation map |
| map-ucm-to-pbl-submitflow | submitFlow request map |
| proc-sub-hcm-pbl-allowances | Main sub-process |

## Client questions

- **Q-001** Are the Philippine de minimis ceilings still Rice 1,500, Christmas 400, Uniform 400 with the remainder as Laundry, and are these the correct element names in payroll?
  **If this is not answered:** Keep the current ceilings and names, held in a configurable table.
- **Q-002** Should employees with no allowance or a zero allowance be skipped, rather than given an empty or zero allowance entry?
  **If this is not answered:** Skip empty and zero amounts.
- **Q-003** Is the employee's SSS number always the same as their HCM person number and assignment number suffix?
  **If this is not answered:** Yes, as today.
- **Q-004** Should non-Philippine employees' taxable allowances also be loaded, given that every batch is submitted to the PH Legislative Data Group?
  **If this is not answered:** Limit the service to PH employees.
- **Q-005** Should the process wait for the payroll load to finish and report failures to payroll?
  **If this is not answered:** Poll the flow status and email failures to the IT service desk list.
