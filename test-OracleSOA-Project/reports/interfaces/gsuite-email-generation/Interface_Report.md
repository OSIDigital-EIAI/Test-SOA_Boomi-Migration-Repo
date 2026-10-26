# ProcessEmailGenerationABCSImpl

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

Finds a free Google Workspace email address for a new hire. Starting from the proposed address it looks the address up in Google; if it belongs to the same employee (same employee ID in externalIds) it returns it with status Existing; if it belongs to someone else it tries the next variant (local part + 02, 03 ... 09, then 10, 11 ...); when Google reports the address does not exist it returns it as the address to use.

| Field | Value |
| --- | --- |
| Interface | ProcessEmailGenerationABCSImpl |
| Type | Listener |
| Complexity | simple |
| Components to build | 2 |
| Mapping rules | 3 |
| Business rules | 2 |

## Interface Summary

Before a Google account is created for a new hire, this service checks that the proposed email address is free. It asks Google whether the address exists. If nobody has it, that address is used. If it already belongs to the same employee (for example a rehire), it reports that the account already exists. If it belongs to someone else, it adds a number and tries again: firstname.lastname02, 03 and so on.  The logic works by deliberately raising errors and recognising them by their message text ("Resource Not Found", "Employee already created in Gsuite"), which is fragile: if Google or the relay changes the wording, every new hire would be treated as an error. The numbering starts at 02, never 01. Unexpected errors are emailed in a format the notification service cannot read and the caller receives an empty answer.  In production the lookup goes through the custom Java relay, and the Google access key is sent in the web address of a plain-HTTP request, where it can end up in server logs. In Boomi this becomes a short loop calling Google directly and checking the HTTP status (404 = free).

**Complexity:** simple (score 16)

## Functional Analysis

1. **Receive request** — Async SOAP process with Email and EmployeeID.
2. **Get access token** — Call ProcessGetGSuitAccessToken.
3. **Look up candidate** — GET getGSuiteUser on the Java relay with email and authorization query parameters.
4. **Same employee?** — primaryEmail returned and externalIds value = EmployeeID -\> Existing.
5. **Taken by someone else** — Build next candidate with counter (02-09, then 10+), loop.
6. **Free address** — Error "Resource Not Found: userKey" -\> return candidate.
7. **Callback** — processResponse with Email and Status.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | processemailgenerationabcsimplprocess\_client\_ep | none declared |
| outbound | REST over plain HTTP | http://129.144.156.139:9073/google-restservice-context-root/resources/googleServices/getGSuiteUser | Google bearer token as a query parameter |
| outbound | REST over HTTPS (from relay) | https://www.googleapis.com/admin/directory/v1/users/{email} | OAuth 2.0 bearer |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Candidate email generation | candidate Email (loop) | process/Email, counter | looped | Address taken by another employee | local + ("0"+counter if counter \< 10 else counter) + "@" + domain; counter starts at 2 | medium | — |
| MR-002 | Candidate email generation | Status | Google user externalIds value | conditional | = EmployeeID | "Existing" | low | — |
| MR-003 | Candidate email generation | Email | candidate | passthrough | Not found or Existing | — | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Rehire keeps address | If the address exists and carries the same employee ID, return it as Existing instead of creating a new account. | High |
| BR-002 | Numbered alternatives | Clashing address gets 02..09 then 10.. appended to the local part. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive proposed email** — From gsuite-user-lifecycle
2. **Get access token**
3. **Look up address** — Google users.get via relay
4. **Address exists?**
5. **Notify, empty reply** — Unexpected error
6. **Same employee ID?**
7. **Next numbered address** — 02..09, 10..
8. **Return email** — Free or Existing

## Data Model

_Uses the Google user profile owned by gsuite-create-user; the request/response are two-field inline schemas._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback | Sub-process start + Return Documents |
| repeatUntil + throw/catch on message text | Loop with Decision on HTTP status |
| GetUser\_JavService relay | op-google-user-get |

**Components**

| Component | Role |
| --- | --- |
| op-google-user-get | GET /users/{userKey} |
| proc-sub-gsuite-unique-email | Main sub-process |

## Client questions

- **Q-001** When a proposed email is taken, should the next option be firstname.lastname01 rather than 02, and is a number the preferred way to make it unique?
  **If this is not answered:** Keep the current convention (02, 03 ...).
- **Q-002** Do all existing Google accounts carry the employee ID, so that rehires are recognised by it?
  **If this is not answered:** Yes.
