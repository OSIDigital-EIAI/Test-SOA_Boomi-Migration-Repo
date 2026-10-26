# Migration Plan — ProcessUpdateOUADUser

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.918Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessUpdateOUADUser |
| Identifier | ad-update-ou |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | complex (62) |
| Components to build | 4 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel |
| Depends on | ad-create-user, ad-remove-old-groups, hcm-otbi-sql-query, util-send-notification |

## 2. End-to-End Process Architecture

Async BPEL composite with three LDAP adapter references (modifyDN, search, modify), ProcessHCMExtractSyncProv and XXSendNotification via MDS WSDLs, an async reference to ProcessRemoveOldADGroups (callback not awaited) and six MDS DVMs. Returns result/Description by WS-Addressing callback.

Boomi sub-process "AD User OU MOVE" called synchronously by the four jobs: Process Call hcm-otbi-sql-query; cross reference lookups; locate the account with op-ldap-search by employeeID on every path; Route on Action to map-ouad-active / map-ouad-inactive or the Change branch; op-ldap-modifydn (or a Groovy JNDI rename if the connector cannot move entries); op-ldap-modify; Process Call ad-remove-old-groups; return result/Description; Try/Catch with util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback | Sub-process start + Return Documents | Keep Success@/Failure@ convention. |
| HCMExtractSyncProv | Process Call hcm-otbi-sql-query |  |
| MoveADUser (modifyDN) | op-ldap-modifydn | Send RDN and newSuperior correctly. |
| SearchADUser / ModifyADUser | op-ldap-search / op-ldap-modify (ad-create-user) |  |
| XForm_Active_AD.xsl / XForm_Inactive_AD.xsl | map-ouad-active / map-ouad-inactive | Fix the empty-OU branches. |
| ProcessRemoveOldADGroups (async, not awaited) | Process Call ad-remove-old-groups (synchronous) | Failures become visible. |
| DVMs | Cross Reference Tables (shared with ad-create-user) |  |
| catchAll | Try/Catch + util-send-notification |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive request | Async SOAP process (createInstance) with CN, Employee_ID, Employee_Email, Action, Status, OU. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Schemas/ProcessUpdateOUADUser.xsd |
| 2 | Read HCM details | If Employee_ID is not empty: OTBI on "Workforce Management - Worker Assignment Real Time" for the person number, latest assignment first. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel |
| 3 | Derive OU and department | Country US: US_OUNames / US_ADGroupDNs written to a non-existent input path (OU stays empty). Otherwise OUNames(project ID) into OU_Variable, department from ADGroupDNs (A-), ADGroupDNs_TLS (SA-) or ADGroupDNs_Others. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel |
| 4 | Route on Action | "Active", "Change", anything else = Inactive. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel |
| 5 | Active: move back | XForm_Active_AD: dn CN=<CN>,OU=Pending Termination,OU=Users,OU=TaskUs,DC=taskus,DC=prv; newSuperior = project OU; LDAP modifyDN; email EmailDL "suspended". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/XForm_Active_AD.xsl |
| 6 | Inactive: park account | XForm_Inactive_AD: dn CN=<CN>,<project OU>; newSuperior OU=Pending Termination,OU=Users,OU=TaskUs,DC=taskus,DC=prv (Temporary) or OU=Disabled Accounts,OU=Users,OU=TaskUs,DC=taskus,DC=prv (Permanent); LDAP modifyDN; email EmailDL "suspended". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/XForm_Inactive_AD.xsl |
| 7 | Change: find user | Search (&(objectClass=*)(employeeID=<CN from 3rd character>)) under OU=Users,OU=TaskUs,DC=taskus,DC=prv. Continue only if found, OU not empty and project ID not empty. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/xform_searchADUser.xsl |
| 8 | Change: find manager | Search sAMAccountName = initials of Manager Name ("Last, First") + Manager Person Number. If not found: Failure@Manager AD not found. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel |
| 9 | Change: move and update | modifyDN found dn to the new OU; modify department (grade group table), manager DN, title = Job Name. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel |
| 10 | Change: groups | GroupDN from ADGroupDNs_TLS / ADGroupDNs / ADGroupDNs_Others; if found add member, then call ProcessRemoveOldADGroups (sAMAccountName = CN, dnValue = new DN, newGroup = GroupDN) without waiting; else Success@No Group Membership mappings Available. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel |
| 11 | Return result | Callback with result "Success@..." or "Failure@..." and Description = project description. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel |

## 4. Payload Profiles

> **Not available.** This interface owns no profile components; request/response are small inline schemas and LDAP shapes come from the connector operations.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | dn | CN | calculated |  | "CN=" + CN + ",OU=Pending Termination,OU=Users,OU=TaskUs,DC=taskus,DC=prv" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/XForm_Active_AD.xsl | Request to modifyDN (Active) |
| MR-002 | newRdn | CN, OU/Value | conditional | OU not empty | "CN=" + CN + "," + OU; else "CN=" + CN + ",OU=Floating,OU=Operations,OU=Users,OU=TaskUs,DC=taskus,DC=prv" | medium | A full DN is placed in newRdn; Boomi should send "CN=<CN>" only. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/XForm_Active_AD.xsl | Request to modifyDN (Active) |
| MR-003 | newSuperior | OU/Value | conditional | OU not empty | OU; when empty the source writes a second newRdn instead (defect) | medium | Intended fallback is the Floating OU. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/XForm_Active_AD.xsl | Request to modifyDN (Active) |
| MR-004 | dn | CN, OU/Value | conditional | OU not empty | "CN=" + CN + "," + OU; when empty no dn is written (defect) | medium | Assumes the account is in its project OU. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/XForm_Inactive_AD.xsl | Request to modifyDN (Inactive) |
| MR-005 | newRdn | CN, Status | conditional | Status | Temporary: "CN=" + CN + ",OU=Pending Termination,OU=Users,OU=TaskUs,DC=taskus,DC=prv"; Permanent: "CN=" + CN + ",OU=Disabled Accounts,OU=Users,OU=TaskUs,DC=taskus,DC=prv" | medium | Full DN in newRdn. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/XForm_Inactive_AD.xsl | Request to modifyDN (Inactive) |
| MR-006 | newSuperior | Status | conditional | Status | Temporary: OU=Pending Termination,OU=Users,OU=TaskUs,DC=taskus,DC=prv; Permanent: OU=Disabled Accounts,OU=Users,OU=TaskUs,DC=taskus,DC=prv | low | Other Status values produce no target. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/XForm_Inactive_AD.xsl | Request to modifyDN (Inactive) |
| MR-007 | dn | Search response dn, OU | calculated |  | First RDN of the found dn + "," + new OU | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel | Change: user attribute update |
| MR-008 | department | OTBI Column4 (project ID), Column11 (grade) | conditional | Grade SA- / A- / other | Department from ADGroupDNs_TLS / ADGroupDNs / ADGroupDNs_Others, replace | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/ADGroupDNs_TLS.dvm | Change: user attribute update |
| MR-009 | manager | Manager search response dn | transformed | Manager found | replace | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel | Change: user attribute update |
| MR-010 | title | OTBI Column9 (Job Name) | passthrough |  | replace | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel | Change: user attribute update |
| MR-011 | group member (on GroupDN) | New user DN | conditional | GroupDN not empty | add | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel | Change: user attribute update |
| MR-012 | result | (outcome) | conditional |  | "Success@<text>" or "Failure@<text>" | low | Active/Inactive success leaves result empty. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel | Outcome to response |
| MR-013 | Description | OTBI Column10 (project description) | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel | Outcome to response |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Inactive destination | Status Temporary -> Pending Termination; Permanent -> Disabled Accounts. | High | Decision on Status; map-ouad-inactive; op-ldap-modifydn. |
| BR-002 | Active destination | From Pending Termination to the project OU (OUNames by project ID), else Floating OU. | High | Cross reference OUNames; map-ouad-active. |
| BR-003 | Change prerequisites | Move only if the account exists, the new project has an OU, the project ID is known and the new manager exists in AD. | High | Decision shapes after op-ldap-search. |
| BR-004 | Department group swap | Add the new project's department group, then remove all other groups under OU=Departments. | High | op-ldap-modify + Process Call ad-remove-old-groups. |
| BR-005 | Result convention | result = "Success@..." or "Failure@..." with project description. | Medium | Return document with the same fields. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateOUADUser/processupdateouaduser_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processupdateouaduser_client_ep.wsdl |
| outbound LDAP | JNDI eis/ldap/TestConnection (host not in export) | bind account configured in WebLogic | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Adapters/MoveADUser_ldap.jca |
| outbound SOAP async (composite) | http://129.144.156.28:80/soa-infra/services/default/ProcessRemoveOldADGroups/processremoveoldadgroups_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processremoveoldadgroups_client_ep.wsdl |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/composite.xml |
| outbound SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/composite.xml |
| LDAP modifyDN (MoveADUser) | JNDI eis/ldap/TestConnection; LdapModifyDNInteractionSpec; DeleteOldRDN true; dn, newRdn, newSuperior from XSL/BPEL. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Adapters/MoveADUser_ldap.jca |
| LDAP search (SearchADUser) | JNDI eis/ldap/TestConnection; wholeSubtree; ReturnAttributes cn; SizeLimit 1000; TimeLimit 0; FollowReferrals ignore; AliasDereferencing never. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Adapters/SearchADUser_ldap.jca |
| LDAP modify (ModifyADUser) | JNDI eis/ldap/TestConnection; LdapModifyInteractionSpec; department, manager, title replace; group member add. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Adapters/ModifyADUser_ldap.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| ad-create-user | Shared AD LDAP connection, search and modify operations, OU/group cross references | synchronous |
| ad-remove-old-groups | Sub: AD Remove Old Department Groups | synchronous |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/XForm_Active_AD.xsl | modifyDN request for Active. | map-ouad-active. | Medium |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/XForm_Inactive_AD.xsl | modifyDN request for Inactive (Temporary/Permanent). | map-ouad-inactive. | Medium |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/xform_searchADUser.xsl | Search by employeeID derived from CN. | Parameter on op-ldap-search. | Low |
| other | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel | Oracle XPath: oraext:parseEscapedXML, oraext:get-content-as-string with string slicing to read dn and entry counts, dvm:lookupValue, ora:getFaultAsString. | LDAP response profile, cross reference lookups, Try/Catch message. | Medium |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Process catchAll: email EmailDL notification, set Failure@ result, callback. |
| Retries | None. |
| Compensation | None: a moved account is not moved back if the later modify or group change fails. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| OUNames, US_OUNames, ADGroupDNs, ADGroupDNs_TLS, ADGroupDNs_Others, US_ADGroupDNs | dvm | yes |
| EmailDL.dvm | dvm | yes |
| oramds:/apps/Provider/AD/processupdateouaduser_client_ep.wsdl | mds | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| TaskUs Active Directory | LDAP (LDAPS to confirm) | eis/ldap/TestConnection (host not in export) | TaskUs IT |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Service endpoint | http://129.144.156.28:[redacted] | [redacted].wsdl |

## 14. Boomi Migration Readiness Summary

This service moves employee accounts around Active Directory when their status or project changes in HCM. When someone is suspended, goes on leave or leaves, it moves the account to a "Pending Termination" area (temporary) or "Disabled Accounts" (permanent). When they come back it moves the account from Pending Termination back to their project's area. When their project (campaign) changes it moves the account to the new project's area, updates department, manager and job title, adds the new department group and then asks a helper service (ad-remove-old-groups) to take the person out of their old department groups.

The area for each project comes from the same lookup tables used when accounts are created. The Change path finds the account by employee ID, which is reliable. The Active and Inactive paths instead build the account's full directory address from a name passed by the caller (initials plus employee number) and the expected current area. That only works if the account really is named that way and really is where the service expects it. Accounts created by ad-create-user are named "First Last", so this needs to be confirmed against the real directory.

The source also has confirmed defects. US staff never get a project area, because the US branch writes its result to a field that is never read. When no project area is found, the generated move requests are malformed. The Change path does nothing if the new manager cannot be found in AD. These need business decisions before rebuilding, and the Boomi version should look accounts up by employee ID on every path.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | AD Entry MOVE Op | op-ldap-modifydn | connector-action | conn-ad-ldap |
| 2 | Map \| OU Request to AD Move Active | map-ouad-active | transform.map | — |
| 3 | Map \| OU Request to AD Move Inactive | map-ouad-inactive | transform.map | — |
| 4 | AD User OU MOVE | proc-ad-update-ou | process | op-ldap-search, op-ldap-modify, op-ldap-modifydn, map-ouad-active, map-ouad-inactive, proc-sub-otbi-query, proc-ad-remove-old-groups, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Adapters/MoveADUser_ldap.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Adapters/SearchADUser_ldap.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Adapters/ModifyADUser_ldap.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Schemas/ProcessUpdateOUADUser.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/XForm_Active_AD.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/XForm_Inactive_AD.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/Transformations/xform_searchADUser.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processupdateouaduser_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processremoveoldadgroups_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/OUNames.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpCampaignChange_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmpMaternityLeave_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/Transformations/xform_User_Details.xsl

## Open questions

### Still outstanding

REVIEW-NEEDED: How should US employees be handled when they are suspended, return or change project, given that today they never get a project area in AD? (The US branch writes the OU to a path that does not exist, so US users get no OU. Confirm the intended behaviour for US users and for missing project mappings.)
  - If this is not answered: Use the US OU and group tables for US employees, the same way account creation does.

REVIEW-NEEDED: When a project has no AD area configured, should the account go to the general "Floating" area or should the move stop and be reported? (Today this case produces a malformed request that fails.)
  - If this is not answered: Stop and report to the notification list.

REVIEW-NEEDED: Are TaskUs AD accounts named by full name ("First Last") or by initials plus employee number? (Suspension, leave and return moves build the account address from initials plus employee number; accounts created by the AD creation process use the full name.)
  - If this is not answered: Find every account by employee ID instead of building its address.

REVIEW-NEEDED: When an agent changes project but their new manager has no AD account yet, should the move still happen? (Today nothing is changed and a failure is returned.)
  - If this is not answered: Move the account and update department and groups, leave the manager unchanged and report it.

REVIEW-NEEDED: Who should receive the "user moved" notices and the error notices? (Moves are emailed to the "suspended" list and errors to the "notification" list; confirm these are still right.)
  - If this is not answered: Keep the "suspended" and "notification" lists.

