# ProcessUpdateExternalIdentifier

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

Daily job that gives new US hires their third-party payroll identifier in Oracle HCM. It selects US workers whose assignment starts today and is Active - Payroll Eligible, builds an HDL ExternalIdentifier.dat with type ORA\_3RD\_PARTY\_PAY\_ID whose number is a legal-employer/location prefix (MX4, 6PZ or AK0) plus the person number without its first character, loads it through UCM and HCM Data Loader, waits five minutes, reads the HDL errors and emails the result.

| Field | Value |
| --- | --- |
| Interface | ProcessUpdateExternalIdentifier |
| Type | Scheduled |
| Complexity | medium |
| Components to build | 3 |
| Mapping rules | 7 |
| Business rules | 2 |

## Interface Summary

Each day this job looks in HCM for new US hires whose assignment starts that day and gives each one a third-party payroll ID. The ID is built from a three-character code that depends on the legal employer and, for TaskUs USA LLC, on whether the location is Santa Monica, followed by the person number without its first character. The IDs are loaded into HCM with HCM Data Loader. After five minutes the job checks the load for errors and emails the outcome.  Only hires starting on the day the job runs are picked up, so a missed run means those hires never get the ID. Workers of any other legal employer get an empty ID. The result emails go to one hardcoded person in a format the notification service cannot read, so they are probably not delivered, and the five-minute wait is a guess at how long HCM needs.  In Boomi this becomes a scheduled process using the shared OTBI and HDL loading sub-processes, with the prefix rules held in a cross-reference table and the load status checked instead of a fixed wait.

**Complexity:** medium (score 38)

## Functional Analysis

1. **Scheduled start** — Direct binding; XXTS window read (InputDate = current date).
2. **Find today's US starters** — OTBI query (see key points).
3. **Any rows?** — count(Row) \> 0, else nothing.
4. **Build ExternalIdentifier.dat** — xform\_Int\_ExtIdDAT; File adapter writes Worker.dat.
5. **Zip and load** — Java ZIPtoBASE64 on /u01/data/domains/SOACS-Pr\_domain/ExternalIdentifier; UCM check-in; HDL importAndLoadData.
6. **Check errors** — Wait 5 minutes; getHDLErrors (ContentId = dDocName); first MSG\_TEXT.
7. **Notify and finish** — Error or success email; advance XXTS window and last run.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding / SOAP async | processupdateexternalidentifier\_direct\_ep / \_client\_ep | none declared |
| outbound | File (SOA server) | /u01/data/domains/SOACS-Pr\_domain/ExternalIdentifier/Worker.dat | n/a |
| outbound | SOAP over HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | HTTP Authorization header (Credentials.dvm) |
| outbound | SOAP over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCommonDataLoader/HCMDataLoader | HTTP Authorization header (Credentials.dvm) |
| outbound | SOAP (composites) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl, gethdlerrorsprovabcsimpl\_client\_ep.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | OTBI rowset to ExternalIdentifier DAT | METADATA / ExternalIdentifier / ExternalIdentifierSequence | (none) | hardcoded | — | "MERGE" / "ExternalIdentifier" / "1" | low | — |
| MR-002 | OTBI rowset to ExternalIdentifier DAT | PersonNumber | Column3 | passthrough | — | — | low | — |
| MR-003 | OTBI rowset to ExternalIdentifier DAT | ExternalIdentifierNumber | Column1 (legal employer), Column2 (location), Column3 | conditional | TaskUs USA LLC + Santa Monica / TaskUs USA LLC / TaskUs Inc. | "MX4" / "6PZ" / "AK0" + substring(PersonNumber, 2) | medium | Empty for other employers. |
| MR-004 | OTBI rowset to ExternalIdentifier DAT | ExternalIdentifierType | (none) | hardcoded | — | "ORA\_3RD\_PARTY\_PAY\_ID" | low | — |
| MR-005 | OTBI rowset to ExternalIdentifier DAT | AssignmentNumber | Column4 | passthrough | — | — | low | — |
| MR-006 | OTBI rowset to ExternalIdentifier DAT | DateFrom / DateTo | (current time) | calculated | — | now (yyyy/MM/dd HH:mm:ss) / "4712/12/31 11:59:00" | low | Not the assignment start date. |
| MR-007 | OTBI rowset to ExternalIdentifier DAT | SourceSystemId / SourceSystemOwner | Column4 | calculated | — | "EXT\_" + substring-after(Column4, "ASSIGNMENT\_") / "TASKUS" | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Selection | US legislation, Active - Payroll Eligible, assignment starting today. | High |
| BR-002 | Payroll ID prefix | Prefix by legal employer and Santa Monica location. | High |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Scheduled start** — XXTS direct binding
2. **Find today's US starters** — OTBI
3. **Any starters?**
4. **Build ExternalIdentifier DAT** — Prefix rules
5. **Load into HCM** — UCM + HDL
6. **Wait 5 min, get errors** — hcm-get-hdl-errors
7. **Notify fault** — catchAll
8. **Errors?**
9. **Email error** — Hardcoded recipient
10. **Email success**
11. **Advance window, end**

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| HCM HDL ExternalIdentifier DAT Prof | flatfile | prof-hdl-extid-dat |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| XXTS window | Schedule-window sub-process (newhire-emp-sch) or Boomi schedule |
| ProcessHCMExtractSyncProv | Process Call hcm-otbi-sql-query |
| xform\_Int\_ExtIdDAT | map-rowset-to-extid-dat |
| File + Java + UCM + HDL | Process Call hcm-hdl-loader |
| Wait 5 min + getHDLErrors | Load status polling + hcm-get-hdl-errors |

**Components**

| Component | Role |
| --- | --- |
| prof-hdl-extid-dat | DAT profile |
| map-rowset-to-extid-dat | DAT map |
| proc-hcm-external-identifier-sch | Scheduled process |

## Client questions

- **Q-001** Is the US third-party payroll ID job running in production today?
  **If this is not answered:** It is live and will be rebuilt.
- **Q-004** When should the US third-party payroll ID job run each day?
  **If this is not answered:** Daily at 23:00 UTC, picking up all starters since the last successful run.
- **Q-002** Are the payroll ID prefixes still MX4 (TaskUs USA LLC, Santa Monica), 6PZ (other TaskUs USA LLC) and AK0 (TaskUs Inc.), and what should happen for legal employers that match no prefix rule?
  **If this is not answered:** Keep the three rules in a maintainable table owned by US payroll.
- **Q-003** Who should receive the success and error notices of this job?
  **If this is not answered:** The US HRIS distribution list.
