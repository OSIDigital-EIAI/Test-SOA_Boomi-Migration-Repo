# XXSendNotification

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

XXSendNotification is the shared email utility of the TaskUs integration estate. Any integration that needs to tell people something (an error, a completed provisioning step, a terminated employee) sends it a recipient list, a subject and a body; the utility formats the body as an HTML table and sends it through an SMTP mail account.

| Field | Value |
| --- | --- |
| Interface | XXSendNotification |
| Type | Listener |
| Complexity | simple |
| Components to build | 5 |
| Mapping rules | 6 |
| Business rules | 4 |

## Interface Summary

This interface sends formatted email notifications on behalf of every other TaskUs integration. A calling process passes three things: who to send to (a comma-separated list), a subject line, and a body written as label/value pairs separated by the characters "@!-". The utility turns those pairs into a simple HTML table, with labels as column headings and values underneath, and sends the message as high priority through a Gmail SMTP account.  The logic is small, but the utility is used by almost every integration, so it is one of the first things to build in Boomi. In Boomi it becomes a shared sub-process with a mail connection, rather than a separately deployed web service. Callers will hand it the recipients, subject and content as process properties.  Two behaviours need attention. The mail account password is written directly in the process code and must be replaced by a properly secured connection (and rotated). And if mail sending fails, the failure is only written to the server log: the caller is still told the request was handled, so failed notifications go unnoticed today.

**Complexity:** simple (score 12)

## Functional Analysis

1. **Receive notification request** — One-way SOAP operation "process" receives Email (comma-separated), Subject and Content. The audit trail records Email, Subject and Content.
2. **Read request fields** — Java embedding reads Email, Subject and Content with getVariableData. Only one Content value is read even though the schema allows several.
3. **Format body as HTML table** — FormatEmailContent splits Content on "@!-": even positions become \<th\> headers, odd positions become cell values; a value containing "!/" is split across several table rows; a fixed CSS style block draws borders.
4. **Send email via SMTP** — SendMail opens an authenticated session to smtp.gmail.com:587 with STARTTLS, splits recipients on commas (all as To), sets X-Priority 1, sets the subject and the HTML body, and sends.
5. **Log result** — Writes "success==\> Success" or "error==\> \<exception\>" to the audit trail. Exceptions never propagate.
6. **Acknowledge caller** — Invokes the processResponse callback with an empty result, whether or not the mail was sent.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 (async, WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/XXSendNotification/xxsendnotification\_client\_ep (PROD MDS client WSDL) | none declared (no OWSM policy in composite.xml) |
| inbound | SOAP 1.1 | http://129.144.156.126:9073/soa-infra/services/default/XXSendNotification/xxsendnotification\_client\_ep (DEV MDS client WSDL) | none declared |
| outbound | SMTP with STARTTLS | smtp.gmail.com:587 | SMTP AUTH username/password (account svc.soa@taskus.com; password hardcoded in BPEL) |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Notification request to email message | To recipients | /process/Email | transformed | — | split on "," and trim each address | low | All recipients are To; no CC/BCC. |
| MR-002 | Notification request to email message | Subject | /process/Subject | passthrough | — | — | low | — |
| MR-003 | Notification request to email message | Body (text/html) | /process/Content\[1\] | calculated | Content must contain at least one "@!-" pair | FormatEmailContent.formatContent: labels at even "@!-" positions -\> \<th\>, values at odd positions -\> \<td\>; "!/" inside a value -\> additional rows; fixed bordered-table CSS | medium | Only the first Content element is used. |
| MR-004 | Notification request to email message | From | (none) | hardcoded | — | svc.soa@taskus.com | low | Becomes the Mail connection user / from address. |
| MR-005 | Notification request to email message | Header X-Priority | (none) | hardcoded | — | "1" | low | — |
| MR-006 | Notification request to email message | processResponse/result | (none) | hardcoded | — | left empty | low | The callback carries no result. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Label/value body convention | Content is a sequence "Label@!-Value@!-Label@!-Value..."; labels become table headings, values become the cells beneath; a value containing "!/" is split into several rows, other columns are padded with empty cells. | High |
| BR-002 | Multiple recipients | Email may contain several addresses separated by commas; all are sent as To recipients. | Medium |
| BR-003 | High priority | Every message is flagged X-Priority 1. | Low |
| BR-004 | Never fail the caller | Mail failures are logged only; the caller always receives a response. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive notification request** — SOAP one-way process(Email, Subject, Content)
2. **Read fields, audit log** — Email, Subject and Content written to audit trail
3. **Format content as HTML** — FormatEmailContent: @!- pairs to table, !/ to rows
4. **Send via Gmail SMTP** — smtp.gmail.com:587 STARTTLS, X-Priority 1
5. **Log error only** — Exception caught in Java and audited; not raised
6. **Callback empty result** — processResponse sent in all cases

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| Notification Request XML Prof | xml | prof-notification-req-xml |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| SOAP service xxsendnotification\_client\_ep (async one-way + callback) | Sub-process start (Data Passthrough) called by Process Call / Process Route |
| Java embedding SendMail\_Java\_Embedding + com.osi.SendMail | Mail connector Send operation (op-mail-send-html) on conn-smtp-mail |
| com.osi.FormatEmailContent | Groovy transform.function fn-format-email-html |
| XXSendNotification.xsd process element | prof-notification-req-xml |
| Audit trail entries | Boomi process logging / tracked fields |

**Components**

| Component | Role |
| --- | --- |
| conn-smtp-mail | SMTP mail connection |
| op-mail-send-html | Mail Send operation, HTML body |
| prof-notification-req-xml | Request profile (Email, Subject, Content) |
| fn-format-email-html | Groovy port of FormatEmailContent |
| proc-sub-send-notification | Shared sub-process |

## Client questions

- **Q-001** Which mail service and sending account should the migrated notifications be sent from?
  **If this is not answered:** Keep sending through Gmail SMTP (smtp.gmail.com:587) from svc.soa@taskus.com with a newly issued credential stored only on the Boomi connection.
- **Q-002** If a notification email cannot be sent, should the calling integration be told it failed, or should the failure only be logged as today?
  **If this is not answered:** Log the failure against the execution and return a failure flag to the caller without stopping the caller's processing.
- **Q-003** Should notifications keep being sent as high-priority emails?
  **If this is not answered:** Keep high priority on all notifications.
- **Q-004** Is any system outside the integrations being migrated (for example a monitoring tool or script) calling the notification service directly?
  **If this is not answered:** No external callers; built as an internal sub-process only.
- **Q-005** Roughly how many integration notification emails are sent on a normal day and on the busiest day?
  **If this is not answered:** Assume fewer than 500 per day, within standard relay limits.
