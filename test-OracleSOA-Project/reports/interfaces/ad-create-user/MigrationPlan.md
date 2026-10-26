# Migration Plan — ProcessCreateADUser

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.883Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessCreateADUser |
| Identifier | ad-create-user |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | complex (78) |
| Components to build | 8 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/SCA-INF/src/processcreateaduser/PwdGenerationClass.java |
| Depends on | hcm-otbi-sql-query, hcm-create-user-dat, hcm-dat-generation, util-send-notification |

## 2. End-to-End Process Architecture

Async BPEL composite called by ProcessNewHireEmpReq_Sch. It calls ProcessHCMExtractSyncProv (MDS WSDL) for OTBI, five LDAP adapter references on eis/ldap/TestConnection, ProcessCreateUserDAT (async, MDS WSDL) and XXSendNotification, uses seven MDS DVMs and an embedded Java password generator, and returns a TransmissionAck by WS-Addressing callback. ProcessCreateADUser_temp.bpel is present but not wired in composite.xml.

Boomi sub-process "AD User CREATE" called synchronously by the new-hire process (a Web Services Server listener only if other callers exist). Steps: Process Call hcm-otbi-sql-query; op-ldap-search; decision; manager search; cross reference lookups; map-ebm-to-ad-user; op-ldap-add; op-ldap-search confirm; op-ldap-modify for enable and groups; Process Call hcm-create-user-dat; notification through util-send-notification to EmailDL keys; return the TransmissionAck. Password handling per the security decision; no password in logs, emails or the response.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive/callback (async SOAP) | Sub-process start + Return Documents | Synchronous Process Call from newhire-emp-sch. |
| HCMExtractSyncProv | Process Call hcm-otbi-sql-query |  |
| SearchUser / AddUser / EnableADUser / ADDGroupToUser / ADEmailUpdate LDAP adapters | conn-ad-ldap + op-ldap-search / op-ldap-add / op-ldap-modify | unicodePwd and userAccountControl need a connector spike; Groovy/JNDI fallback. |
| xform_User_Details.xsl | map-ebm-to-ad-user + fn-ad-sam-build |  |
| PwdGenerationClass (Java embedding) | fn-password-generator (SecureRandom) | Only if still needed after the security decision. |
| Seven DVMs | Cross Reference Tables | US_OUNames, OUNames, US_ADGroupDNs, ADGroupDNs_TLS, ADGroupDNs, ADGroupDNs_Others, ADOfficeLocations. |
| ProcessCreateUserDAT (async) | Process Call hcm-create-user-dat |  |
| catchAll | Try/Catch returning Failed + notification | Legacy notification disabled. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive new hire | Async SOAP process operation with EmployeeEBM EmployeeRequest (createInstance). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel |
| 2 | Enrich from HCM | If EmployeeID is not empty: OTBI query on "Workforce Management - Worker Assignment Real Time" for Person Number = EmployeeID; sets HiringManager email (Column12), first/last name split from "Last, First" Manager Name (Column10), manager person number (Column11), JobTitle (Column8), WorkLocation (Column9), Country (Column6), CountryCode (Column7). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel |
| 3 | Check for existing account | LDAP search by employeeID. Treated as new when entriesReturned contains "0" or the result mentions "OU=Disabled Accounts". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xfrom_verify_aduser.xsl |
| 4 | Find manager in AD | Search by employeeID = manager person number; if none found and the manager first name contains "Ma. ", search by sAMAccountName built from the name after "Ma. ". Manager DN taken from the response text. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xfrom_getManager_request.xsl |
| 5 | Generate random password | Java PwdGenerationClass.randomString(12,1,1,1,1); value written to the audit trail; not used for the AD account. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/SCA-INF/src/processcreateaduser/PwdGenerationClass.java |
| 6 | Derive OU and department | Country US: US_OUNames OU and US_ADGroupDNs Department; grade SA-: OUNames + ADGroupDNs_TLS; grade A-: OUNames + ADGroupDNs; otherwise OUNames + ADGroupDNs_Others. Key = project ID (Column4). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel |
| 7 | Create account | xform_User_Details builds the LDAP add (see MAP-001); AddUser add with objectClass organizationalPerson, person, top, user. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl |
| 8 | Confirm creation | Search by employeeID again; created when entriesReturned contains "1". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_cofrim_adUser.xsl |
| 9 | Enable account | Modify: pwdLastSet 0 (replace), userPrincipalName <sAM>@taskus.prv (add), unicodePwd hardcoded literal (add), userAccountControl 512 (replace). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel |
| 10 | Add to groups | Grade SA- (non-US): add member to CN=TU-wificorp-GG,OU=Special,OU=Groups,OU=TaskUs,DC=taskus,DC=prv. Then GroupDN from the matching group table; if not empty add member. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel |
| 11 | Load HCM user name | ProcessCreateUserDAT with PersonNumber and Username = sAMAccountName; waits for the async callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel |
| 12 | Notify | "User Created Successfully in AD" to: manager + one consultant (non-US); five named HR/recruitment people + consultant + manager (US); content includes the initial password. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel |
| 13 | Existing account path | Replace mail with the input EmailAddress on the found DN; notification skipped; status Existing. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Adapters/ADEmailUpdate_ldap.jca |
| 14 | Return result | Callback EmployeeResponse/TransmissionAck with status Success, Existing or Failed and identifiers. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel |

## 4. Payload Profiles

> **Not available.** This interface owns no profile components: the request and response use the EmployeeEBM and TransmissionAck profiles of hcm-dat-generation and the OTBI rowset of hcm-otbi-sql-query; LDAP request/response shapes come from the connector.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | dn | Person/FirstName, Person/LastName, Campaign (OU) | conditional | Campaign not empty | "CN=" + First + " " + Last + "," + OU; else ",OU=Floating,OU=Operations,OU=Users,OU=TaskUs,DC=taskus,DC=prv" | medium | OU from US_OUNames or OUNames by project ID. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-002 | cn / displayName | Person/FirstName, Person/LastName | calculated |  | First + " " + Last | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-003 | c | ContactAddress/CountryCode (OTBI Column7) | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-004 | co | ContactAddress/Country (OTBI Column6) | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-005 | company | (none) | hardcoded |  | "TaskUS" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-006 | department | OTBI Column4 (project ID) | conditional | Only when the lookup returns a value | Department from US_ADGroupDNs / ADGroupDNs_TLS / ADGroupDNs / ADGroupDNs_Others by country and grade | medium | Overrides the HCM department name. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel | Employee EBM and HCM row to AD add request |
| MR-007 | description | (none) | hardcoded |  | "Employee" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-008 | employeeID | EmployeeID | passthrough |  |  | low | HCM person number. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-009 | givenName | Person/FirstName | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-010 | sn | Person/LastName | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-011 | mail | Person/EmailAddress | conditional | not empty |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-012 | manager | GetManagerfromAD response dn | transformed |  | Text between ' dn="' and '">' of the search response | medium | Empty when the manager is not found. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel | Employee EBM and HCM row to AD add request |
| MR-013 | mobile | EmployeePSFTType/MobilePhone | conditional | not empty |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-014 | physicalDeliveryOfficeName | EmploymentAssignments/WorkLocation (OTBI Column9) | conditional | not empty | ADOfficeLocations: lower-case Location -> Code, default empty | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/ADOfficeLocations.dvm | Employee EBM and HCM row to AD add request |
| MR-015 | pwdLastSet | (none) | hardcoded |  | "0" | low | Forces change at first logon. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-016 | sAMAccountName | Person/FirstName, Person/LastName, EmployeeID | calculated |  | lower(first character of FirstName) + lower(first character of LastName) + EmployeeID | medium | Legacy idiom substring-before(x, substring(x,2)) returns empty when the 2nd character equals the 1st (fn-ad-sam-build). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-017 | title | EmploymentAssignments/JobType/JobTitle (OTBI Column8) | conditional | not empty |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | Employee EBM and HCM row to AD add request |
| MR-018 | status | (outcome) | conditional |  | "Success" \| "Existing" \| "Failed" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel | Result to TransmissionAck |
| MR-019 | TransmissionMessageID / TransmissionID / SourceObjectName / SourceObjectIdentifier | OTBI Row Column1 / Column4 / Column5 / Column6 | passthrough |  |  | low | Department name, project ID, project description, country (fields reused). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel | Result to TransmissionAck |
| MR-020 | employeeID / candidateID | EmployeeID / Person/EmailAddress | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel | Result to TransmissionAck |
| MR-021 | attribute1 | sAMAccountName | calculated |  | sAMAccountName + "@taskus.prv" on success; sAMAccountName otherwise | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel | Result to TransmissionAck |
| MR-022 | attribute2 | cn | passthrough |  |  | low | CN from the search on the Existing path. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel | Result to TransmissionAck |
| MR-023 | ReferenceValues/Value | Generated password | passthrough | Success only |  | low | Should be removed in Boomi (security). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel | Result to TransmissionAck |
| MR-024 | ReferenceValues/UserDefinedID | ProcessCreateUserDAT result | passthrough | Success only |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel | Result to TransmissionAck |
| MR-025 | MessageText | Fault text | conditional | Failed | Text inside <detail> when present, else the full fault string | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel | Result to TransmissionAck |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Account name convention | sAMAccountName = lower(first character of FirstName) + lower(first character of LastName) + EmployeeID; UPN = sAMAccountName@taskus.prv; CN = First Last. | High | fn-ad-sam-build in map-ebm-to-ad-user. |
| BR-002 | Existing account detection | An account with the same employeeID that is not in OU=Disabled Accounts means the person already exists; only mail is updated. | High | op-ldap-search + Decision on result count and DN. |
| BR-003 | OU placement | US (country code US): US_OUNames by project ID; otherwise OUNames by project ID; empty -> Floating OU. | High | Cross reference tables US_OUNames / OUNames. |
| BR-004 | Group membership | US: US_ADGroupDNs; PH grade SA- (team lead): Wi-Fi group plus ADGroupDNs_TLS; grade A- (agent): ADGroupDNs; other grades: ADGroupDNs_Others. No group when the lookup is empty. | High | Cross reference tables and op-ldap-modify (member add). |
| BR-005 | Account activation | userAccountControl 512 (normal enabled account) and pwdLastSet 0 (change password at next logon). | High | op-ldap-modify. |
| BR-006 | Manager lookup fallback | If no AD account has the manager's employeeID and the manager first name contains "Ma. ", search by sAMAccountName built from the name after "Ma. ". | Medium | Second op-ldap-search with fn-ad-sam-build. |
| BR-007 | HCM user name alignment | After a successful creation, HCM user name is set to sAMAccountName. | Medium | Process Call hcm-create-user-dat. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessCreateADUser/processcreateaduser_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processcreateaduser_client_ep.wsdl |
| outbound LDAP | JNDI eis/ldap/TestConnection (host not in export) | bind account configured in WebLogic | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Adapters/AddUser_ldap.jca |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/composite.xml |
| outbound SOAP async (composite) | oramds:/apps/Provider/AD/processcreateuserdat_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/composite.xml |
| outbound SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/composite.xml |
| LDAP search (SearchUser) | JNDI eis/ldap/TestConnection; SearchScope wholeSubtree; ReturnAttributes displayName; SizeLimit 1000; TimeLimit 0; FollowReferrals ignore; AliasDereferencing never; used for VerifyUser, GetManagerfromAD and ConfirmUser. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Adapters/SearchUser_ldap.jca |
| LDAP add (AddUser) | JNDI eis/ldap/TestConnection; objectClass organizationalPerson,person,top,user; SuppressEntryAlreadyExists false. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Adapters/AddUser_ldap.jca |
| LDAP modify (EnableADUser) | JNDI eis/ldap/TestConnection; pwdLastSet, userPrincipalName, unicodePwd, userAccountControl. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Adapters/EnableADUser_ldap.jca |
| LDAP modify (ADDGroupToUser) | JNDI eis/ldap/TestConnection; member add on the group DN. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Adapters/ADDGroupToUser_ldap.jca |
| LDAP modify (ADEmailUpdate) | JNDI eis/ldap/TestConnection; mail replace. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Adapters/ADEmailUpdate_ldap.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| hcm-create-user-dat | Sub: HCM Create User DAT | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/SCA-INF/src/processcreateaduser/PwdGenerationClass.java | Random 12-character password with at least one upper, lower, digit and special character, using java.util.Random (not cryptographically secure). | fn-password-generator (Groovy, SecureRandom) if a generated password is still required after the security decision. | High |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl | LDAP add request: DN, cn, c, co, company, department, description, displayName, employeeID, givenName, mail, manager, mobile, physicalDeliveryOfficeName (ADOfficeLocations), pwdLastSet, sAMAccountName, sn, title. | map-ebm-to-ad-user with fn-ad-sam-build and the ADOfficeLocations cross reference. | Medium |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xfrom_verify_aduser.xsl | Search filter by employeeID. | Parameter on op-ldap-search. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xfrom_getManager_request.xsl | Manager search filter by employeeID = manager person number. | Parameter on op-ldap-search. | Low |
| other | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel | Oracle XPath extensions: oraext:parseXML, oraext:get-content-as-string with string slicing to read the DN and entry count, dvm:lookupValue, xp20:lower-case. | Profile-based LDAP response parsing, cross reference lookups, string functions. | Medium |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Process-level catchAll builds a Failed TransmissionAck (fault detail text) and calls back; the IT service desk notification inside it is skipped. |
| Retries | None configured. |
| Compensation | None: a created account is not removed if a later step (enable, group, HCM user name) fails. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| US_OUNames, OUNames, US_ADGroupDNs, ADGroupDNs_TLS, ADGroupDNs, ADGroupDNs_Others, ADOfficeLocations | dvm | yes |
| EmailDL.dvm | dvm | yes |
| oramds:/apps/Provider/AD/processcreateaduser_client_ep.wsdl | mds | yes |
| oramds:/apps/EnterpriseObjectLibrary/Core/EBO/Employee/V1/EmployeeEBM.xsd (PROD_MDS snapshot) | mds | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| TaskUs Active Directory | LDAP (LDAPS to confirm) | eis/ldap/TestConnection (host not in export) | TaskUs IT |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Service endpoint | http://129.144.156.28:[redacted] | [redacted].wsdl |

## 14. Boomi Migration Readiness Summary

This service creates a network (Active Directory) account for each new hire. The new-hire job calls it with the person's ID and name. It looks up the rest of the details in HCM (department, project, grade, country, job title, location, manager), checks whether the person already has an account, and if not creates one in the right part of the directory for their project, sets it active, adds it to the right security group and makes the HCM user name match. The account name follows the TaskUs rule: first letter of the first name, first letter of the last name, then the employee number, all lower case.

Where the account goes and which group it joins are decided by project-ID lookup tables, with separate tables for the US and the Philippines and, in the Philippines, for team leads (grade codes starting "SA-"), agents ("A-") and everyone else. Team leads are also added to the corporate Wi-Fi group. If the project is not in the table the account goes into a general "Floating" area with no project group.

The most important finding is about passwords. Every new account is given the same fixed initial password written into the code, and that password is included in the success email sent to the manager and HR. A random password is generated as well, but it is never set in AD: it is written to the server log and returned to the calling job. This must be redesigned, not copied. Other points to settle: who receives the emails (today a fixed list of named people), what happens when two hires share a name in the same area, and whether a person with a disabled old account should get a new one.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | TaskUs Active Directory LDAP Conn | conn-ad-ldap | connector-settings | — |
| 2 | AD Entry MODIFY Op | op-ldap-modify | connector-action | conn-ad-ldap |
| 3 | AD User ADD Op | op-ldap-add | connector-action | conn-ad-ldap |
| 4 | AD User SEARCH Op | op-ldap-search | connector-action | conn-ad-ldap |
| 5 | AD sAMAccountName Builder Script | fn-ad-sam-build | transform.function | — |
| 6 | Secure Password Generator Script | fn-password-generator | transform.function | — |
| 7 | Map \| Employee EBM to AD User Attributes | map-ebm-to-ad-user | transform.map | prof-employee-ebm-xml, prof-otbi-rowset, fn-ad-sam-build |
| 8 | AD User CREATE | proc-ad-create-user | process | op-ldap-search, op-ldap-add, op-ldap-modify, fn-password-generator, map-ebm-to-ad-user, proc-sub-otbi-query, proc-sub-hcm-create-user-dat, proc-sub-send-notification, prof-transmission-ack-xml |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/SCA-INF/src/processcreateaduser/PwdGenerationClass.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/WSDLs/ProcessCreateADUser.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Adapters/SearchUser_ldap.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Adapters/AddUser_ldap.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Adapters/EnableADUser_ldap.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Adapters/ADDGroupToUser_ldap.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Adapters/ADEmailUpdate_ldap.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xfrom_verify_aduser.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xfrom_getManager_request.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_cofrim_adUser.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processcreateaduser_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/US_OUNames.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/OUNames.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/US_ADGroupDNs.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/ADGroupDNs_TLS.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/ADGroupDNs.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/ADGroupDNs_Others.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/ADOfficeLocations.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel

## Open questions

### Still outstanding

REVIEW-NEEDED: How should a new employee's first AD password be set and given to them? (Today every account gets the same fixed password, which is also emailed to managers and HR, while a separate random password is logged and never used. A security decision on initial-password generation and delivery is needed before this process can be built.)
  - If this is not answered: A unique random password per account, change forced at first login, delivered by a separate channel agreed with IT; no password in emails or logs.

REVIEW-NEEDED: Which Active Directory server should the new process connect to, over which connection type, and which IT team will set up its service account in Boomi? (The production AD host, port and protocol and the bind account are not in the export.)
  - If this is not answered: LDAPS on port 636 to the production domain controller, service account configured by TaskUs IT.

REVIEW-NEEDED: Who should receive the "AD user created" notice for Philippine hires and for US hires? (Hardcoded recipients (one OSI consultant for PH; five TaskUs HR/recruitment people plus the consultant for US) must be replaced by distribution lists. TaskUs to confirm the lists.)
  - If this is not answered: The hiring manager plus the EmailDL "AD" list for PH, and the manager plus the EmailDL "UShrisJobvite" list for US.

REVIEW-NEEDED: When a new hire has the same first and last name as someone already in the same AD area, how should the account be named? (The account's directory name is "First Last" with no tie-breaker, so the second person's account cannot be created.)
  - If this is not answered: Append the employee number to the display name for the second person.

REVIEW-NEEDED: If a returning employee still has a disabled AD account, should that account be re-enabled or should a new account be created? (Today a disabled account is ignored and a new one is attempted, which can clash with the old account name.)
  - If this is not answered: Report the case to the IT service desk and do not create a second account.

REVIEW-NEEDED: Should the IT service desk be emailed when an AD account cannot be created? (The failure notices exist but are switched off, so failures are only visible in the new-hire job's reports.)
  - If this is not answered: Yes, send failures to the EmailDL "itservicedesk" list.

REVIEW-NEEDED: When a new hire's project is not in the OU and group tables, is placing the account in the general "Floating" area with no project group still correct, and should anyone be told? (The project tables are maintained by hand; missing projects silently fall back to the Floating area.)
  - If this is not answered: Keep the Floating fallback and include such accounts in the success notice.

