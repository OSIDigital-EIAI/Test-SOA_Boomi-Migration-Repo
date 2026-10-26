# Migration Plan — ProcessRemoveOldADGroups

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.933Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessRemoveOldADGroups |
| Identifier | ad-remove-old-groups |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | simple (20) |
| Components to build | 1 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/BPEL/ProcessRemoveOldADGroups.bpel |
| Depends on | ad-create-user |

## 2. End-to-End Process Architecture

Async BPEL composite with two LDAP adapter references and a Java embedding; no MDS dependencies other than its own client WSDL used by the caller.

Boomi sub-process "Sub: AD Remove Old Department Groups" called synchronously by ad-update-ou: op-ldap-search (memberOf), split values, Decision, op-ldap-modify member delete, Try/Catch, return a status.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback | Sub-process start + Return Documents | Return a real status. |
| GetADGroups search + Java string clean-up | op-ldap-search (ad-create-user) + profile split |  |
| forEach + RemoveADUserFromGroup modify | Data split + Decision + op-ldap-modify (ad-create-user) |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive request | Async SOAP process (createInstance) with sAMAccountName, dnValue, newGroup. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/Schemas/ProcessRemoveOldADGroups.xsd |
| 2 | Read group memberships | LDAP search by samAccountName, ReturnAttributes cn,memberOf. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/Adapters/GetADGroups_ldap.jca |
| 3 | Extract group list | Response serialised to text, "tns:" removed by Java, memberOf element cut out and parsed (GroupValues.xsd). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/BPEL/ProcessRemoveOldADGroups.bpel |
| 4 | Loop over groups | Sequential forEach over memberOf/value. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/BPEL/ProcessRemoveOldADGroups.bpel |
| 5 | Remove from old department group | If the group DN contains "OU=Departments" and is not newGroup: LDAP modify on the group, member delete dnValue. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/Adapters/RemoveADUserFromGroup_ldap.jca |
| 6 | Callback | processResponse with an empty result. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/BPEL/ProcessRemoveOldADGroups.bpel |

## 4. Payload Profiles

> **Not available.** This interface owns only its process component; the inputs are a small inline schema and the LDAP shapes come from the shared connector operations.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | dn | memberOf/value[i] | looped | contains "OU=Departments" and != newGroup |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/BPEL/ProcessRemoveOldADGroups.bpel | Old group to member-delete request |
| MR-002 | modifyAttributes/member/method | (none) | hardcoded |  | "delete" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/BPEL/ProcessRemoveOldADGroups.bpel | Old group to member-delete request |
| MR-003 | modifyAttributes/member/attributeValue/value | dnValue | passthrough |  |  | low | User DN after the move. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/BPEL/ProcessRemoveOldADGroups.bpel | Old group to member-delete request |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Which groups are removed | Every group whose DN contains "OU=Departments" and is not the new group. | High | Split memberOf, Decision, op-ldap-modify. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessRemoveOldADGroups/processremoveoldadgroups_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processremoveoldadgroups_client_ep.wsdl |
| outbound LDAP | JNDI eis/ldap/TestConnection (host not in export) | bind account configured in WebLogic | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/Adapters/GetADGroups_ldap.jca |
| LDAP search (GetADGroups) | JNDI eis/ldap/TestConnection; wholeSubtree; ReturnAttributes cn,memberOf; SizeLimit 1000; TimeLimit 0; FollowReferrals ignore; AliasDereferencing never. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/Adapters/GetADGroups_ldap.jca |
| LDAP modify (RemoveADUserFromGroup) | JNDI eis/ldap/TestConnection; LdapModifyInteractionSpec; member delete on the group DN. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/Adapters/RemoveADUserFromGroup_ldap.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| ad-create-user | Shared AD LDAP connection, search and modify operations | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/BPEL/ProcessRemoveOldADGroups.bpel | bpelx:exec Replace_Code: remove "tns:" prefixes from the serialised search response. | Not needed: read memberOf from the LDAP response profile. | Low |
| other | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/BPEL/ProcessRemoveOldADGroups.bpel | oraext:get-content-as-string and oraext:parseXML with substring-before/after to isolate memberOf. | Profile-based split of memberOf values. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | None (no faultHandlers). |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/AD/processremoveoldadgroups_client_ep.wsdl | mds | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| TaskUs Active Directory | LDAP (LDAPS to confirm) | eis/ldap/TestConnection (host not in export) | TaskUs IT |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Service endpoint | http://129.144.156.28:[redacted] | [redacted].wsdl |

## 14. Boomi Migration Readiness Summary

When an agent moves to a new project, this helper takes them out of the department groups of their old project. It looks the person up in Active Directory, reads the list of groups they belong to, and for every department group other than the new one, removes them from it. Groups outside the "Departments" area (for example Wi-Fi or special groups) are left alone.

The logic is simple, but it is built fragilely: it cuts the group list out of the directory response as text and needs a small Java step to strip prefixes. It has no error handling and never returns a result, and its caller does not wait for it, so if it fails the person keeps their old access and nobody is told. In Boomi it becomes a small sub-process with a proper loop, error handling and a result.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Sub: AD Remove Old Department Groups | proc-ad-remove-old-groups | process | op-ldap-search, op-ldap-modify |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/BPEL/ProcessRemoveOldADGroups.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/Adapters/GetADGroups_ldap.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/Adapters/RemoveADUserFromGroup_ldap.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/Schemas/ProcessRemoveOldADGroups.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/Schemas/GroupValues.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/WSDLs/ProcessRemoveOldADGroups.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessRemoveOldADGroups/SOA/Transformations/xform_memberOf.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processremoveoldadgroups_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateOUADUser/SOA/BPEL/ProcessUpdateOUADUser.bpel

## Open questions

### Still outstanding

REVIEW-NEEDED: When an agent changes project, should they be removed from all department groups of earlier projects, or only from the group of the project they just left? (Today every group under "Departments" other than the new one is removed, which also removes any deliberately granted extra department groups.)
  - If this is not answered: Remove all other department groups (as today).

REVIEW-NEEDED: Who should be told if removing old group access fails? (Failures are silent today, so old project access can remain.)
  - If this is not answered: Report failures to the "notification" distribution list via the calling OU-move process.

