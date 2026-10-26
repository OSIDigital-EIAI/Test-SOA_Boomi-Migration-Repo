# ProcessUpdateOUADUser

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

Moves an employee's Active Directory account between organisational units (OUs) according to the requested action: "Inactive" parks the account in Pending Termination (temporary, e.g. suspension or leave) or Disabled Accounts (permanent, termination); "Active" brings it back from Pending Termination to the OU of the worker's project; "Change" moves it to the new project's OU after a campaign change and updates department, manager, title and department-group membership.

| Field | Value |
| --- | --- |
| Interface | ProcessUpdateOUADUser |
| Type | Listener |
| Complexity | complex |
| Components to build | 4 |
| Mapping rules | 13 |
| Business rules | 5 |

## Interface Summary

This service moves employee accounts around Active Directory when their status or project changes in HCM. When someone is suspended, goes on leave or leaves, it moves the account to a "Pending Termination" area (temporary) or "Disabled Accounts" (permanent). When they come back it moves the account from Pending Termination back to their project's area. When their project (campaign) changes it moves the account to the new project's area, updates department, manager and job title, adds the new department group and then asks a helper service (ad-remove-old-groups) to take the person out of their old department groups.  The area for each project comes from the same lookup tables used when accounts are created. The Change path finds the account by employee ID, which is reliable. The Active and Inactive paths instead build the account's full directory address from a name passed by the caller (initials plus employee number) and the expected current area. That only works if the account really is named that way and really is where the service expects it. Accounts created by ad-create-user are named "First Last", so this needs to be confirmed against the real directory.  The source also has confirmed defects. US staff never get a project area, because the US branch writes its result to a field that is never read. When no project area is found, the generated move requests are malformed. The Change path does nothing if the new manager cannot be found in AD. These need business decisions before rebuilding, and the Boomi version should look accounts up by employee ID on every path.

**Complexity:** complex (score 62)

## Functional Analysis

1. **Receive request** — Async SOAP process (createInstance) with CN, Employee\_ID, Employee\_Email, Action, Status, OU.
2. **Read HCM details** — If Employee\_ID is not empty: OTBI on "Workforce Management - Worker Assignment Real Time" for the person number, latest assignment first.
3. **Derive OU and department** — Country US: US\_OUNames / US\_ADGroupDNs written to a non-existent input path (OU stays empty). Otherwise OUNames(project ID) into OU\_Variable, department from ADGroupDNs (A-), ADGroupDNs\_TLS (SA-) or ADGroupDNs\_Others.
4. **Route on Action** — "Active", "Change", anything else = Inactive.
5. **Active: move back** — XForm\_Active\_AD: dn CN=\<CN\>,OU=Pending Termination,OU=Users,OU=TaskUs,DC=taskus,DC=prv; newSuperior = project OU; LDAP modifyDN; email EmailDL "suspended".
6. **Inactive: park account** — XForm\_Inactive\_AD: dn CN=\<CN\>,\<project OU\>; newSuperior OU=Pending Termination,OU=Users,OU=TaskUs,DC=taskus,DC=prv (Temporary) or OU=Disabled Accounts,OU=Users,OU=TaskUs,DC=taskus,DC=prv (Permanent); LDAP modifyDN; email EmailDL "suspended".
7. **Change: find user** — Search (&(objectClass=\*)(employeeID=\<CN from 3rd character\>)) under OU=Users,OU=TaskUs,DC=taskus,DC=prv. Continue only if found, OU not empty and project ID not empty.
8. **Change: find manager** — Search sAMAccountName = initials of Manager Name ("Last, First") + Manager Person Number. If not found: Failure@Manager AD not found.
9. **Change: move and update** — modifyDN found dn to the new OU; modify department (grade group table), manager DN, title = Job Name.
10. **Change: groups** — GroupDN from ADGroupDNs\_TLS / ADGroupDNs / ADGroupDNs\_Others; if found add member, then call ProcessRemoveOldADGroups (sAMAccountName = CN, dnValue = new DN, newGroup = GroupDN) without waiting; else Success@No Group Membership mappings Available.
11. **Return result** — Callback with result "Success@..." or "Failure@..." and Description = project description.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateOUADUser/processupdateouaduser\_client\_ep | none declared |
| outbound | LDAP | JNDI eis/ldap/TestConnection (host not in export) | bind account configured in WebLogic |
| outbound | SOAP async (composite) | http://129.144.156.28:80/soa-infra/services/default/ProcessRemoveOldADGroups/processremoveoldadgroups\_client\_ep | none declared |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared |
| outbound | SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification\_client\_ep.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Request to modifyDN (Active) | dn | CN | calculated | — | "CN=" + CN + ",OU=Pending Termination,OU=Users,OU=TaskUs,DC=taskus,DC=prv" | low | — |
| MR-002 | Request to modifyDN (Active) | newRdn | CN, OU/Value | conditional | OU not empty | "CN=" + CN + "," + OU; else "CN=" + CN + ",OU=Floating,OU=Operations,OU=Users,OU=TaskUs,DC=taskus,DC=prv" | medium | A full DN is placed in newRdn; Boomi should send "CN=\<CN\>" only. |
| MR-003 | Request to modifyDN (Active) | newSuperior | OU/Value | conditional | OU not empty | OU; when empty the source writes a second newRdn instead (defect) | medium | Intended fallback is the Floating OU. |
| MR-004 | Request to modifyDN (Inactive) | dn | CN, OU/Value | conditional | OU not empty | "CN=" + CN + "," + OU; when empty no dn is written (defect) | medium | Assumes the account is in its project OU. |
| MR-005 | Request to modifyDN (Inactive) | newRdn | CN, Status | conditional | Status | Temporary: "CN=" + CN + ",OU=Pending Termination,OU=Users,OU=TaskUs,DC=taskus,DC=prv"; Permanent: "CN=" + CN + ",OU=Disabled Accounts,OU=Users,OU=TaskUs,DC=taskus,DC=prv" | medium | Full DN in newRdn. |
| MR-006 | Request to modifyDN (Inactive) | newSuperior | Status | conditional | Status | Temporary: OU=Pending Termination,OU=Users,OU=TaskUs,DC=taskus,DC=prv; Permanent: OU=Disabled Accounts,OU=Users,OU=TaskUs,DC=taskus,DC=prv | low | Other Status values produce no target. |
| MR-007 | Change: user attribute update | dn | Search response dn, OU | calculated | — | First RDN of the found dn + "," + new OU | medium | — |
| MR-008 | Change: user attribute update | department | OTBI Column4 (project ID), Column11 (grade) | conditional | Grade SA- / A- / other | Department from ADGroupDNs\_TLS / ADGroupDNs / ADGroupDNs\_Others, replace | medium | — |
| MR-009 | Change: user attribute update | manager | Manager search response dn | transformed | Manager found | replace | medium | — |
| MR-010 | Change: user attribute update | title | OTBI Column9 (Job Name) | passthrough | — | replace | low | — |
| MR-011 | Change: user attribute update | group member (on GroupDN) | New user DN | conditional | GroupDN not empty | add | low | — |
| MR-012 | Outcome to response | result | (outcome) | conditional | — | "Success@\<text\>" or "Failure@\<text\>" | low | Active/Inactive success leaves result empty. |
| MR-013 | Outcome to response | Description | OTBI Column10 (project description) | passthrough | — | — | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Inactive destination | Status Temporary -\> Pending Termination; Permanent -\> Disabled Accounts. | High |
| BR-002 | Active destination | From Pending Termination to the project OU (OUNames by project ID), else Floating OU. | High |
| BR-003 | Change prerequisites | Move only if the account exists, the new project has an OU, the project ID is known and the new manager exists in AD. | High |
| BR-004 | Department group swap | Add the new project's department group, then remove all other groups under OU=Departments. | High |
| BR-005 | Result convention | result = "Success@..." or "Failure@..." with project description. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive OU request** — From four scheduled jobs
2. **Read HCM details** — OTBI, if Employee\_ID given
3. **Derive OU and department** — Country / grade / project tables
4. **Which action?** — Active, Change, else Inactive
5. **Move to project OU** — modifyDN from Pending Termination
6. **Move to parking OU** — Temporary: Pending Termination; Permanent: Disabled
7. **Find user by employeeID** — Change path
8. **Email move notice** — EmailDL suspended
9. **User, OU, project found?**
10. **Find manager in AD** — sAMAccountName of manager
11. **Manager found?**
12. **Move and update user** — modifyDN, department, manager, title
13. **Add new group** — If GroupDN mapped
14. **Notify error, return Failure** — EmailDL notification
15. **Remove old groups** — ad-remove-old-groups, not awaited
16. **Return result** — Success@ / Failure@ by callback

## Data Model

_This interface owns no profile components; request/response are small inline schemas and LDAP shapes come from the connector operations._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback | Sub-process start + Return Documents |
| HCMExtractSyncProv | Process Call hcm-otbi-sql-query |
| MoveADUser (modifyDN) | op-ldap-modifydn |
| SearchADUser / ModifyADUser | op-ldap-search / op-ldap-modify (ad-create-user) |
| XForm\_Active\_AD.xsl / XForm\_Inactive\_AD.xsl | map-ouad-active / map-ouad-inactive |
| ProcessRemoveOldADGroups (async, not awaited) | Process Call ad-remove-old-groups (synchronous) |
| DVMs | Cross Reference Tables (shared with ad-create-user) |
| catchAll | Try/Catch + util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| op-ldap-modifydn | Move/rename AD entry |
| map-ouad-active | Active move request |
| map-ouad-inactive | Inactive move request |
| proc-ad-update-ou | Main sub-process |

## Client questions

- **Q-001** How should US employees be handled when they are suspended, return or change project, given that today they never get a project area in AD?
  **If this is not answered:** Use the US OU and group tables for US employees, the same way account creation does.
- **Q-002** When a project has no AD area configured, should the account go to the general "Floating" area or should the move stop and be reported?
  **If this is not answered:** Stop and report to the notification list.
- **Q-003** Are TaskUs AD accounts named by full name ("First Last") or by initials plus employee number?
  **If this is not answered:** Find every account by employee ID instead of building its address.
- **Q-004** When an agent changes project but their new manager has no AD account yet, should the move still happen?
  **If this is not answered:** Move the account and update department and groups, leave the manager unchanged and report it.
- **Q-005** Who should receive the "user moved" notices and the error notices?
  **If this is not answered:** Keep the "suspended" and "notification" lists.
