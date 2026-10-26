# ProcessCreateGSuitUser

Test-OracleSOA-Project · Listener · medium complexity

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

Creates a Google Workspace account for a new employee (Admin SDK users.insert) with name, work email, address, organisation, phone and the employee ID as external ID, and, when a preferred name is supplied, adds the alias \<PreferredName\>.\<FamilyName\>@taskus.com. It returns Success with the work email and alias, or the error text.

| Field | Value |
| --- | --- |
| Interface | ProcessCreateGSuitUser |
| Type | Listener |
| Complexity | medium |
| Components to build | 6 |
| Mapping rules | 15 |
| Business rules | 2 |

## Interface Summary

This service opens a Google Workspace account for each new employee. It gets an access key from Google, creates the user with their name, company email, work address, job title, department, phone and employee ID, and makes them visible in the company address book. If the employee has a preferred name it also creates an extra address in the form preferredname.surname@taskus.com.  The main concern is the password. In production every new Google account is created with the same fixed password written into the integration code, and the user is not forced to change it at first sign-in. This is the same password that the AD account creation uses and that appears in the new-hire reports. This must be replaced with a secure process. The older version of the service took the password from the caller instead.  Other points: all calls go to Google through a custom Java relay over plain HTTP, carrying the access key; Google errors such as "insufficient licences" come back as normal replies, so the special service-desk alerts written for them only fire when the relay itself fails; and the relay sends names in the ISO-8859-1 character set, which can damage names with characters outside it. In Boomi the process calls Google directly through the shared Google connection.

**Complexity:** medium (score 30)

## Functional Analysis

1. **Receive request** — Async SOAP process with primaryEmail, employeeID, name (givenName, familyName, AliasName), password, emails, addresses, organizations, phones.
2. **Get access token** — Synchronous call to ProcessGetGSuitAccessToken.
3. **Build user** — xfrom\_HCM\_GSuite: user JSON plus authorization field (see MAP-001).
4. **Create user** — Java relay createGsuiteUser -\> Google users.insert.
5. **User created?** — Reply primaryEmail not empty; else result = reply error.
6. **Create alias** — If AliasName not empty: createGsuiteAlias with email, authorization and alias \<AliasName\>.\<familyName\>@taskus.com.
7. **Return result** — Callback result Success, workemail, alias (or error text).

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | processcreategsuituserprocess\_client\_ep | none declared |
| outbound | REST over plain HTTP | http://129.144.156.139:9073/google-restservice-context-root/resources/googleServices/createGsuiteUser | Google bearer token passed in the JSON body |
| outbound | REST over plain HTTP | http://129.144.156.139:9073/google-restservice-context-root/resources/googleServices/createGsuiteAlias | Google bearer token passed in the JSON body |
| outbound | REST over HTTPS (from the relay) | https://www.googleapis.com/admin/directory/v1/users | OAuth 2.0 bearer |
| outbound | SOAP (composite) | oramds:/apps/Provider/GSuit/ProcessGetGSuitAccessToken.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Create request to Google user | primaryEmail | primaryEmail | passthrough | — | — | low | — |
| MR-002 | Create request to Google user | name.givenName / name.familyName | name/givenName, name/familyName | passthrough | — | — | low | — |
| MR-003 | Create request to Google user | suspended / ipWhitelisted | (none) | hardcoded | — | false | low | — |
| MR-004 | Create request to Google user | password | (none) | hardcoded | — | \[redacted literal\] | low | Must be replaced per security decision. |
| MR-005 | Create request to Google user | changePasswordAtNextLogin | (none) | hardcoded | — | false | low | — |
| MR-006 | Create request to Google user | ims\[work, gtalk, primary\] | primaryEmail | passthrough | — | — | low | gtalk is obsolete. |
| MR-007 | Create request to Google user | emails.address | emails/address | passthrough | — | — | low | — |
| MR-008 | Create request to Google user | addresses\[work\] streetAddress, locality, region, postalCode | addresses/\* | passthrough | — | — | low | — |
| MR-009 | Create request to Google user | externalIds\[organization\] | employeeID | passthrough | — | — | low | — |
| MR-010 | Create request to Google user | relations | (none) | hardcoded | — | value "manager", type "referred\_by" | low | Constant placeholder, not the real manager. |
| MR-011 | Create request to Google user | organizations\[work, primary\] name, title, description | organizations/\* | passthrough | — | — | low | — |
| MR-012 | Create request to Google user | phones\[work\] | phones/value | conditional | not empty | — | low | — |
| MR-013 | Create request to Google user | orgUnitPath / includeInGlobalAddressList | (none) | hardcoded | — | "/" / true | low | — |
| MR-014 | Alias request | alias | name/AliasName, name/familyName | calculated | AliasName not empty | AliasName + "." + familyName (spaces removed) + "@taskus.com" | low | Inventory notes lower-casing for Boomi; legacy does not lower-case. |
| MR-015 | Result | result / workemail / alias | user reply primaryEmail, alias reply alias, error | conditional | primaryEmail not empty | "Success" + email (+ alias) else error text | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Preferred-name alias | If a preferred name exists, add alias \<PreferredName\>.\<FamilyName\>@taskus.com. | Medium |
| BR-002 | Licence error routing | "Insufficient licenses" and "Domain user limit reached" go to the IT service desk list; other errors to the notification list. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive new user request** — From gsuite-user-lifecycle
2. **Get Google access token** — gsuite-get-access-token
3. **Create Google user** — Via Java relay
4. **User created?** — primaryEmail returned
5. **Notify by error type** — Licences -\> service desk; else notification
6. **Preferred name given?** — AliasName
7. **Create alias** — \<Alias\>.\<family\>@taskus.com
8. **Return result** — Success/email/alias or error

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| Google Directory User JSON Prof | json | prof-google-user-json |
| Google Directory Alias JSON Prof | json | prof-google-alias-json |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback | Sub-process start + Return Documents |
| GetGSuitAccessToken call | (none) OAuth on conn-google-directory |
| xfrom\_HCM\_GSuite.xsl | map-create-req-to-google-user |
| Java relay createGsuiteUser / createGsuiteAlias | op-google-user-insert / op-google-alias-insert |
| catchAll text routing | Try/Catch + util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| op-google-user-insert | POST /users |
| op-google-alias-insert | POST /users/{userKey}/aliases |
| prof-google-user-json | User resource profile |
| prof-google-alias-json | Alias profile |
| map-create-req-to-google-user | Request map |
| proc-sub-gsuite-create-user | Main sub-process |

## Client questions

- **Q-001** How should a new employee's first Google password be set, and must they change it at first sign-in?
  **If this is not answered:** Unique random password per account with change forced at first sign-in, delivered through the process agreed for AD; no password in emails.
- **Q-002** Should the alias be all lower case, and what should happen if the alias address is already taken?
  **If this is not answered:** Lower-case the alias and report clashes to the notification list.
- **Q-003** Who should be alerted when Google refuses a new account because licences have run out?
  **If this is not answered:** The "itservicedesk" distribution list.
- **Q-004** Should new Google accounts be placed in a specific organisational unit rather than the top level?
  **If this is not answered:** Keep the root unit.
