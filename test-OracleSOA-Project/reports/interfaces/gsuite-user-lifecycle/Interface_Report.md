# ProcessUpdateGSuitUser

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

Single entry point for Google Workspace account lifecycle events. Depending on Operation it creates the account for a new hire (via gsuite-email-generation and gsuite-create-user), updates the user's name (Update), suspends or reactivates the account (Suspend with suspended true/false), or restores a rehire's account with a new password (Rehire). Before acting it reads the worker's department, project, job, location, manager email and country from HCM and looks the user up in Google.

| Field | Value |
| --- | --- |
| Interface | ProcessUpdateGSuitUser |
| Type | Listener |
| Complexity | complex |
| Components to build | 5 |
| Mapping rules | 8 |
| Business rules | 4 |

## Interface Summary

This service is the single place where Google Workspace accounts are changed when something happens to an employee in HCM. The calling job says what to do: create, update the name, suspend or reactivate, or rehire. The service first looks up the employee's current details in HCM and finds their Google account, then acts.  For a new hire it checks that nobody already has the proposed address (and that the person does not already have an account under their employee ID), creates the account, and emails the account details to the manager and HR. For a leaver or suspension it suspends the account and tells the "suspended" list; for reactivation it lifts the suspension. For a rehire it restores the account with a newly generated password and emails that password to a trainers list.  Passwords are the main issue. New-hire emails send the same fixed password used everywhere (see gsuite-create-user and ad-create-user) to the manager and, for US hires, to six named people; rehire emails send the real new password in plain text to a distribution list; generated passwords are also written to the server log and returned to the caller. Other issues: a failed suspension is silently ignored (so a leaver may keep access without anyone knowing); success and failure are told apart by matching error message text; and all Google calls go through the custom Java relay over plain HTTP.  In Boomi this becomes a process with a Route on Operation, calling Google directly through the shared connection, with a secure password-delivery approach and explicit failure handling.

**Complexity:** complex (score 75)

## Functional Analysis

1. **Receive event** — Async SOAP process with Operation and user data.
2. **Read HCM details** — If EID not empty: OTBI by Person Number; Department := s\_1, ProjectID := s\_4, ManagerEmail := s\_8.
3. **Get access token** — ProcessGetGSuitAccessToken.
4. **Look up Google user** — Relay getGSuiteUser by Email; empty primaryEmail -\> fault with relay error text.
5. **Not found (Create/Rehire)** — Fault handler: random password, xform\_create\_gsuit\_user\_1 -\> gsuite-create-user (async); on Success email the manager/HR; set Operation=Skip.
6. **Update** — updateGSuiteUser with givenName/familyName; notify trainers; output from xform\_update\_output.
7. **Suspend / reactivate** — updateGSuiteUser with suspended flag; notify EmailDL suspended.
8. **Rehire** — Random password; updateGSuiteUser with name, password, suspended, changePasswordAtNextLogin from input; notify trainers and pwdtrainers (password in body).
9. **Create (user found)** — If externalIds value != EID: gsuite-email-generation; if Status != Existing create as in step 5; else report existing account.
10. **Callback** — Google user structure with kind/id/emails/title/orgUnitPath etc.; password included.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateGSuitUser/processupdategsuituser\_client\_ep | none declared |
| outbound | REST over plain HTTP | http://129.144.156.139:9073/google-restservice-context-root/resources/googleServices/getGSuiteUser | Google bearer token as parameter |
| outbound | REST over plain HTTP | http://129.144.156.139:9073/google-restservice-context-root/resources/googleServices/updateGSuiteUser | Google bearer token in body |
| outbound | SOAP async (composites) | oramds:/apps/Provider/GSuit/processemailgenerationabcsimplprocess\_client\_ep.wsdl, processcreategsuituserprocess\_client\_ep.wsdl | none declared |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Name update | userKey / primaryEmail | Email | passthrough | Operation = Update | — | low | — |
| MR-002 | Name update | name.givenName / name.familyName | User/name/givenName, familyName | passthrough | — | — | low | — |
| MR-003 | Suspend / reactivate | primaryEmail | Email | passthrough | Operation = Suspend | — | low | — |
| MR-004 | Suspend / reactivate | suspended | User/suspended | passthrough | — | — | low | true = suspend, false = reactivate |
| MR-005 | Rehire restore | primaryEmail | Google user primaryEmail | passthrough | Operation = Rehire | — | low | — |
| MR-006 | Rehire restore | name, suspended, changePasswordAtNextLogin, ipWhitelisted, ims, emails, ... | User/\* | conditional | present in input | — | medium | — |
| MR-007 | Rehire restore | password | generated | calculated | — | random 12 chars | medium | Per security decision. |
| MR-008 | Lifecycle request to create request | primaryEmail, employeeID, name, AliasName, password, emails, addresses, phones | User/\*, EID, Alias | passthrough | Create | — | low | primaryEmail replaced by the gsuite-email-generation result. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | One account per employee | Do not create when the found account carries the same employee ID or email generation reports Existing. | High |
| BR-002 | Suspension | suspended flag set from the caller; notify the suspended list. | High |
| BR-003 | Rehire restore | Restore with new password and caller-supplied flags; notify trainers. | Medium |
| BR-004 | Notice routing | US new hires: manager + named HR list; others: manager + consultant; name updates and rehires: trainers; suspensions: suspended. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive lifecycle event** — Operation from calling job
2. **Read HCM details** — OTBI by EID
3. **Look up Google user** — Relay getGSuiteUser
4. **User found?**
5. **Which operation?** — Update/Suspend/Rehire/Create
6. **Notify error, callback** — Licences -\> service desk
7. **Update name** — Notify trainers
8. **Suspend or reactivate** — Errors not raised
9. **Restore rehire** — New password emailed
10. **Same employee ID?** — Create with found user
11. **Create new account** — Email generation + gsuite-create-user
12. **Email manager and HR** — Includes default password
13. **Callback result**

## Data Model

_Uses the Google user profile owned by gsuite-create-user; the lifecycle request is an inline schema replicating Google user fields._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback | Process start + Return Documents |
| Relay getGSuiteUser / updateGSuiteUser | op-google-user-get (gsuite-email-generation) / op-google-user-update |
| Fault-driven not-found handling | Decision on HTTP 404 |
| Async ProcessEmailGenerationABCSImpl / ProcessCreateGSuitUser | Process Calls gsuite-email-generation / gsuite-create-user |
| xform\_suspend\_gsuiteuser / xform\_rehire\_gsuite\_user | map-gsuite-suspend / map-gsuite-rehire |
| PwdGenerationClass | fn-password-generator (ad-create-user) if retained |
| catchAll | Try/Catch + util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| op-google-user-update | PUT/PATCH /users/{userKey} |
| map-gsuite-update-name | Name update map |
| map-gsuite-suspend | Suspend/reactivate map |
| map-gsuite-rehire | Rehire map |
| proc-gsuite-user-lifecycle | Main process |

## Client questions

- **Q-001** How should new and returning employees receive their Google sign-in details, given that today passwords are sent by plain email to managers, named HR staff and the trainers list?
  **If this is not answered:** No password in any email; users set their password through the agreed first-sign-in process, and notices only confirm the account.
- **Q-002** Which distribution lists should receive the "Google account created" notice for US and for non-US hires, instead of the named individuals in the code?
  **If this is not answered:** Hiring manager plus the EmailDL "GSuit" list.
- **Q-003** If suspending a leaver's Google account fails, who must be alerted?
  **If this is not answered:** Alert the IT service desk list immediately.
- **Q-004** For rehires, should the old Google account always be restored, or should a new account be created when the old one was deleted?
  **If this is not answered:** Keep the current behaviour.
