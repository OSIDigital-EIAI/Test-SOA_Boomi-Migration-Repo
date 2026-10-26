# Migration Plan — ProcessUpdateADUser

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.898Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessUpdateADUser |
| Identifier | ad-update-user |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | simple (24) |
| Components to build | 1 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/BPEL/ProcessUpdateADUser.bpel |
| Depends on | ad-create-user, hcm-otbi-sql-query, util-send-notification |

## 2. End-to-End Process Architecture

Async BPEL composite with one LDAP adapter reference (modify), ProcessHCMExtractSyncProv and XXSendNotification through MDS WSDLs, and the OUNames DVM. Called fire-and-forget by ProcessEmpAssignmentReq_Sch; also wired in ProcessEmpUpdateReq_Sch (call disabled there).

Boomi sub-process "AD User Attributes UPDATE" called synchronously by emp-assignment-sch: Process Call hcm-otbi-sql-query by person number, locate the account with op-ldap-search by employeeID (recommended instead of a constructed DN), op-ldap-modify for department/displayName, return a status document; Try/Catch with util-send-notification to the service-desk list.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback (async SOAP) | Sub-process start + Return Documents | Return a real status. |
| HCMExtractSyncProv | Process Call hcm-otbi-sql-query | Run unconditionally, keyed on Employee_ID. |
| OUNames DVM | Cross reference OUNames (from ad-create-user) |  |
| UpdateUser LDAP adapter | op-ldap-modify (ad-create-user) |  |
| catchAll + XXSendNotification | Try/Catch + util-send-notification | Fix subject/content. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive request | Async SOAP process operation (createInstance) with the ProcessUpdateADUser.xsd process element. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/Schemas/ProcessUpdateADUser.xsd |
| 2 | Read HCM details | Only if Employee_Email is not empty: OTBI by Person Number = Employee_ID; parseEscapedXML; Department := Column1. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/BPEL/ProcessUpdateADUser.bpel |
| 3 | Find OU | Grade check on Column5 (SA-, A-, other) – all branches: OU = OUNames.dvm ProjectID(Column4) -> OU, default empty. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/OUNames.dvm |
| 4 | Build DN and changes | dn = DN + "," + OU, or the Floating OU when empty; department replace with Department; displayName method and value from the input. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/BPEL/ProcessUpdateADUser.bpel |
| 5 | Modify AD entry | LDAP modify through UpdateUser (eis/ldap/TestConnection). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/Adapters/UpdateUser_ldap.jca |
| 6 | Notify success | XXSendNotification to a hardcoded individual; Subject "User <CN> Updated Successfully in <DN> #EmailID #EID #Department"; Content "User successfully updated". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/BPEL/ProcessUpdateADUser.bpel |
| 7 | Callback | processResponse with an empty outputVariable (result never set). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/BPEL/ProcessUpdateADUser.bpel |

## 4. Payload Profiles

> **Not available.** This interface owns only its process component; the request shape is a small inline schema and the LDAP shapes come from the shared connector operations of ad-create-user.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | dn | DN, OTBI Column4 (project ID) | conditional | OU found | DN + "," + OUNames(project ID); else DN + ",OU=Floating,OU=Operations,OU=Users,OU=TaskUs,DC=taskus,DC=prv" | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/BPEL/ProcessUpdateADUser.bpel | Update request to LDAP modify |
| MR-002 | modifyAttributes/department/method | (none) | hardcoded |  | "replace" | low | Department_Method from the input is ignored. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/BPEL/ProcessUpdateADUser.bpel | Update request to LDAP modify |
| MR-003 | modifyAttributes/department/attributeValue/value | Department, overwritten by OTBI Column1 | conditional | OTBI runs only when Employee_Email is not empty |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/BPEL/ProcessUpdateADUser.bpel | Update request to LDAP modify |
| MR-004 | modifyAttributes/displayName/method | DisplayName_Method | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/BPEL/ProcessUpdateADUser.bpel | Update request to LDAP modify |
| MR-005 | modifyAttributes/displayName/attributeValue/value | DisplayName | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/BPEL/ProcessUpdateADUser.bpel | Update request to LDAP modify |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | OU from project | OU = OUNames(project ID) for every grade; empty -> Floating OU. | Medium | Cross reference OUNames (shared with ad-create-user); prefer locating the account by employeeID search. |
| BR-002 | Department from HCM | When the employee email is supplied, AD department = HCM Department Name. | Medium | Process Call hcm-otbi-sql-query then op-ldap-modify. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateADUser/processupdateaduser_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processupdateaduser_client_ep.wsdl |
| outbound LDAP | JNDI eis/ldap/TestConnection (host not in export) | bind account configured in WebLogic | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/Adapters/UpdateUser_ldap.jca |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/composite.xml |
| outbound SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/composite.xml |
| LDAP modify (UpdateUser) | JNDI eis/ldap/TestConnection; modifyRequest with dn and modifyAttributes department, displayName, pwdLastSet (schema); only department and displayName are set. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/Adapters/UpdateUser_ldap.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| ad-create-user | Shared AD LDAP connection and operations | synchronous |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| other | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/BPEL/ProcessUpdateADUser.bpel | Oracle XPath: oraext:parseEscapedXML, dvm:lookupValue, oraext:get-content-as-string. | OTBI sub-process output profile and cross reference lookup. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll sends a notification to EmailDL itservicedesk; no callback, no result. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| OUNames.dvm | dvm | yes |
| EmailDL.dvm | dvm | yes |
| oramds:/apps/Provider/AD/processupdateaduser_client_ep.wsdl | mds | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| TaskUs Active Directory | LDAP (LDAPS to confirm) | eis/ldap/TestConnection (host not in export) | TaskUs IT |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Service endpoint | http://129.144.156.28:[redacted] | [redacted].wsdl |

## 14. Boomi Migration Readiness Summary

This service changes an employee's department (and, if asked, display name) on their Active Directory account. The assignment-change job calls it when an employee moves to a different organisation in HCM.

It is designed to look the employee up in HCM, take their current department, work out which part of the directory their project belongs to, and update the account there. In practice the only active caller does not send the employee's email address, and the HCM lookup only runs when an email address is present. So on the live path the service has no department or project to work with: it would address the account in the general "Floating" area with an empty department, which most likely fails or changes nothing. The caller does not wait for an answer, so nobody would notice.

There are further mismatches. The caller builds the account name from initials and employee number, but new accounts are created with the person's full name ("First Last"), so the address will not match accounts created by ad-create-user. US projects are not in the table used here, so US staff always resolve to the Floating area. Success emails go to one hardcoded person and use a format the notification service cannot read. In Boomi this becomes a small sub-process, but the business needs to confirm what should actually be updated and how the account is found.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | AD User Attributes UPDATE | proc-ad-update-user | process | op-ldap-modify, proc-sub-otbi-query, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/BPEL/ProcessUpdateADUser.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/Adapters/UpdateUser_ldap.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/Schemas/ProcessUpdateADUser.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateADUser/SOA/Schemas/UpdateUser.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processupdateaduser_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/OUNames.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/BPEL/ProcessEmpAssignmentReq_Sch.bpel

## Open questions

### Still outstanding

REVIEW-NEEDED: When an employee changes organisation in HCM, which details on their AD account should change: department only, department and display name, or also the OU and groups? (The live caller sends no department or display name, so it is unclear what the update is meant to achieve; OU moves are handled separately by ad-update-ou.)
  - If this is not answered: Update the department from HCM only.

REVIEW-NEEDED: US users get no OU because the OU table has no US projects, so their account address falls back to the Floating area and the update will likely fail. How should US staff be handled? (This interface uses only the Philippine OU table.)
  - If this is not answered: Find the account by employee ID instead of building its address, so US and PH staff are handled the same way.

REVIEW-NEEDED: Who should be told when an AD update succeeds or fails? (Success notices go to one named person and failure notices are formatted so they are probably not delivered.)
  - If this is not answered: No success notices; failures to the "itservicedesk" distribution list.

