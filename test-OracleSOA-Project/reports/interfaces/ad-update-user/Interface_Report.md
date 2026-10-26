# ProcessUpdateADUser

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

Updates an existing Active Directory account's department and display name. The caller gives the account's common name (CN), the employee ID and optionally the new values; when an employee email is supplied the service reads the worker's department, project and grade from Oracle HCM, finds the OU for the project and modifies the entry at CN,\<OU\>.

| Field | Value |
| --- | --- |
| Interface | ProcessUpdateADUser |
| Type | Listener |
| Complexity | simple |
| Components to build | 1 |
| Mapping rules | 5 |
| Business rules | 2 |

## Interface Summary

This service changes an employee's department (and, if asked, display name) on their Active Directory account. The assignment-change job calls it when an employee moves to a different organisation in HCM.  It is designed to look the employee up in HCM, take their current department, work out which part of the directory their project belongs to, and update the account there. In practice the only active caller does not send the employee's email address, and the HCM lookup only runs when an email address is present. So on the live path the service has no department or project to work with: it would address the account in the general "Floating" area with an empty department, which most likely fails or changes nothing. The caller does not wait for an answer, so nobody would notice.  There are further mismatches. The caller builds the account name from initials and employee number, but new accounts are created with the person's full name ("First Last"), so the address will not match accounts created by ad-create-user. US projects are not in the table used here, so US staff always resolve to the Floating area. Success emails go to one hardcoded person and use a format the notification service cannot read. In Boomi this becomes a small sub-process, but the business needs to confirm what should actually be updated and how the account is found.

**Complexity:** simple (score 24)

## Functional Analysis

1. **Receive request** — Async SOAP process operation (createInstance) with the ProcessUpdateADUser.xsd process element.
2. **Read HCM details** — Only if Employee\_Email is not empty: OTBI by Person Number = Employee\_ID; parseEscapedXML; Department := Column1.
3. **Find OU** — Grade check on Column5 (SA-, A-, other) – all branches: OU = OUNames.dvm ProjectID(Column4) -\> OU, default empty.
4. **Build DN and changes** — dn = DN + "," + OU, or the Floating OU when empty; department replace with Department; displayName method and value from the input.
5. **Modify AD entry** — LDAP modify through UpdateUser (eis/ldap/TestConnection).
6. **Notify success** — XXSendNotification to a hardcoded individual; Subject "User \<CN\> Updated Successfully in \<DN\> \#EmailID \#EID \#Department"; Content "User successfully updated".
7. **Callback** — processResponse with an empty outputVariable (result never set).

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateADUser/processupdateaduser\_client\_ep | none declared |
| outbound | LDAP | JNDI eis/ldap/TestConnection (host not in export) | bind account configured in WebLogic |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared |
| outbound | SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification\_client\_ep.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Update request to LDAP modify | dn | DN, OTBI Column4 (project ID) | conditional | OU found | DN + "," + OUNames(project ID); else DN + ",OU=Floating,OU=Operations,OU=Users,OU=TaskUs,DC=taskus,DC=prv" | medium | — |
| MR-002 | Update request to LDAP modify | modifyAttributes/department/method | (none) | hardcoded | — | "replace" | low | Department\_Method from the input is ignored. |
| MR-003 | Update request to LDAP modify | modifyAttributes/department/attributeValue/value | Department, overwritten by OTBI Column1 | conditional | OTBI runs only when Employee\_Email is not empty | — | low | — |
| MR-004 | Update request to LDAP modify | modifyAttributes/displayName/method | DisplayName\_Method | passthrough | — | — | low | — |
| MR-005 | Update request to LDAP modify | modifyAttributes/displayName/attributeValue/value | DisplayName | passthrough | — | — | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | OU from project | OU = OUNames(project ID) for every grade; empty -\> Floating OU. | Medium |
| BR-002 | Department from HCM | When the employee email is supplied, AD department = HCM Department Name. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive update request** — From emp-assignment-sch
2. **Email supplied?** — Employee\_Email != ""
3. **Read HCM department** — OTBI by person number
4. **Look up OU** — OUNames by project ID
5. **OU found?**
6. **DN under project OU**
7. **DN under Floating OU**
8. **Modify AD entry** — department, displayName
9. **Notify, callback** — Hardcoded recipient; empty result
10. **Notify service desk** — catchAll; no callback

## Data Model

_This interface owns only its process component; the request shape is a small inline schema and the LDAP shapes come from the shared connector operations of ad-create-user._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback (async SOAP) | Sub-process start + Return Documents |
| HCMExtractSyncProv | Process Call hcm-otbi-sql-query |
| OUNames DVM | Cross reference OUNames (from ad-create-user) |
| UpdateUser LDAP adapter | op-ldap-modify (ad-create-user) |
| catchAll + XXSendNotification | Try/Catch + util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| proc-ad-update-user | Main sub-process |

## Client questions

- **Q-001** When an employee changes organisation in HCM, which details on their AD account should change: department only, department and display name, or also the OU and groups?
  **If this is not answered:** Update the department from HCM only.
- **Q-002** US users get no OU because the OU table has no US projects, so their account address falls back to the Floating area and the update will likely fail. How should US staff be handled?
  **If this is not answered:** Find the account by employee ID instead of building its address, so US and PH staff are handled the same way.
- **Q-003** Who should be told when an AD update succeeds or fails?
  **If this is not answered:** No success notices; failures to the "itservicedesk" distribution list.
