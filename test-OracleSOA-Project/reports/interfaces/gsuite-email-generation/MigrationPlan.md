# Migration Plan — ProcessEmailGenerationABCSImpl

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.186Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessEmailGenerationABCSImpl |
| Identifier | gsuite-email-generation |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | simple (20) |
| Components to build | 2 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/BPEL/ProcessEmailGenerationABCSImplProcess.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmailGenerationABCSImpl/SOA/BPEL/ProcessEmailGenerationABCSImplProcess.bpel |
| Depends on | gsuite-get-access-token, gsuite-create-user |

## 2. End-to-End Process Architecture

Async BPEL composite with a repeatUntil loop that ends only through thrown faults; token from ProcessGetGSuitAccessToken; lookup via the Java relay (production) or Google directly (Employee variant).

Boomi sub-process "Sub: GSuite Generate Unique Email": op-google-user-get on conn-google-directory inside a bounded loop; HTTP 404 = free, 200 with matching externalIds = Existing, 200 otherwise = next number; Try/Catch with util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback | Sub-process start + Return Documents |  |
| repeatUntil + throw/catch on message text | Loop with Decision on HTTP status |  |
| GetUser_JavService relay | op-google-user-get | Direct HTTPS. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive request | Async SOAP process with Email and EmployeeID. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/Schemas/ProcessEmailGenerationABCSImplProcess.xsd |
| 2 | Get access token | Call ProcessGetGSuitAccessToken. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/BPEL/ProcessEmailGenerationABCSImplProcess.bpel |
| 3 | Look up candidate | GET getGSuiteUser on the Java relay with email and authorization query parameters. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/WADLs/GetUser_JavService.wadl |
| 4 | Same employee? | primaryEmail returned and externalIds value = EmployeeID -> Existing. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/BPEL/ProcessEmailGenerationABCSImplProcess.bpel |
| 5 | Taken by someone else | Build next candidate with counter (02-09, then 10+), loop. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/BPEL/ProcessEmailGenerationABCSImplProcess.bpel |
| 6 | Free address | Error "Resource Not Found: userKey" -> return candidate. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/BPEL/ProcessEmailGenerationABCSImplProcess.bpel |
| 7 | Callback | processResponse with Email and Status. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/BPEL/ProcessEmailGenerationABCSImplProcess.bpel |

## 4. Payload Profiles

> **Not available.** Uses the Google user profile owned by gsuite-create-user; the request/response are two-field inline schemas.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | candidate Email (loop) | process/Email, counter | looped | Address taken by another employee | local + ("0"+counter if counter < 10 else counter) + "@" + domain; counter starts at 2 | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/BPEL/ProcessEmailGenerationABCSImplProcess.bpel | Candidate email generation |
| MR-002 | Status | Google user externalIds value | conditional | = EmployeeID | "Existing" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/BPEL/ProcessEmailGenerationABCSImplProcess.bpel | Candidate email generation |
| MR-003 | Email | candidate | passthrough | Not found or Existing |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/BPEL/ProcessEmailGenerationABCSImplProcess.bpel | Candidate email generation |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Rehire keeps address | If the address exists and carries the same employee ID, return it as Existing instead of creating a new account. | High | Decision on externalIds. |
| BR-002 | Numbered alternatives | Clashing address gets 02..09 then 10.. appended to the local part. | Medium | Loop with counter property. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | processemailgenerationabcsimplprocess_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/GSuit/processemailgenerationabcsimplprocess_client_ep.wsdl |
| outbound REST over plain HTTP | http://129.144.156.139:9073/google-restservice-context-root/resources/googleServices/getGSuiteUser | Google bearer token as a query parameter | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/soaconfigplan.xml |
| outbound REST over HTTPS (from relay) | https://www.googleapis.com/admin/directory/v1/users/{email} | OAuth 2.0 bearer | legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/client/GetGSuiteUser_ClientClass.java |
| REST reference GetUser_JavService | WADL /googleServices/getGSuiteUser GET, query params email and authorization; PROD location http://129.144.156.139:[redacted]. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/WADLs/GetUser_JavService.wadl |
| REST reference GetUser (skipped) | Direct Google users.get kept but skipped. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/BPEL/ProcessEmailGenerationABCSImplProcess.bpel |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| gsuite-get-access-token | Google Directory connection (OAuth 2.0) | synchronous |
| gsuite-create-user | Google user profile (prof-google-user-json) | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/client/GetGSuiteUser_ClientClass.java | Relay GET user to Google. | op-google-user-get on conn-google-directory. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Faults used for control flow; catchAll branches on message text; other errors notify and call back empty. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| GSuiteServices Java relay | library | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Google Admin SDK Directory API | HTTPS REST | https://www.googleapis.com/admin/directory/v1/users | Google |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Java relay | http://129.144.156.139:[redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

Before a Google account is created for a new hire, this service checks that the proposed email address is free. It asks Google whether the address exists. If nobody has it, that address is used. If it already belongs to the same employee (for example a rehire), it reports that the account already exists. If it belongs to someone else, it adds a number and tries again: firstname.lastname02, 03 and so on.

The logic works by deliberately raising errors and recognising them by their message text ("Resource Not Found", "Employee already created in Gsuite"), which is fragile: if Google or the relay changes the wording, every new hire would be treated as an error. The numbering starts at 02, never 01. Unexpected errors are emailed in a format the notification service cannot read and the caller receives an empty answer.

In production the lookup goes through the custom Java relay, and the Google access key is sent in the web address of a plain-HTTP request, where it can end up in server logs. In Boomi this becomes a short loop calling Google directly and checking the HTTP status (404 = free).

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Google User GET Op | op-google-user-get | connector-action | conn-google-directory |
| 2 | Sub: GSuite Generate Unique Email | proc-sub-gsuite-unique-email | process | op-google-user-get, prof-google-user-json |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/BPEL/ProcessEmailGenerationABCSImplProcess.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/soaconfigplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/Schemas/ProcessEmailGenerationABCSImplProcess.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/WADLs/GetUser_JavService.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmailGenerationABCSImpl/SOA/BPEL/ProcessEmailGenerationABCSImplProcess.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmailGenerationABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/service/GoogleServices.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/client/GetGSuiteUser_ClientClass.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/GSuit/processemailgenerationabcsimplprocess_client_ep.wsdl

## Open questions

### Still outstanding

REVIEW-NEEDED: When a proposed email is taken, should the next option be firstname.lastname01 rather than 02, and is a number the preferred way to make it unique? (Numbering starts at 02 today; changing the convention affects new addresses only.)
  - If this is not answered: Keep the current convention (02, 03 ...).

REVIEW-NEEDED: Do all existing Google accounts carry the employee ID, so that rehires are recognised by it? (A rehire whose old account lacks the employee ID gets a new numbered address and a second account.)
  - If this is not answered: Yes.

