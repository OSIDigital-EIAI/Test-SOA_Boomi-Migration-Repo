# ProcessCreateADUser

Test-OracleSOA-Project · Listener · complex complexity

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

Creates the Active Directory (AD) account for a new hire. Given the employee record from the new-hire job, it reads the worker's department, project, grade, country, job, location and manager from Oracle HCM, checks whether an AD account with that employee ID already exists, looks up the manager in AD, derives the organisational unit (OU) and security group from the project ID, creates and enables the account, adds it to its groups, loads the AD user name into HCM as the Fusion user name, and notifies the manager and HR.

| Field | Value |
| --- | --- |
| Interface | ProcessCreateADUser |
| Type | Listener |
| Complexity | complex |
| Components to build | 8 |
| Mapping rules | 25 |
| Business rules | 7 |

## Interface Summary

This service creates a network (Active Directory) account for each new hire. The new-hire job calls it with the person's ID and name. It looks up the rest of the details in HCM (department, project, grade, country, job title, location, manager), checks whether the person already has an account, and if not creates one in the right part of the directory for their project, sets it active, adds it to the right security group and makes the HCM user name match. The account name follows the TaskUs rule: first letter of the first name, first letter of the last name, then the employee number, all lower case.  Where the account goes and which group it joins are decided by project-ID lookup tables, with separate tables for the US and the Philippines and, in the Philippines, for team leads (grade codes starting "SA-"), agents ("A-") and everyone else. Team leads are also added to the corporate Wi-Fi group. If the project is not in the table the account goes into a general "Floating" area with no project group.  The most important finding is about passwords. Every new account is given the same fixed initial password written into the code, and that password is included in the success email sent to the manager and HR. A random password is generated as well, but it is never set in AD: it is written to the server log and returned to the calling job. This must be redesigned, not copied. Other points to settle: who receives the emails (today a fixed list of named people), what happens when two hires share a name in the same area, and whether a person with a disabled old account should get a new one.

**Complexity:** complex (score 78)

## Functional Analysis

1. **Receive new hire** — Async SOAP process operation with EmployeeEBM EmployeeRequest (createInstance).
2. **Enrich from HCM** — If EmployeeID is not empty: OTBI query on "Workforce Management - Worker Assignment Real Time" for Person Number = EmployeeID; sets HiringManager email (Column12), first/last name split from "Last, First" Manager Name (Column10), manager person number (Column11), JobTitle (Column8), WorkLocation (Column9), Country (Column6), CountryCode (Column7).
3. **Check for existing account** — LDAP search by employeeID. Treated as new when entriesReturned contains "0" or the result mentions "OU=Disabled Accounts".
4. **Find manager in AD** — Search by employeeID = manager person number; if none found and the manager first name contains "Ma. ", search by sAMAccountName built from the name after "Ma. ". Manager DN taken from the response text.
5. **Generate random password** — Java PwdGenerationClass.randomString(12,1,1,1,1); value written to the audit trail; not used for the AD account.
6. **Derive OU and department** — Country US: US\_OUNames OU and US\_ADGroupDNs Department; grade SA-: OUNames + ADGroupDNs\_TLS; grade A-: OUNames + ADGroupDNs; otherwise OUNames + ADGroupDNs\_Others. Key = project ID (Column4).
7. **Create account** — xform\_User\_Details builds the LDAP add (see MAP-001); AddUser add with objectClass organizationalPerson, person, top, user.
8. **Confirm creation** — Search by employeeID again; created when entriesReturned contains "1".
9. **Enable account** — Modify: pwdLastSet 0 (replace), userPrincipalName \<sAM\>@taskus.prv (add), unicodePwd hardcoded literal (add), userAccountControl 512 (replace).
10. **Add to groups** — Grade SA- (non-US): add member to CN=TU-wificorp-GG,OU=Special,OU=Groups,OU=TaskUs,DC=taskus,DC=prv. Then GroupDN from the matching group table; if not empty add member.
11. **Load HCM user name** — ProcessCreateUserDAT with PersonNumber and Username = sAMAccountName; waits for the async callback.
12. **Notify** — "User Created Successfully in AD" to: manager + one consultant (non-US); five named HR/recruitment people + consultant + manager (US); content includes the initial password.
13. **Existing account path** — Replace mail with the input EmailAddress on the found DN; notification skipped; status Existing.
14. **Return result** — Callback EmployeeResponse/TransmissionAck with status Success, Existing or Failed and identifiers.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessCreateADUser/processcreateaduser\_client\_ep | none declared |
| outbound | LDAP | JNDI eis/ldap/TestConnection (host not in export) | bind account configured in WebLogic |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared |
| outbound | SOAP async (composite) | oramds:/apps/Provider/AD/processcreateuserdat\_client\_ep.wsdl | none declared |
| outbound | SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification\_client\_ep.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Employee EBM and HCM row to AD add request | dn | Person/FirstName, Person/LastName, Campaign (OU) | conditional | Campaign not empty | "CN=" + First + " " + Last + "," + OU; else ",OU=Floating,OU=Operations,OU=Users,OU=TaskUs,DC=taskus,DC=prv" | medium | OU from US\_OUNames or OUNames by project ID. |
| MR-002 | Employee EBM and HCM row to AD add request | cn / displayName | Person/FirstName, Person/LastName | calculated | — | First + " " + Last | low | — |
| MR-003 | Employee EBM and HCM row to AD add request | c | ContactAddress/CountryCode (OTBI Column7) | passthrough | — | — | low | — |
| MR-004 | Employee EBM and HCM row to AD add request | co | ContactAddress/Country (OTBI Column6) | passthrough | — | — | low | — |
| MR-005 | Employee EBM and HCM row to AD add request | company | (none) | hardcoded | — | "TaskUS" | low | — |
| MR-006 | Employee EBM and HCM row to AD add request | department | OTBI Column4 (project ID) | conditional | Only when the lookup returns a value | Department from US\_ADGroupDNs / ADGroupDNs\_TLS / ADGroupDNs / ADGroupDNs\_Others by country and grade | medium | Overrides the HCM department name. |
| MR-007 | Employee EBM and HCM row to AD add request | description | (none) | hardcoded | — | "Employee" | low | — |
| MR-008 | Employee EBM and HCM row to AD add request | employeeID | EmployeeID | passthrough | — | — | low | HCM person number. |
| MR-009 | Employee EBM and HCM row to AD add request | givenName | Person/FirstName | passthrough | — | — | low | — |
| MR-010 | Employee EBM and HCM row to AD add request | sn | Person/LastName | passthrough | — | — | low | — |
| MR-011 | Employee EBM and HCM row to AD add request | mail | Person/EmailAddress | conditional | not empty | — | low | — |
| MR-012 | Employee EBM and HCM row to AD add request | manager | GetManagerfromAD response dn | transformed | — | Text between ' dn="' and '"\>' of the search response | medium | Empty when the manager is not found. |
| MR-013 | Employee EBM and HCM row to AD add request | mobile | EmployeePSFTType/MobilePhone | conditional | not empty | — | low | — |
| MR-014 | Employee EBM and HCM row to AD add request | physicalDeliveryOfficeName | EmploymentAssignments/WorkLocation (OTBI Column9) | conditional | not empty | ADOfficeLocations: lower-case Location -\> Code, default empty | medium | — |
| MR-015 | Employee EBM and HCM row to AD add request | pwdLastSet | (none) | hardcoded | — | "0" | low | Forces change at first logon. |
| MR-016 | Employee EBM and HCM row to AD add request | sAMAccountName | Person/FirstName, Person/LastName, EmployeeID | calculated | — | lower(first character of FirstName) + lower(first character of LastName) + EmployeeID | medium | Legacy idiom substring-before(x, substring(x,2)) returns empty when the 2nd character equals the 1st (fn-ad-sam-build). |
| MR-017 | Employee EBM and HCM row to AD add request | title | EmploymentAssignments/JobType/JobTitle (OTBI Column8) | conditional | not empty | — | low | — |
| MR-018 | Result to TransmissionAck | status | (outcome) | conditional | — | "Success" \| "Existing" \| "Failed" | low | — |
| MR-019 | Result to TransmissionAck | TransmissionMessageID / TransmissionID / SourceObjectName / SourceObjectIdentifier | OTBI Row Column1 / Column4 / Column5 / Column6 | passthrough | — | — | low | Department name, project ID, project description, country (fields reused). |
| MR-020 | Result to TransmissionAck | employeeID / candidateID | EmployeeID / Person/EmailAddress | passthrough | — | — | low | — |
| MR-021 | Result to TransmissionAck | attribute1 | sAMAccountName | calculated | — | sAMAccountName + "@taskus.prv" on success; sAMAccountName otherwise | low | — |
| MR-022 | Result to TransmissionAck | attribute2 | cn | passthrough | — | — | low | CN from the search on the Existing path. |
| MR-023 | Result to TransmissionAck | ReferenceValues/Value | Generated password | passthrough | Success only | — | low | Should be removed in Boomi (security). |
| MR-024 | Result to TransmissionAck | ReferenceValues/UserDefinedID | ProcessCreateUserDAT result | passthrough | Success only | — | low | — |
| MR-025 | Result to TransmissionAck | MessageText | Fault text | conditional | Failed | Text inside \<detail\> when present, else the full fault string | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Account name convention | sAMAccountName = lower(first character of FirstName) + lower(first character of LastName) + EmployeeID; UPN = sAMAccountName@taskus.prv; CN = First Last. | High |
| BR-002 | Existing account detection | An account with the same employeeID that is not in OU=Disabled Accounts means the person already exists; only mail is updated. | High |
| BR-003 | OU placement | US (country code US): US\_OUNames by project ID; otherwise OUNames by project ID; empty -\> Floating OU. | High |
| BR-004 | Group membership | US: US\_ADGroupDNs; PH grade SA- (team lead): Wi-Fi group plus ADGroupDNs\_TLS; grade A- (agent): ADGroupDNs; other grades: ADGroupDNs\_Others. No group when the lookup is empty. | High |
| BR-005 | Account activation | userAccountControl 512 (normal enabled account) and pwdLastSet 0 (change password at next logon). | High |
| BR-006 | Manager lookup fallback | If no AD account has the manager's employeeID and the manager first name contains "Ma. ", search by sAMAccountName built from the name after "Ma. ". | Medium |
| BR-007 | HCM user name alignment | After a successful creation, HCM user name is set to sAMAccountName. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive new hire** — EmployeeEBM from newhire-emp-sch
2. **Enrich from HCM** — OTBI by person number
3. **Search AD by employeeID** — VerifyUser
4. **Active account exists?** — count 0 or Disabled Accounts = new
5. **Update mail, return Existing** — ADEmailUpdate
6. **Find manager in AD** — employeeID, then "Ma. " fallback
7. **Derive OU and department** — Country / grade / project tables
8. **Create AD account** — LDAP add
9. **Account found after add?** — ConfirmUser count contains 1
10. **Fault: return Failed** — catchAll; notification skipped
11. **Return Failed** — Notification skipped
12. **Enable account** — UPN, password, UAC 512
13. **Add to groups** — Wi-Fi (SA-) and project group
14. **Load HCM user name** — hcm-create-user-dat, async
15. **Notify manager and HR** — Hardcoded recipients
16. **Return Success** — Callback TransmissionAck

## Data Model

_This interface owns no profile components: the request and response use the EmployeeEBM and TransmissionAck profiles of hcm-dat-generation and the OTBI rowset of hcm-otbi-sql-query; LDAP request/response shapes come from the connector._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive/callback (async SOAP) | Sub-process start + Return Documents |
| HCMExtractSyncProv | Process Call hcm-otbi-sql-query |
| SearchUser / AddUser / EnableADUser / ADDGroupToUser / ADEmailUpdate LDAP adapters | conn-ad-ldap + op-ldap-search / op-ldap-add / op-ldap-modify |
| xform\_User\_Details.xsl | map-ebm-to-ad-user + fn-ad-sam-build |
| PwdGenerationClass (Java embedding) | fn-password-generator (SecureRandom) |
| Seven DVMs | Cross Reference Tables |
| ProcessCreateUserDAT (async) | Process Call hcm-create-user-dat |
| catchAll | Try/Catch returning Failed + notification |

**Components**

| Component | Role |
| --- | --- |
| conn-ad-ldap | LDAP(S) connection to TaskUs AD |
| op-ldap-search | Search by employeeID / sAMAccountName |
| op-ldap-add | Create user entry |
| op-ldap-modify | Enable, group membership, mail update |
| fn-ad-sam-build | sAMAccountName / UPN builder |
| fn-password-generator | Initial password generator |
| map-ebm-to-ad-user | EBM + HCM row to LDAP add |
| proc-ad-create-user | Main sub-process |

## Client questions

- **Q-001** How should a new employee's first AD password be set and given to them?
  **If this is not answered:** A unique random password per account, change forced at first login, delivered by a separate channel agreed with IT; no password in emails or logs.
- **Q-002** Which Active Directory server should the new process connect to, over which connection type, and which IT team will set up its service account in Boomi?
  **If this is not answered:** LDAPS on port 636 to the production domain controller, service account configured by TaskUs IT.
- **Q-003** Who should receive the "AD user created" notice for Philippine hires and for US hires?
  **If this is not answered:** The hiring manager plus the EmailDL "AD" list for PH, and the manager plus the EmailDL "UShrisJobvite" list for US.
- **Q-004** When a new hire has the same first and last name as someone already in the same AD area, how should the account be named?
  **If this is not answered:** Append the employee number to the display name for the second person.
- **Q-005** If a returning employee still has a disabled AD account, should that account be re-enabled or should a new account be created?
  **If this is not answered:** Report the case to the IT service desk and do not create a second account.
- **Q-006** Should the IT service desk be emailed when an AD account cannot be created?
  **If this is not answered:** Yes, send failures to the EmailDL "itservicedesk" list.
- **Q-007** When a new hire's project is not in the OU and group tables, is placing the account in the general "Floating" area with no project group still correct, and should anyone be told?
  **If this is not answered:** Keep the Floating fallback and include such accounts in the success notice.
