# ProcessGetGSuitAccessToken

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

Returns a Google OAuth 2.0 access token for the Google Workspace Admin SDK. It performs a refresh-token grant with a client ID, client secret and refresh token that are hardcoded in the BPEL; in the production (GSuite\_Java\_Process) version the request goes through a custom Java REST service on a separate server, which calls https://www.googleapis.com/oauth2/v3/token.

| Field | Value |
| --- | --- |
| Interface | ProcessGetGSuitAccessToken |
| Type | Listener |
| Complexity | simple |
| Components to build | 1 |
| Mapping rules | 3 |
| Business rules | 1 |

## Interface Summary

Before any integration can create or change a Google Workspace account it needs a short-lived access key from Google. This service obtains one. It sends Google a long-lived "refresh token" together with the application's ID and secret, and passes the access key back to the caller.  The ID, secret and refresh token are written directly in the integration code, and in production they are sent over plain HTTP to a small custom Java web service running on another server, which forwards them to Google with the secret in the web address. These values have been exposed in the source code and must be replaced (rotated) as part of the migration. The custom Java service was apparently introduced because the integration server could not validate Google's certificate.  In Boomi this service disappears: the Google connection is configured with OAuth 2.0 and refreshes tokens itself, with the secrets stored encrypted in Boomi. TaskUs should also consider a Google service account with domain-wide delegation instead of a personal refresh token.

**Complexity:** simple (score 8)

## Functional Analysis

1. **Receive request** — Synchronous process (input element Test, unused).
2. **Build token request** — Hardcoded client\_id, client\_secret, refresh\_token and grant\_type refresh\_token \[values redacted\].
3. **Call token service** — Production: Java REST service getAccessToken; Employee variant: Google /oauth2/v3/token directly. HTTP Connection: close.
4. **Reply** — Access\_Token, Expires\_IN, Token\_Type, Error.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 synchronous | processgetgsuitaccesstoken\_client\_ep | none declared |
| outbound | REST over plain HTTP | http://129.144.156.139:9073/google-restservice-context-root/resources/googleServices/getAccessToken | none (secrets in JSON body) |
| outbound | REST over HTTPS | https://www.googleapis.com/oauth2/v3/token | OAuth 2.0 refresh\_token grant (client credentials in query string) |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Token request | grant\_type | (none) | hardcoded | — | "refresh\_token" | low | — |
| MR-002 | Token request | client\_id / client\_secret / refresh\_token | (none) | hardcoded | — | \[redacted literals\] | low | Move to Boomi connection; rotate. |
| MR-003 | Token request | Access\_Token / Token\_Type / Expires\_IN / Error | token response | passthrough | — | — | low | Response mapping. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Token per call | Every Google operation is preceded by a refresh-token grant. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive token request** — From Google integrations
2. **Call Java token service** — Plain HTTP; secrets in body
3. **Java calls Google token** — Secrets in query string
4. **Reply access token** — Error field if failed

## Data Model

_The only component is the Google connection; no profiles are needed once OAuth is handled by the connection._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| ProcessGetGSuitAccessToken composite | OAuth 2.0 on conn-google-directory |
| GSuiteServices Java REST service | (retired) |
| Hardcoded OAuth literals | Encrypted connection fields, rotated values |

**Components**

| Component | Role |
| --- | --- |
| conn-google-directory | Google Admin SDK Directory connection with OAuth 2.0 |

## Client questions

- **Q-001** Will TaskUs's Google administrators create a new OAuth client or service account for Boomi and revoke the credentials used by the current integration?
  **If this is not answered:** A new service account with domain-wide delegation for the Admin SDK user and alias scopes, created by TaskUs, with the old credentials revoked at cut-over.
- **Q-002** Can the custom Java Google service on the separate server be retired once Boomi calls Google directly?
  **If this is not answered:** Yes, retire it after cut-over.
