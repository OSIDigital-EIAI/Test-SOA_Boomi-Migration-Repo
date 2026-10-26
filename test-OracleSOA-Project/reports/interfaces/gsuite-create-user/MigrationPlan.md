# Migration Plan — ProcessCreateGSuitUser

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.166Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessCreateGSuitUser |
| Identifier | gsuite-create-user |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | medium (30) |
| Components to build | 6 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/BPEL/ProcessCreateGSuitUserProcess.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuitUser/SOA/BPEL/ProcessCreateGSuitUserProcess.bpel |
| Depends on | gsuite-get-access-token, util-send-notification |

## 2. End-to-End Process Architecture

Async BPEL composite that calls ProcessGetGSuitAccessToken, then a custom Java REST relay (GSuiteServices) over HTTP for user and alias creation; the direct Google REST reference is still present but skipped. The Employee copy is an older version calling Google directly.

Boomi sub-process "Sub: GSuite User CREATE": map-create-req-to-google-user, op-google-user-insert and op-google-alias-insert on conn-google-directory (OAuth 2.0), Try/Catch and HTTP-status decisions routing licence errors to the service desk list, return result/workemail/alias.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback | Sub-process start + Return Documents |  |
| GetGSuitAccessToken call | (none) OAuth on conn-google-directory |  |
| xfrom_HCM_GSuite.xsl | map-create-req-to-google-user | No fixed password. |
| Java relay createGsuiteUser / createGsuiteAlias | op-google-user-insert / op-google-alias-insert | Direct HTTPS to Google. |
| catchAll text routing | Try/Catch + util-send-notification | Also inspect HTTP error replies. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive request | Async SOAP process with primaryEmail, employeeID, name (givenName, familyName, AliasName), password, emails, addresses, organizations, phones. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Schemas/ProcessCreateGSuitUserProcess.xsd |
| 2 | Get access token | Synchronous call to ProcessGetGSuitAccessToken. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/BPEL/ProcessCreateGSuitUserProcess.bpel |
| 3 | Build user | xfrom_HCM_GSuite: user JSON plus authorization field (see MAP-001). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl |
| 4 | Create user | Java relay createGsuiteUser -> Google users.insert. | legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/client/CreateGSuiteUser_ClientClass.java |
| 5 | User created? | Reply primaryEmail not empty; else result = reply error. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/BPEL/ProcessCreateGSuitUserProcess.bpel |
| 6 | Create alias | If AliasName not empty: createGsuiteAlias with email, authorization and alias <AliasName>.<familyName>@taskus.com. | legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/client/CreateGSuiteAlias_ClientClass.java |
| 7 | Return result | Callback result Success, workemail, alias (or error text). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/BPEL/ProcessCreateGSuitUserProcess.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| Google Directory User JSON Prof | json | primaryEmail | string | yes |
| Google Directory User JSON Prof | json | name.givenName | string | yes |
| Google Directory User JSON Prof | json | name.familyName | string | yes |
| Google Directory User JSON Prof | json | password | string | yes |
| Google Directory User JSON Prof | json | changePasswordAtNextLogin | boolean | no |
| Google Directory User JSON Prof | json | suspended | boolean | no |
| Google Directory User JSON Prof | json | emails[].address | string | no |
| Google Directory User JSON Prof | json | addresses[].streetAddress/locality/region/postalCode/type | string | no |
| Google Directory User JSON Prof | json | externalIds[].value/type | string | no |
| Google Directory User JSON Prof | json | organizations[].name/title/description/primary/type | string | no |
| Google Directory User JSON Prof | json | phones[].value/type | string | no |
| Google Directory User JSON Prof | json | orgUnitPath | string | no |
| Google Directory User JSON Prof | json | includeInGlobalAddressList | boolean | no |
| Google Directory Alias JSON Prof | json | alias | string | yes |
| Google Directory Alias JSON Prof | json | primaryEmail | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | primaryEmail | primaryEmail | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Create request to Google user |
| MR-002 | name.givenName / name.familyName | name/givenName, name/familyName | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Create request to Google user |
| MR-003 | suspended / ipWhitelisted | (none) | hardcoded |  | false | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Create request to Google user |
| MR-004 | password | (none) | hardcoded |  | [redacted literal] | low | Must be replaced per security decision. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Create request to Google user |
| MR-005 | changePasswordAtNextLogin | (none) | hardcoded |  | false | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Create request to Google user |
| MR-006 | ims[work, gtalk, primary] | primaryEmail | passthrough |  |  | low | gtalk is obsolete. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Create request to Google user |
| MR-007 | emails.address | emails/address | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Create request to Google user |
| MR-008 | addresses[work] streetAddress, locality, region, postalCode | addresses/* | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Create request to Google user |
| MR-009 | externalIds[organization] | employeeID | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Create request to Google user |
| MR-010 | relations | (none) | hardcoded |  | value "manager", type "referred_by" | low | Constant placeholder, not the real manager. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Create request to Google user |
| MR-011 | organizations[work, primary] name, title, description | organizations/* | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Create request to Google user |
| MR-012 | phones[work] | phones/value | conditional | not empty |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Create request to Google user |
| MR-013 | orgUnitPath / includeInGlobalAddressList | (none) | hardcoded |  | "/" / true | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Create request to Google user |
| MR-014 | alias | name/AliasName, name/familyName | calculated | AliasName not empty | AliasName + "." + familyName (spaces removed) + "@taskus.com" | low | Inventory notes lower-casing for Boomi; legacy does not lower-case. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/BPEL/ProcessCreateGSuitUserProcess.bpel | Alias request |
| MR-015 | result / workemail / alias | user reply primaryEmail, alias reply alias, error | conditional | primaryEmail not empty | "Success" + email (+ alias) else error text | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/BPEL/ProcessCreateGSuitUserProcess.bpel | Result |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Preferred-name alias | If a preferred name exists, add alias <PreferredName>.<FamilyName>@taskus.com. | Medium | Decision + op-google-alias-insert. |
| BR-002 | Licence error routing | "Insufficient licenses" and "Domain user limit reached" go to the IT service desk list; other errors to the notification list. | Medium | Try/Catch + Decision on Google error reason/message (also for HTTP error replies). |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | processcreategsuituserprocess_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/GSuit/processcreategsuituserprocess_client_ep.wsdl |
| outbound REST over plain HTTP | http://129.144.156.139:9073/google-restservice-context-root/resources/googleServices/createGsuiteUser | Google bearer token passed in the JSON body | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/ProcessCreateGSuitUser_Prod_cfgplan.xml |
| outbound REST over plain HTTP | http://129.144.156.139:9073/google-restservice-context-root/resources/googleServices/createGsuiteAlias | Google bearer token passed in the JSON body | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/composite.xml |
| outbound REST over HTTPS (from the relay) | https://www.googleapis.com/admin/directory/v1/users | OAuth 2.0 bearer | legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/client/CreateGSuiteUser_ClientClass.java |
| outbound SOAP (composite) | oramds:/apps/Provider/GSuit/ProcessGetGSuitAccessToken.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/composite.xml |
| REST reference CreateGSuiteUser | WADL [redacted] POST JSON; PROD location http://129.144.156.139:[redacted]. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/WADLs/CreateGSuiteUser.wadl |
| REST reference CreateGsuiteAlias_Java_Service | WADL /createGsuiteAlias POST JSON; base http://129.144.156.139:[redacted]. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/WADLs/CreateGsuiteAlias_Java_Service.wadl |
| REST reference CreateUser (skipped) | Direct https://www.googleapis.com call kept but skipped (skipCondition 1=1). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/BPEL/ProcessCreateGSuitUserProcess.bpel |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| gsuite-get-access-token | Google Directory connection (OAuth 2.0) | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/client/CreateGSuiteUser_ClientClass.java | Relay: POST user JSON to Google with the caller's Authorization; return JSON or error. | op-google-user-insert on conn-google-directory. | Medium |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/client/CreateGSuiteAlias_ClientClass.java | Relay: POST alias to Google. | op-google-alias-insert. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl | Build the Google user resource. | map-create-req-to-google-user. | Medium |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll with message-text routing to two EmailDL lists, callback with fault text; relay errors returned as data. |
| Retries | None. |
| Compensation | None: a user created without its alias is not rolled back. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/GSuit/ProcessGetGSuitAccessToken.wsdl | mds | yes |
| GSuiteServices Java application | library | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Google Admin SDK Directory API | HTTPS REST | https://www.googleapis.com/admin/directory/v1 | Google |
| GSuiteServices (custom Java relay) | HTTP REST | http://129.144.156.139:9073/google-restservice-context-root | TaskUs / OSI |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Java relay | http://129.144.156.139:[redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

This service opens a Google Workspace account for each new employee. It gets an access key from Google, creates the user with their name, company email, work address, job title, department, phone and employee ID, and makes them visible in the company address book. If the employee has a preferred name it also creates an extra address in the form preferredname.surname@taskus.com.

The main concern is the password. In production every new Google account is created with the same fixed password written into the integration code, and the user is not forced to change it at first sign-in. This is the same password that the AD account creation uses and that appears in the new-hire reports. This must be replaced with a secure process. The older version of the service took the password from the caller instead.

Other points: all calls go to Google through a custom Java relay over plain HTTP, carrying the access key; Google errors such as "insufficient licences" come back as normal replies, so the special service-desk alerts written for them only fire when the relay itself fails; and the relay sends names in the ISO-8859-1 character set, which can damage names with characters outside it. In Boomi the process calls Google directly through the shared Google connection.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Google Directory Alias JSON Prof | prof-google-alias-json | profile.json | — |
| 2 | Google Directory User JSON Prof | prof-google-user-json | profile.json | — |
| 3 | Google User Alias CREATE Op | op-google-alias-insert | connector-action | conn-google-directory |
| 4 | Google User CREATE Op | op-google-user-insert | connector-action | conn-google-directory |
| 5 | Map \| GSuite Create Request to Google User | map-create-req-to-google-user | transform.map | prof-google-user-json |
| 6 | Sub: GSuite User CREATE | proc-sub-gsuite-create-user | process | op-google-user-insert, op-google-alias-insert, prof-google-alias-json, map-create-req-to-google-user, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/BPEL/ProcessCreateGSuitUserProcess.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/ProcessCreateGSuitUser_Prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Schemas/ProcessCreateGSuitUserProcess.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/WADLs/CreateGSuiteUser.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/WADLs/CreateGsuiteAlias_Java_Service.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuitUser/SOA/BPEL/ProcessCreateGSuitUserProcess.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuitUser/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateGSuitUser/SOA/Transformations/xform_Process_to_CreateUser.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/service/GoogleServices.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/client/CreateGSuiteUser_ClientClass.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/client/CreateGSuiteAlias_ClientClass.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/GSuit/processcreategsuituserprocess_client_ep.wsdl

## Open questions

### Still outstanding

REVIEW-NEEDED: How should a new employee's first Google password be set, and must they change it at first sign-in? (Today every Google account is created with the same fixed password, the same one used for AD, and no change is forced. This needs the same security decision as the AD account password.)
  - If this is not answered: Unique random password per account with change forced at first sign-in, delivered through the process agreed for AD; no password in emails.

REVIEW-NEEDED: Should the alias be all lower case, and what should happen if the alias address is already taken? (Aliases are built from the preferred name and surname without checks; a clash fails silently apart from an email.)
  - If this is not answered: Lower-case the alias and report clashes to the notification list.

REVIEW-NEEDED: Who should be alerted when Google refuses a new account because licences have run out? (Licence errors are meant to go to the IT service desk list, but in production they arrive as normal replies and nobody is alerted.)
  - If this is not answered: The "itservicedesk" distribution list.

REVIEW-NEEDED: Should new Google accounts be placed in a specific organisational unit rather than the top level? (All accounts are created in the root unit "/", so Google policies per site or department cannot apply automatically.)
  - If this is not answered: Keep the root unit.

