# ProcessDatProfileGeneration (archive file poll)

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

Second entry point of the DAT profile generation composite: a File adapter polls the Interviewed archive folder every 10 minutes for archived new-hire files (names matching .\*\\.csv\_.\*) at least two hours old, deletes them after reading, and runs the same post-load core as dat-profile-generation-rest (person-number check, Jobvite status, HR lists, consolidated rows, salary/talent DAT, HDL and PBL loads) on the whole file.

| Field | Value |
| --- | --- |
| Interface | ProcessDatProfileGeneration (archive file poll) |
| Type | Scheduled |
| Complexity | simple |
| Components to build | 2 |
| Mapping rules | 1 |
| Business rules | 1 |

## Interface Summary

After the new-hire file intake has processed a file from the recruiting system, the file is moved to an archive folder. Two hours later this poll picks the archived file up again and runs the follow-up work for every hire in it: checking that HCM created the person, updating Jobvite with success or the error, emailing HR lists, adding the hires to the nightly summary, and loading salary, talent profile and allowances. The archived file is then deleted, so no archive copy remains.  The two-hour delay gives HCM time to finish loading the workers. In the production configuration plan the folder setting for this poll is blank (the composite itself names InterviewedArchive), so whether the poll is actually active in production cannot be confirmed from the export. In Boomi this becomes a scheduled process that reads the same files (or, better, is triggered once the worker loads are confirmed) and calls the shared core sub-process.

**Complexity:** simple (score 14)

## Functional Analysis

1. **Poll archive folder** — Every 10 minutes; files .\*\\.csv\_.\* older than 2 hours; delete after read.
2. **Parse file** — ReadFileService NXSD (INT new-hire CSV).
3. **Run post-load core** — Same loop and batch steps as dat-profile-generation-rest.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | File (SOA server) | /u01/data/domains/SOACS-Pr\_domain/InterviewedArchive/.\*\\.csv\_.\* | n/a |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Archived INT CSV to Employees | Employees/Employee\[\] | CSV rows | passthrough | — | NXSD parse; same element names | low | No XSLT; the BPEL uses the parsed file directly. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Settle time | Files are only processed when at least two hours old. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Poll archive every 10 min** — Files older than 2 h
2. **Parse INT file** — Delete after read
3. **Run post-load core** — dat-profile-generation-rest core
4. **Done**

## Data Model

_No profile components owned here; it uses the INT CSV profile of int-file-intake._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| ReadINTFile File adapter | op-int-archive-get on conn-int-file-share |
| BPEL core | Process Call proc-sub-dat-profile-core |

**Components**

| Component | Role |
| --- | --- |
| op-int-archive-get | Get archived CSV files |
| proc-dat-profile-archive-poll | Scheduled process |

## Client questions

- **Q-001** Do hires that arrive by file currently receive the Jobvite status update and the salary and allowance loads?
  **If this is not answered:** Yes; build the follow-up as a step that runs once the worker load is confirmed.
- **Q-002** How long should processed new-hire files be kept, given that they contain personal data?
  **If this is not answered:** Keep 30 days in a restricted archive, then delete.
