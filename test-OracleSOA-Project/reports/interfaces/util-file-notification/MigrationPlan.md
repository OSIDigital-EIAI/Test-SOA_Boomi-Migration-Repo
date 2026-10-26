# Migration Plan — FileNotification

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.702Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | FileNotification |
| Identifier | util-file-notification |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | simple (10) |
| Components to build | 2 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/BPEL/FileNotification.bpel |
| Depends on | util-send-notification |

## 2. End-to-End Process Architecture

Standalone SCA composite with an asynchronous one-way SOAP service and a single BPEL component whose Java embedding sends the file as an attachment through JavaMail to Gmail SMTP and deletes it. Callers write the report with the File adapter to the shared EmailAttachment folder (Domain.dvm "Email" path) and then call this service through the MDS client WSDL.

Shared Boomi sub-process "Sub: Send File Notification" receiving the report as its document (from a Combine/Map step in the caller) together with DPP_EmailID, DPP_EmailSubject and DDP_FileName, and sending it with a Mail connector operation on the shared SMTP connection (conn-smtp-mail, owned by util-send-notification). No file system staging.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| SOAP service filenotification_client_ep | Sub-process start (Data Passthrough) via Process Call / Process Route | No external listener needed unless a non-Boomi caller exists. |
| Java SendMailWithAttachment + FileDataSource | op-mail-send-attachment (Mail connector Send with document as attachment) |  |
| File written by caller to /u01/.../osi_file/EmailAttachment | In-memory document passed by the caller | Removes shared-folder dependency and deletion step. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive file notification request | One-way SOAP operation "process" with Email, Subject and optional FilePath. Email, Subject and FilePath are written to the audit trail. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/BPEL/FileNotification.bpel |
| 2 | Build email with attachment | SendMailWithAttachment creates a multipart message: an empty text part plus the file at FilePath as an attachment named after the file. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/SCA-INF/src/com/osi/SendMailWithAttachment.java |
| 3 | Send via SMTP | Authenticated STARTTLS session to smtp.gmail.com:587 as svc.soa@taskus.com; all comma-separated recipients as To. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/SCA-INF/src/com/osi/SendMailWithAttachment.java |
| 4 | Delete the file | If the send returned "Success", the file at FilePath is deleted from the server. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/BPEL/FileNotification.bpel |
| 5 | Acknowledge caller | Callback processResponse with an empty result in all cases. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/BPEL/FileNotification.bpel |

## 4. Payload Profiles

> **Not available.** This interface has no profile component of its own in the inventory: the request is three scalar values passed as process properties, and the attachment content is whatever CSV the caller produces (its layout belongs to the calling interface). Nothing to model here.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | To recipients | /process/Email | transformed |  | split on "," and trim | low | All as To. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/SCA-INF/src/com/osi/SendMailWithAttachment.java | File notification request to email with attachment |
| MR-002 | Subject | /process/Subject | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/SCA-INF/src/com/osi/SendMailWithAttachment.java | File notification request to email with attachment |
| MR-003 | Attachment | /process/FilePath | transformed | FilePath present and file readable | FileDataSource(FilePath); attachment name = file name | low | In Boomi the document itself is the attachment and DDP_FileName carries the name. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/SCA-INF/src/com/osi/SendMailWithAttachment.java | File notification request to email with attachment |
| MR-004 | Body text | (none) | hardcoded |  | empty string | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/SCA-INF/src/com/osi/SendMailWithAttachment.java | File notification request to email with attachment |
| MR-005 | From | (none) | hardcoded |  | svc.soa@taskus.com | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/BPEL/FileNotification.bpel | File notification request to email with attachment |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Delete report after sending | When the email is sent successfully, the report file is deleted from the server; on failure it is kept. | Medium | Not needed: the report is never written to disk in Boomi; if a Disk staging step is kept, delete after send. |
| BR-002 | Attachment-only email | The email has an empty body; all information is in the subject and attachment. | Low | Mail Send with an empty or short standard body (see questions). |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 (async, WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/FileNotification/filenotification_client_ep (PROD MDS client WSDL) | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/Notification/filenotification_client_ep.wsdl |
| inbound SOAP 1.1 | http://129.144.156.126:9073/soa-infra/services/default/FileNotification/filenotification_client_ep (DEV MDS client WSDL) | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/DEV_MDS/apps/Utilities/Notification/filenotification_client_ep.wsdl |
| outbound SMTP with STARTTLS | smtp.gmail.com:587 | SMTP AUTH username/password (svc.soa@taskus.com; password hardcoded in BPEL) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/SCA-INF/src/com/osi/SendMailWithAttachment.java |
| Java embedding (bpelx:exec) + JavaMail | mail.smtp.auth=true, starttls.enable=true, ssl.trust=smtp.gmail.com, host smtp.gmail.com, port 587, mail.debug=true; multipart message: empty text part + FileDataSource(FilePath) attachment; file deleted after success. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/SCA-INF/src/com/osi/SendMailWithAttachment.java |

## 8. Sub-Process Call Hierarchy

> **Not available.** This interface calls no sub-processes.

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/BPEL/FileNotification.bpel | Embedding SendMail_WithAttachment_Java_Embedding: reads Email/Subject/FilePath, holds the account and password literal, sends, deletes the file on success, audits. | Sub-process with Mail connector Send (attachment from the inbound document); no file deletion needed because nothing is written to disk. | Low |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/SCA-INF/src/com/osi/SendMailWithAttachment.java | Builds and sends the multipart email with the file attachment; deleteFile helper. | Mail connector operation op-mail-send-attachment; file name from DDP_FileName. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Try/catch in the Java embedding; error written to audit trail; callback always sent; no BPEL fault handler. |
| Retries | None. |
| Compensation | None. A failed send leaves the file on disk (not deleted). |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Utilities/Notification/filenotification_client_ep.wsdl | mds | yes |
| Domain.dvm (Email attachment folder used by callers) | dvm | yes |
| EmailDL.dvm (recipient lists chosen by callers) | dvm | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Gmail SMTP | SMTP/STARTTLS | smtp.gmail.com:587 | Google (TaskUs Google Workspace account) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | FileNotification endpoint | http://129.144.156.28:[redacted] | [redacted].wsdl |
| DEV | FileNotification endpoint | http://129.144.156.126:[redacted] | [redacted].wsdl |
| PROD | Attachment folder (Domain.dvm Email) | [redacted] | [redacted].dvm |

## 14. Boomi Migration Readiness Summary

This utility sends a report file as an email attachment. Other integrations build CSV reports during their runs and save them in a folder on the integration server; at the end of the run they ask this utility to email the file to the right distribution list. After a successful send the file is deleted from the server.

In Boomi this becomes a shared sub-process that receives the report itself as the document being processed, instead of a file path on a server, and sends it with the same mail connection used for ordinary notifications. That removes the dependency on a shared server folder.

The current utility sends an email with an empty body and only the attachment. Like the notification utility, it holds the mail account password in its code, and it never tells the caller if sending failed.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Mail Attachment SEND Op | op-mail-send-attachment | connector-action | conn-smtp-mail |
| 2 | Sub: Send File Notification | proc-sub-file-notification | process | op-mail-send-attachment |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/BPEL/FileNotification.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/Schemas/FileNotification.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/FileNotification/SOA/SCA-INF/src/com/osi/SendMailWithAttachment.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/Notification/filenotification_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/DEV_MDS/apps/Utilities/Notification/filenotification_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/BPEL/ProcessEmpCampaignChange_Sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Domain.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm

## Open questions

### Still outstanding

REVIEW-NEEDED: Should report emails keep an empty body, or include a short standard message describing the attached report? (Decides the content of every report email sent by the migrated integrations.)
  - If this is not answered: Keep an empty body, as today; the subject and attachment carry the information.

REVIEW-NEEDED: If a report email cannot be sent, who should be told, and should the report be kept for re-sending? (Today a failed send is only logged on the server and the report file is left behind; recipients never learn a report was missed.)
  - If this is not answered: Log the failure, send an error notification to the "notification" distribution list, and do not retain the report.

REVIEW-NEEDED: Are there any limits on attachment size or on sending files containing personal data by email that the migrated reports must respect? (Reports include employee identifiers and, in the current new-hire flow, passwords; mail policy may require changes to what is attached.)
  - If this is not answered: Attachments are sent as today, without passwords (per the separate password-handling decision), with no size limit applied.

