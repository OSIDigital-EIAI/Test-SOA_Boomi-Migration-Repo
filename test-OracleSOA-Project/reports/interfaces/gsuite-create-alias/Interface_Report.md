# ProcessCreateGSuiteAlias

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

Adds an email alias to an existing Google Workspace user (Admin SDK users.aliases.insert) and returns the created alias, kind, etag and id. A success email is sent to the IT service desk list.

| Field | Value |
| --- | --- |
| Interface | ProcessCreateGSuiteAlias |
| Type | Listener |
| Complexity | simple |
| Components to build | 1 |
| Mapping rules | 3 |
| Business rules | 1 |

## Interface Summary

When an employee's preferred name changes in HCM, this service adds that name as an extra email address on their Google account. It gets an access key, asks Google to add the alias and tells the IT service desk.  It calls Google directly (not through the Java relay). However, the web-address template expects a parameter called "Email\_ID" while the code supplies one called "email", so the employee's address may never be filled into the request; whether the live service works cannot be confirmed from the export. Both the success and error emails use a format the notification service cannot read, so they are probably never delivered, and on error the caller receives no reply.  In Boomi this is a small sub-process using the shared alias operation from gsuite-create-user.

**Complexity:** simple (score 10)

## Functional Analysis

1. **Receive request** — Synchronous process with email and alias.
2. **Get access token** — ProcessGetGSuitAccessToken.
3. **Create alias** — POST alias JSON to users/{email}/aliases with bearer token.
4. **Notify success** — EmailDL itservicedesk "Successfully created Alias for Gsuite EmailID".
5. **Reply** — alias, kind, etag, id.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 synchronous | http://129.144.156.28:80/soa-infra/services/default/ProcessCreateGSuiteAlias/processcreategsuitealias\_client\_ep | none declared |
| outbound | REST over HTTPS | https://www.googleapis.com/admin/directory/v1/users/{Email\_ID}/aliases | OAuth 2.0 bearer |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Alias request and response | alias | process/alias | passthrough | — | — | low | — |
| MR-002 | Alias request and response | URI userKey | process/email | passthrough | — | — | low | Property name mismatch in legacy. |
| MR-003 | Alias request and response | response alias/kind/etag/id | Google reply | passthrough | — | — | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Preferred-name alias | Alias value is supplied by the caller (emp-update-sch builds it from the preferred name). | Low |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive alias request** — From emp-update-sch
2. **Get access token**
3. **Create Google alias** — POST users/{email}/aliases
4. **Notify service desk** — Success notice
5. **Notify error, no reply**
6. **Reply alias**

## Data Model

_Uses the alias profile owned by gsuite-create-user._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / reply | Sub-process start + Return Documents |
| CreateGSuitAlias REST reference | op-google-alias-insert |
| Notifications | util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| proc-sub-gsuite-create-alias | Main sub-process |

## Client questions

- **Q-001** Are preferred-name aliases currently being created in Google when employees change their known-as name?
  **If this is not answered:** Build it working as designed; no catch-up.
- **Q-002** Does the IT service desk want an email for every alias created, or only for failures?
  **If this is not answered:** Failures only.
