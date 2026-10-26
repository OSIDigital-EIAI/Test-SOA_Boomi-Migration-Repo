# Migration Plan — ProcessCreateGSuiteAlias

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.206Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessCreateGSuiteAlias |
| Identifier | gsuite-create-alias |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | simple (10) |
| Components to build | 1 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/BPEL/ProcessCreateGSuiteAlias.bpel |
| Depends on | gsuite-create-user, util-send-notification |

## 2. End-to-End Process Architecture

Synchronous BPEL composite: ProcessGetGSuitAccessToken, REST call to Google, XXSendNotification. Only an Employee copy exists.

Boomi sub-process "Sub: GSuite Alias CREATE": op-google-alias-insert (gsuite-create-user) on conn-google-directory, return alias/status; Try/Catch with util-send-notification; success notice optional.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / reply | Sub-process start + Return Documents | Reply also on failure. |
| CreateGSuitAlias REST reference | op-google-alias-insert |  |
| Notifications | util-send-notification | Fix format. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive request | Synchronous process with email and alias. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/Schemas/ProcessCreateGSuiteAlias.xsd |
| 2 | Get access token | ProcessGetGSuitAccessToken. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/BPEL/ProcessCreateGSuiteAlias.bpel |
| 3 | Create alias | POST alias JSON to users/{email}/aliases with bearer token. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/WADLs/CreateGSuitAlias.wadl |
| 4 | Notify success | EmailDL itservicedesk "Successfully created Alias for Gsuite EmailID". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/BPEL/ProcessCreateGSuiteAlias.bpel |
| 5 | Reply | alias, kind, etag, id. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/BPEL/ProcessCreateGSuiteAlias.bpel |

## 4. Payload Profiles

> **Not available.** Uses the alias profile owned by gsuite-create-user.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | alias | process/alias | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/BPEL/ProcessCreateGSuiteAlias.bpel | Alias request and response |
| MR-002 | URI userKey | process/email | passthrough |  |  | low | Property name mismatch in legacy. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/BPEL/ProcessCreateGSuiteAlias.bpel | Alias request and response |
| MR-003 | response alias/kind/etag/id | Google reply | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/BPEL/ProcessCreateGSuiteAlias.bpel | Alias request and response |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Preferred-name alias | Alias value is supplied by the caller (emp-update-sch builds it from the preferred name). | Low | op-google-alias-insert. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 synchronous | http://129.144.156.28:80/soa-infra/services/default/ProcessCreateGSuiteAlias/processcreategsuitealias_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/GSuit/processcreategsuitealias_client_ep.wsdl |
| outbound REST over HTTPS | https://www.googleapis.com/admin/directory/v1/users/{Email_ID}/aliases | OAuth 2.0 bearer | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/WADLs/CreateGSuitAlias.wadl |
| REST reference CreateGSuitAlias | WADL CreateGSuitAlias.wadl, POST, template param Email_ID; HTTP Connection close. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/WADLs/CreateGSuitAlias.wadl |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| gsuite-create-user | Google alias operation and profile | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

> **Not available.** No custom code was found in this interface.

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll notification only; no reply. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/GSuit/ProcessGetGSuitAccessToken.wsdl | mds | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Google Admin SDK Directory API | HTTPS REST | https://www.googleapis.com/admin/directory/v1 | Google |

## 13. Environment Configuration Matrix

> **Not available.** No environment-specific configuration was recorded.

## 14. Boomi Migration Readiness Summary

When an employee's preferred name changes in HCM, this service adds that name as an extra email address on their Google account. It gets an access key, asks Google to add the alias and tells the IT service desk.

It calls Google directly (not through the Java relay). However, the web-address template expects a parameter called "Email_ID" while the code supplies one called "email", so the employee's address may never be filled into the request; whether the live service works cannot be confirmed from the export. Both the success and error emails use a format the notification service cannot read, so they are probably never delivered, and on error the caller receives no reply.

In Boomi this is a small sub-process using the shared alias operation from gsuite-create-user.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Sub: GSuite Alias CREATE | proc-sub-gsuite-create-alias | process | op-google-alias-insert, prof-google-alias-json, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/BPEL/ProcessCreateGSuiteAlias.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/Schemas/ProcessCreateGSuiteAlias.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/WADLs/CreateGSuitAlias.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuiteAlias/SOA/WADLs/CreateAlias.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/GSuit/processcreategsuitealias_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/composite.xml

## Open questions

### Still outstanding

REVIEW-NEEDED: Are preferred-name aliases currently being created in Google when employees change their known-as name? (The legacy request may be malformed; if aliases have not been created, TaskUs may want a one-off catch-up.)
  - If this is not answered: Build it working as designed; no catch-up.

REVIEW-NEEDED: Does the IT service desk want an email for every alias created, or only for failures? (Today a success email is sent each time (though probably not delivered).)
  - If this is not answered: Failures only.

