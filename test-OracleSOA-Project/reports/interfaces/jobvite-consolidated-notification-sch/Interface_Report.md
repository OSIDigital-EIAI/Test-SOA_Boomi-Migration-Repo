# ProcessJobviteConsolidatedNotification\_Sch

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

Nightly summary of the day's Jobvite-to-HCM hires. It reads the consolidated files JobviteConsolidate\_PH.csv and JobviteConsolidate\_US.csv accumulated by dat-profile-generation-rest (deleting them as it reads), turns each into a numbered NewHire\_\<country\>\_\<date\>.csv report (Person Number, Candidate ID, names, Status, Message) and emails PH and US reports to the HCM list recipients.

| Field | Value |
| --- | --- |
| Interface | ProcessJobviteConsolidatedNotification\_Sch |
| Type | Scheduled |
| Complexity | simple |
| Components to build | 3 |
| Mapping rules | 3 |
| Business rules | 2 |

## Interface Summary

Every night at 23:30 this job collects the day's results of the Jobvite hiring flow. During the day, each hire processed by the follow-up service adds a line (person number, candidate ID, name, success or failure and the message) to a Philippine or US file on the integration server. This job reads both files, numbers the lines, saves them as NewHire\_PH\_\<date\>.csv and NewHire\_US\_\<date\>.csv, and emails them to the PH and US HCM list recipients. The day files are deleted as they are read, so the next day starts empty.  If a country had no hires its file does not exist; the read error is silently ignored, but the email step still tries to attach a report that was never written, which most likely makes the job fail at that point without telling anyone. Because rows are appended to a file that has a header line, the day file may also contain repeated header lines. The job does not use the scheduler window; it only records its last run time.  In Boomi this becomes a scheduled process that reads the day's rows from the staging store agreed for dat-profile-generation-rest, builds the two reports and sends them through the file-notification sub-process, sending a "no hires today" notice or nothing when a country has no rows.

**Complexity:** simple (score 16)

## Functional Analysis

1. **Scheduled start** — Direct binding from the scheduler (or SOAP client).
2. **Read PH day file** — Sync read JobviteConsolidate\_PH.csv and delete; errors ignored.
3. **Write PH report** — xfrom\_PH\_consolidated -\> NewHire\_PH\_\<date\>.csv.
4. **Read US day file** — Same for JobviteConsolidate\_US.csv.
5. **Write US report** — xform\_US\_consolidated -\> NewHire\_US\_\<date\>.csv.
6. **Email reports** — FileNotification PH to HCMListEmail, US to USHCMListEmail.
7. **Record run** — Update\_XXTS\_Schedule last\_scheduled\_time (direct path); callback.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding / SOAP async | processjobviteconsolidatednotification\_direct\_ep / \_client\_ep | none declared |
| outbound | File (SOA server) | /u01/data/domains/SOACS-Pr\_domain/osi\_file/EmailAttachment/JobviteConsolidate\_PH.csv, \_US.csv (read+delete); NewHire\_PH/US\_\<date\>.csv (write) | n/a |
| outbound | SOAP (composite) | oramds:/apps/Utilities/Notification/filenotification\_client\_ep.wsdl | none declared |
| outbound | DB (SOA schema) | XXTS\_SCHEDULING | data source |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Consolidated rows to daily summary | Sno | (row position) | calculated | — | position() | low | — |
| MR-002 | Consolidated rows to daily summary | EmployeeId, CandidateId, FirstName, MiddleName, LastName, Status, Message | same fields | looped | each row | — | low | — |
| MR-003 | Consolidated rows to daily summary | Header line | (none) | hardcoded | — | "SL\_No, Person Number, Candidate ID, First Name, Middle Name, Last Name, Status, Message" | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Country split | PH report to HCMListEmail, US report to USHCMListEmail. | Medium |
| BR-002 | Daily reset | Day files are deleted when read. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Scheduled start 23:30** — Direct binding
2. **Read PH day rows** — Delete after read
3. **Write PH report** — NewHire\_PH\_\<date\>.csv
4. **Read US day rows**
5. **Write US report** — NewHire\_US\_\<date\>.csv
6. **Email PH and US reports** — util-file-notification
7. **Record last run** — XXTS last\_scheduled\_time
8. **Unhandled fault** — e.g. missing report

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| NewHire Daily Summary CSV Prof | flatfile | prof-newhire-summary-csv |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| XXTS scheduler direct binding | Boomi schedule 23:30 UTC |
| SyncFileRead of day files | Read from staging store (decision in dat-profile-generation-rest) |
| xfrom\_PH/US\_consolidated + WriteNotifications | map-consolidated-to-summary into prof-newhire-summary-csv (in memory) |
| FileNotification | Process Call util-file-notification |

**Components**

| Component | Role |
| --- | --- |
| prof-newhire-summary-csv | Summary report profile |
| map-consolidated-to-summary | Summary map |
| proc-jobvite-consolidated-notification-sch | Scheduled process |

## Client questions

- **Q-001** On a day with no hires for a country, should that country's team receive a "no hires today" email or nothing?
  **If this is not answered:** Send nothing for that country.
- **Q-002** Is 23:30 UTC still the right time for the daily Jobvite hire summary?
  **If this is not answered:** 23:30 UTC daily.
