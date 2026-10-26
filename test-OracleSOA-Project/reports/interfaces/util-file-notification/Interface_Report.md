# FileNotification

Test-OracleSOA-Project · Listener · simple complexity

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

FileNotification emails a report file to a list of people. A calling integration writes a CSV report to a shared folder on the SOA server and passes the recipients, a subject and the file path; this utility attaches the file to an email, sends it, and deletes the file.

| Field | Value |
| --- | --- |
| Interface | FileNotification |
| Type | Listener |
| Complexity | simple |
| Components to build | 2 |
| Mapping rules | 5 |
| Business rules | 2 |

## Interface Summary

This utility sends a report file as an email attachment. Other integrations build CSV reports during their runs and save them in a folder on the integration server; at the end of the run they ask this utility to email the file to the right distribution list. After a successful send the file is deleted from the server.  In Boomi this becomes a shared sub-process that receives the report itself as the document being processed, instead of a file path on a server, and sends it with the same mail connection used for ordinary notifications. That removes the dependency on a shared server folder.  The current utility sends an email with an empty body and only the attachment. Like the notification utility, it holds the mail account password in its code, and it never tells the caller if sending failed.

**Complexity:** simple (score 10)

## Functional Analysis

1. **Receive file notification request** — One-way SOAP operation "process" with Email, Subject and optional FilePath. Email, Subject and FilePath are written to the audit trail.
2. **Build email with attachment** — SendMailWithAttachment creates a multipart message: an empty text part plus the file at FilePath as an attachment named after the file.
3. **Send via SMTP** — Authenticated STARTTLS session to smtp.gmail.com:587 as svc.soa@taskus.com; all comma-separated recipients as To.
4. **Delete the file** — If the send returned "Success", the file at FilePath is deleted from the server.
5. **Acknowledge caller** — Callback processResponse with an empty result in all cases.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 (async, WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/FileNotification/filenotification\_client\_ep (PROD MDS client WSDL) | none declared |
| inbound | SOAP 1.1 | http://129.144.156.126:9073/soa-infra/services/default/FileNotification/filenotification\_client\_ep (DEV MDS client WSDL) | none declared |
| outbound | SMTP with STARTTLS | smtp.gmail.com:587 | SMTP AUTH username/password (svc.soa@taskus.com; password hardcoded in BPEL) |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | File notification request to email with attachment | To recipients | /process/Email | transformed | — | split on "," and trim | low | All as To. |
| MR-002 | File notification request to email with attachment | Subject | /process/Subject | passthrough | — | — | low | — |
| MR-003 | File notification request to email with attachment | Attachment | /process/FilePath | transformed | FilePath present and file readable | FileDataSource(FilePath); attachment name = file name | low | In Boomi the document itself is the attachment and DDP\_FileName carries the name. |
| MR-004 | File notification request to email with attachment | Body text | (none) | hardcoded | — | empty string | low | — |
| MR-005 | File notification request to email with attachment | From | (none) | hardcoded | — | svc.soa@taskus.com | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Delete report after sending | When the email is sent successfully, the report file is deleted from the server; on failure it is kept. | Medium |
| BR-002 | Attachment-only email | The email has an empty body; all information is in the subject and attachment. | Low |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive file notification** — SOAP one-way process(Email, Subject, FilePath)
2. **Attach file, send email** — Gmail SMTP 587, empty body, file attached
3. **Delete sent file** — Only when send returned Success
4. **Log error only** — Exception caught and audited; file kept
5. **Callback empty result** — processResponse always sent

## Data Model

_This interface has no profile component of its own in the inventory: the request is three scalar values passed as process properties, and the attachment content is whatever CSV the caller produces (its layout belongs to the calling interface). Nothing to model here._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| SOAP service filenotification\_client\_ep | Sub-process start (Data Passthrough) via Process Call / Process Route |
| Java SendMailWithAttachment + FileDataSource | op-mail-send-attachment (Mail connector Send with document as attachment) |
| File written by caller to /u01/.../osi\_file/EmailAttachment | In-memory document passed by the caller |

**Components**

| Component | Role |
| --- | --- |
| op-mail-send-attachment | Mail Send operation with attachment |
| proc-sub-file-notification | Shared sub-process |

## Client questions

- **Q-001** Should report emails keep an empty body, or include a short standard message describing the attached report?
  **If this is not answered:** Keep an empty body, as today; the subject and attachment carry the information.
- **Q-002** If a report email cannot be sent, who should be told, and should the report be kept for re-sending?
  **If this is not answered:** Log the failure, send an error notification to the "notification" distribution list, and do not retain the report.
- **Q-003** Are there any limits on attachment size or on sending files containing personal data by email that the migrated reports must respect?
  **If this is not answered:** Attachments are sent as today, without passwords (per the separate password-handling decision), with no size limit applied.
