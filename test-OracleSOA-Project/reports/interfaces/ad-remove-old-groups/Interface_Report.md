# ProcessRemoveOldADGroups

Test-OracleSOA-Project · Listener · simple complexity

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

Removes an employee's AD account from every department group (groups under OU=Departments) except the new one, after a project change. It reads the account's memberOf list by sAMAccountName and deletes the user's DN from each old department group.

| Field | Value |
| --- | --- |
| Interface | ProcessRemoveOldADGroups |
| Type | Listener |
| Complexity | simple |
| Components to build | 1 |
| Mapping rules | 3 |
| Business rules | 1 |

## Interface Summary

When an agent moves to a new project, this helper takes them out of the department groups of their old project. It looks the person up in Active Directory, reads the list of groups they belong to, and for every department group other than the new one, removes them from it. Groups outside the "Departments" area (for example Wi-Fi or special groups) are left alone.  The logic is simple, but it is built fragilely: it cuts the group list out of the directory response as text and needs a small Java step to strip prefixes. It has no error handling and never returns a result, and its caller does not wait for it, so if it fails the person keeps their old access and nobody is told. In Boomi it becomes a small sub-process with a proper loop, error handling and a result.

**Complexity:** simple (score 20)

## Functional Analysis

1. **Receive request** — Async SOAP process (createInstance) with sAMAccountName, dnValue, newGroup.
2. **Read group memberships** — LDAP search by samAccountName, ReturnAttributes cn,memberOf.
3. **Extract group list** — Response serialised to text, "tns:" removed by Java, memberOf element cut out and parsed (GroupValues.xsd).
4. **Loop over groups** — Sequential forEach over memberOf/value.
5. **Remove from old department group** — If the group DN contains "OU=Departments" and is not newGroup: LDAP modify on the group, member delete dnValue.
6. **Callback** — processResponse with an empty result.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessRemoveOldADGroups/processremoveoldadgroups\_client\_ep | none declared |
| outbound | LDAP | JNDI eis/ldap/TestConnection (host not in export) | bind account configured in WebLogic |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Old group to member-delete request | dn | memberOf/value\[i\] | looped | contains "OU=Departments" and != newGroup | — | low | — |
| MR-002 | Old group to member-delete request | modifyAttributes/member/method | (none) | hardcoded | — | "delete" | low | — |
| MR-003 | Old group to member-delete request | modifyAttributes/member/attributeValue/value | dnValue | passthrough | — | — | low | User DN after the move. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Which groups are removed | Every group whose DN contains "OU=Departments" and is not the new group. | High |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive request** — From ad-update-ou
2. **Read user groups** — LDAP search memberOf
3. **Next group** — Sequential loop
4. **Old department group?** — OU=Departments and not newGroup
5. **Remove user from group** — member delete
6. **More groups?**
7. **Unhandled fault** — No fault handler
8. **Callback (empty result)**

## Data Model

_This interface owns only its process component; the inputs are a small inline schema and the LDAP shapes come from the shared connector operations._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback | Sub-process start + Return Documents |
| GetADGroups search + Java string clean-up | op-ldap-search (ad-create-user) + profile split |
| forEach + RemoveADUserFromGroup modify | Data split + Decision + op-ldap-modify (ad-create-user) |

**Components**

| Component | Role |
| --- | --- |
| proc-ad-remove-old-groups | Main sub-process |

## Client questions

- **Q-001** When an agent changes project, should they be removed from all department groups of earlier projects, or only from the group of the project they just left?
  **If this is not answered:** Remove all other department groups (as today).
- **Q-002** Who should be told if removing old group access fails?
  **If this is not answered:** Report failures to the "notification" distribution list via the calling OU-move process.
