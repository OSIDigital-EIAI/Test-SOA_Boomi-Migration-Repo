# Migration Plan — XXSendNotification

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.683Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | XXSendNotification |
| Identifier | util-send-notification |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | simple (12) |
| Components to build | 5 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/BPEL/XXSendNotification.bpel |
| Depends on | Nothing |

## 2. End-to-End Process Architecture

A standalone SCA composite exposing an asynchronous one-way SOAP service with a callback. Its single BPEL component holds a Java embedding that sends mail through JavaMail to Gmail SMTP with a hardcoded account. Callers bind to it through a client WSDL in MDS (oramds:/apps/Utilities/Notification/xxsendnotification_client_ep.wsdl) whose concrete address is the SOA host IP. There is no configuration plan.

A shared Boomi sub-process "Sub: Send Notification" in the universal shared folder, invoked by Process Call (preferably via Process Route, per the OSI best-practice document, so changes do not require redeploying callers). Inputs are set as dynamic process properties (DPP_EmailID, DPP_EmailSubject, content). A Groovy script builds the HTML table, and the Mail connector sends it using a secured SMTP connection. A Web Services Server listener wrapper is only needed if a non-Boomi system still calls the service.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| SOAP service xxsendnotification_client_ep (async one-way + callback) | Sub-process start (Data Passthrough) called by Process Call / Process Route | Callers are migrated to Boomi, so no web listener is required unless an external caller remains. |
| Java embedding SendMail_Java_Embedding + com.osi.SendMail | Mail connector Send operation (op-mail-send-html) on conn-smtp-mail | Credentials move to the connection. |
| com.osi.FormatEmailContent | Groovy transform.function fn-format-email-html | Add guards for missing separators. |
| XXSendNotification.xsd process element | prof-notification-req-xml |  |
| Audit trail entries | Boomi process logging / tracked fields | Do not log full content where it contains personal data. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive notification request | One-way SOAP operation "process" receives Email (comma-separated), Subject and Content. The audit trail records Email, Subject and Content. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/BPEL/XXSendNotification.bpel |
| 2 | Read request fields | Java embedding reads Email, Subject and Content with getVariableData. Only one Content value is read even though the schema allows several. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/BPEL/XXSendNotification.bpel |
| 3 | Format body as HTML table | FormatEmailContent splits Content on "@!-": even positions become <th> headers, odd positions become cell values; a value containing "!/" is split across several table rows; a fixed CSS style block draws borders. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/FormatEmailContent.java |
| 4 | Send email via SMTP | SendMail opens an authenticated session to smtp.gmail.com:587 with STARTTLS, splits recipients on commas (all as To), sets X-Priority 1, sets the subject and the HTML body, and sends. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/SendMail.java |
| 5 | Log result | Writes "success==> Success" or "error==> <exception>" to the audit trail. Exceptions never propagate. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/BPEL/XXSendNotification.bpel |
| 6 | Acknowledge caller | Invokes the processResponse callback with an empty result, whether or not the mail was sent. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/BPEL/XXSendNotification.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| Notification Request XML Prof | xml | process/Email | string | yes |
| Notification Request XML Prof | xml | process/Subject | string | yes |
| Notification Request XML Prof | xml | process/Content | string | no |
| Notification Request XML Prof | xml | processResponse/result | string | yes |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | To recipients | /process/Email | transformed |  | split on "," and trim each address | low | All recipients are To; no CC/BCC. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/SendMail.java | Notification request to email message |
| MR-002 | Subject | /process/Subject | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/SendMail.java | Notification request to email message |
| MR-003 | Body (text/html) | /process/Content[1] | calculated | Content must contain at least one "@!-" pair | FormatEmailContent.formatContent: labels at even "@!-" positions -> <th>, values at odd positions -> <td>; "!/" inside a value -> additional rows; fixed bordered-table CSS | medium | Only the first Content element is used. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/FormatEmailContent.java | Notification request to email message |
| MR-004 | From | (none) | hardcoded |  | svc.soa@taskus.com | low | Becomes the Mail connection user / from address. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/BPEL/XXSendNotification.bpel | Notification request to email message |
| MR-005 | Header X-Priority | (none) | hardcoded |  | "1" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/SendMail.java | Notification request to email message |
| MR-006 | processResponse/result | (none) | hardcoded |  | left empty | low | The callback carries no result. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/BPEL/XXSendNotification.bpel | Notification request to email message |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Label/value body convention | Content is a sequence "Label@!-Value@!-Label@!-Value..."; labels become table headings, values become the cells beneath; a value containing "!/" is split into several rows, other columns are padded with empty cells. | High | Groovy script fn-format-email-html called before the Mail Send; keep the same convention so callers need no change. |
| BR-002 | Multiple recipients | Email may contain several addresses separated by commas; all are sent as To recipients. | Medium | Pass the list to the Mail connector To field (comma-separated). |
| BR-003 | High priority | Every message is flagged X-Priority 1. | Low | Set the header on the mail message if the Mail connector supports custom headers; otherwise confirm it can be dropped. |
| BR-004 | Never fail the caller | Mail failures are logged only; the caller always receives a response. | Medium | Try/Catch around the Send; decide whether to return a failure flag or log only (see questions). |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 (async, WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/XXSendNotification/xxsendnotification_client_ep (PROD MDS client WSDL) | none declared (no OWSM policy in composite.xml) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/Notification/xxsendnotification_client_ep.wsdl |
| inbound SOAP 1.1 | http://129.144.156.126:9073/soa-infra/services/default/XXSendNotification/xxsendnotification_client_ep (DEV MDS client WSDL) | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/DEV_MDS/apps/Utilities/Notification/xxsendnotification_client_ep.wsdl |
| outbound SMTP with STARTTLS | smtp.gmail.com:587 | SMTP AUTH username/password (account svc.soa@taskus.com; password hardcoded in BPEL) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/SendMail.java |
| Java embedding (bpelx:exec) + JavaMail | mail.smtp.auth=true, mail.smtp.starttls.enable=true, mail.smtp.ssl.trust=smtp.gmail.com, mail.smtp.host=smtp.gmail.com, mail.smtp.port=587, mail.debug=true; From svc.soa@taskus.com; all recipients as To; header X-Priority 1; body text/html. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/SendMail.java |

## 8. Sub-Process Call Hierarchy

> **Not available.** This interface calls no sub-processes.

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/BPEL/XXSendNotification.bpel | Embedding SendMail_Java_Embedding: reads request fields, holds the sender account and password literal, calls SendMail, audits the result. | Set Properties from the request profile + Mail connector Send operation; credentials on the Mail connection. | Medium |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/SendMail.java | JavaMail session, recipient parsing, high-priority HTML message send. | Mail connector (SMTP) with Send operation; To from DPP_EmailID; custom header X-Priority via message properties if supported. | Low |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/FormatEmailContent.java | Converts the "@!-" / "!/" convention into an HTML table. | Groovy data-process script fn-format-email-html (port the algorithm, guard the empty and odd-count cases, HTML-escape values). | Medium |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Try/catch inside the Java embedding only; errors are written to the audit trail and the callback is always sent with an empty result. No BPEL fault handler. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Utilities/Notification/xxsendnotification_client_ep.wsdl | mds | yes |
| EmailDL.dvm (used by callers to choose recipients) | dvm | yes |
| java-mail-1.4.4.jar, javax.activation.jar, common-lang3.jar | library | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Gmail SMTP | SMTP/STARTTLS | smtp.gmail.com:587 | Google (TaskUs Google Workspace account) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | XXSendNotification endpoint | http://129.144.156.28:[redacted] | [redacted].wsdl |
| DEV | XXSendNotification endpoint | http://129.144.156.126:[redacted] | [redacted].wsdl |

## 14. Boomi Migration Readiness Summary

This interface sends formatted email notifications on behalf of every other TaskUs integration. A calling process passes three things: who to send to (a comma-separated list), a subject line, and a body written as label/value pairs separated by the characters "@!-". The utility turns those pairs into a simple HTML table, with labels as column headings and values underneath, and sends the message as high priority through a Gmail SMTP account.

The logic is small, but the utility is used by almost every integration, so it is one of the first things to build in Boomi. In Boomi it becomes a shared sub-process with a mail connection, rather than a separately deployed web service. Callers will hand it the recipients, subject and content as process properties.

Two behaviours need attention. The mail account password is written directly in the process code and must be replaced by a properly secured connection (and rotated). And if mail sending fails, the failure is only written to the server log: the caller is still told the request was handled, so failed notifications go unnoticed today.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Notification Request XML Prof | prof-notification-req-xml | profile.xml | — |
| 2 | TaskUs SMTP Mail Conn | conn-smtp-mail | connector-settings | — |
| 3 | Mail Notification SEND Op | op-mail-send-html | connector-action | conn-smtp-mail |
| 4 | Format Email HTML Script | fn-format-email-html | transform.function | — |
| 5 | Sub: Send Notification | proc-sub-send-notification | process | op-mail-send-html, prof-notification-req-xml, fn-format-email-html |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/BPEL/XXSendNotification.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/Schemas/XXSendNotification.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/WSDLs/XXSendNotification.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/SendMail.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/FormatEmailContent.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/Notification/xxsendnotification_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/DEV_MDS/apps/Utilities/Notification/xxsendnotification_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm

## Open questions

### Still outstanding

REVIEW-NEEDED: Which mail service and sending account should the migrated notifications be sent from? (The current Gmail account is configured inside the code; the Boomi mail connection cannot be set up without a target relay and account.)
  - If this is not answered: Keep sending through Gmail SMTP (smtp.gmail.com:587) from svc.soa@taskus.com with a newly issued credential stored only on the Boomi connection.

REVIEW-NEEDED: If a notification email cannot be sent, should the calling integration be told it failed, or should the failure only be logged as today? (Today failures are silently logged, so alerts can be lost; changing this affects how every caller handles errors.)
  - If this is not answered: Log the failure against the execution and return a failure flag to the caller without stopping the caller's processing.

REVIEW-NEEDED: Should notifications keep being sent as high-priority emails? (Every message is flagged high priority today; recipients may prefer normal priority for informational notices.)
  - If this is not answered: Keep high priority on all notifications.

REVIEW-NEEDED: Is any system outside the integrations being migrated (for example a monitoring tool or script) calling the notification service directly? (Decides whether a web service listener must be published in Boomi or an internal sub-process is enough.)
  - If this is not answered: No external callers; built as an internal sub-process only.

REVIEW-NEEDED: Roughly how many integration notification emails are sent on a normal day and on the busiest day? (Needed to check the mail relay's sending limits.)
  - If this is not answered: Assume fewer than 500 per day, within standard relay limits.

