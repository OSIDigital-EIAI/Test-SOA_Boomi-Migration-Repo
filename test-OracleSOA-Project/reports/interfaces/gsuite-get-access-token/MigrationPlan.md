# Migration Plan — ProcessGetGSuitAccessToken

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.139Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessGetGSuitAccessToken |
| Identifier | gsuite-get-access-token |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | simple (8) |
| Components to build | 1 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/BPEL/ProcessGetGSuitAccessToken.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessGetGSuitAccessToken/SOA/BPEL/ProcessGetGSuitAccessToken.bpel |
| Depends on | Nothing |

## 2. End-to-End Process Architecture

Synchronous BPEL composite whose production variant calls a custom JAX-RS service (GSuiteServices) over HTTP; the service calls Google. The Employee variant calls Google directly. Both hold the OAuth secrets as literals.

No Boomi process. conn-google-directory (HTTP Client, base https://www.googleapis.com/admin/directory/v1) uses OAuth 2.0 with secrets entered in Boomi (or a service account with domain-wide delegation). Callers use the connection directly.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| ProcessGetGSuitAccessToken composite | OAuth 2.0 on conn-google-directory | No process. |
| GSuiteServices Java REST service | (retired) | Boomi calls Google directly over HTTPS. |
| Hardcoded OAuth literals | Encrypted connection fields, rotated values | Never copy the legacy values. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive request | Synchronous process (input element Test, unused). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/Schemas/ProcessGetGSuitAccessToken.xsd |
| 2 | Build token request | Hardcoded client_id, client_secret, refresh_token and grant_type refresh_token [values redacted]. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/BPEL/ProcessGetGSuitAccessToken.bpel |
| 3 | Call token service | Production: Java REST service getAccessToken; Employee variant: Google /oauth2/v3/token directly. HTTP Connection: close. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/WADLs/GetAccessToken_JavaService.wadl |
| 4 | Reply | Access_Token, Expires_IN, Token_Type, Error. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/BPEL/ProcessGetGSuitAccessToken.bpel |

## 4. Payload Profiles

> **Not available.** The only component is the Google connection; no profiles are needed once OAuth is handled by the connection.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | grant_type | (none) | hardcoded |  | "refresh_token" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/BPEL/ProcessGetGSuitAccessToken.bpel | Token request |
| MR-002 | client_id / client_secret / refresh_token | (none) | hardcoded |  | [redacted literals] | low | Move to Boomi connection; rotate. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/BPEL/ProcessGetGSuitAccessToken.bpel | Token request |
| MR-003 | Access_Token / Token_Type / Expires_IN / Error | token response | passthrough |  |  | low | Response mapping. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/BPEL/ProcessGetGSuitAccessToken.bpel | Token request |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Token per call | Every Google operation is preceded by a refresh-token grant. | Medium | OAuth 2.0 on conn-google-directory (automatic refresh). |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 synchronous | processgetgsuitaccesstoken_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/composite.xml |
| outbound REST over plain HTTP | http://129.144.156.139:9073/google-restservice-context-root/resources/googleServices/getAccessToken | none (secrets in JSON body) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/ProcessGetGSuitAccessToken_Prod_cfgplan.xml |
| outbound REST over HTTPS | https://www.googleapis.com/oauth2/v3/token | OAuth 2.0 refresh_token grant (client credentials in query string) | legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/client/GetAccessToken_ClientClass.java |
| REST reference GetAccessToken_JavaService | WADL resource [redacted], POST, application/json; location replaced by PROD cfgplan. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/WADLs/GetAccessToken_JavaService.wadl |
| REST reference GetAccessToken (skipped in production variant) | Base https://www.googleapis.com, resource /oauth2/v3/token, POST, [redacted] in, JSON out. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/WADLs/GetAccessToken.wadl |

## 8. Sub-Process Call Hierarchy

> **Not available.** This interface calls no sub-processes.

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/client/GetAccessToken_ClientClass.java | Call Google token endpoint with credentials in the URL; return JSON. | OAuth 2.0 settings on conn-google-directory. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Production: none in BPEL, errors returned in Error. Employee variant: catchAll notify + rethrow. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| GSuiteServices Java application | library | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Google OAuth 2.0 | HTTPS | https://www.googleapis.com/oauth2/v3/token | Google |
| GSuiteServices (custom Java) | HTTP | http://129.144.156.139:9073/google-restservice-context-root | TaskUs / OSI |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Java service location | http://129.144.156.139:[redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

Before any integration can create or change a Google Workspace account it needs a short-lived access key from Google. This service obtains one. It sends Google a long-lived "refresh token" together with the application's ID and secret, and passes the access key back to the caller.

The ID, secret and refresh token are written directly in the integration code, and in production they are sent over plain HTTP to a small custom Java web service running on another server, which forwards them to Google with the secret in the web address. These values have been exposed in the source code and must be replaced (rotated) as part of the migration. The custom Java service was apparently introduced because the integration server could not validate Google's certificate.

In Boomi this service disappears: the Google connection is configured with OAuth 2.0 and refreshes tokens itself, with the secrets stored encrypted in Boomi. TaskUs should also consider a Google service account with domain-wide delegation instead of a personal refresh token.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Google Admin SDK Directory Conn | conn-google-directory | connector-settings | — |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/BPEL/ProcessGetGSuitAccessToken.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/ProcessGetGSuitAccessToken_Prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/WADLs/GetAccessToken.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/WADLs/GetAccessToken_JavaService.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessGetGSuitAccessToken/SOA/Schemas/ProcessGetGSuitAccessToken.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessGetGSuitAccessToken/SOA/BPEL/ProcessGetGSuitAccessToken.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessGetGSuitAccessToken/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/service/GoogleServices.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/client/GetAccessToken_ClientClass.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessEmailGenerationABCSImpl/SOA/composite.xml

## Open questions

### Still outstanding

REVIEW-NEEDED: Will TaskUs's Google administrators create a new OAuth client or service account for Boomi and revoke the credentials used by the current integration? (The current Google credentials are written in the integration code and must be considered exposed; the Boomi Google connection cannot be set up without new ones.)
  - If this is not answered: A new service account with domain-wide delegation for the Admin SDK user and alias scopes, created by TaskUs, with the old credentials revoked at cut-over.

REVIEW-NEEDED: Can the custom Java Google service on the separate server be retired once Boomi calls Google directly? (It is only a relay to Google; other systems may also use it.)
  - If this is not answered: Yes, retire it after cut-over.

