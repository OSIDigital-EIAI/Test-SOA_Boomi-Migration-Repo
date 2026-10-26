# ProcessGetPersonNumber

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

Finds the HCM Person Number of a person created in Oracle HCM today, either by national identifier (e.g. the Philippine SSS number) or, in the Employee\_JobVite version, by Jobvite candidate ID held in the assignment flexfield PER\_ASG\_DF\_CANDIDATE\_ID\_. It runs an OTBI logical SQL query and returns the first Person Number found.

| Field | Value |
| --- | --- |
| Interface | ProcessGetPersonNumber |
| Type | Listener |
| Complexity | simple |
| Components to build | 1 |
| Mapping rules | 3 |
| Business rules | 2 |

## Interface Summary

When a new hire from Jobvite has been loaded into HCM, the follow-up steps need the employee number HCM gave them. This service looks it up: if a national ID (such as the Philippine SSS number) is supplied it searches people by that ID; otherwise it searches by the Jobvite candidate ID stored on the assignment. In both cases it only considers people created in HCM today.  The "created today" rule uses the integration server's date, so a hire loaded late in the day in one time zone may not be found, and a hire whose load ran on a previous day will never be found. Inputs are put directly into the query text, and when nothing is found the service returns an empty number without saying why. The deployed contract and the caller send first name, last name and start date, but the service ignores them.

**Complexity:** simple (score 14)

## Functional Analysis

1. **Receive request** — Async SOAP process (createInstance).
2. **Choose lookup key** — If National\_Id != "": national identifier query; else candidate ID query.
3. **Run OTBI query** — ProcessHCMExtractSyncProv with the logical SQL; creation date LIKE current server date.
4. **Return person number** — parseXML(rowset); Person\_Number = Row\[1\]/Column1; callback.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessGetPersonNumber/processgetpersonnumber\_client\_ep | none declared |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared |
| outbound | SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification\_client\_ep.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Request to OTBI SQL and result | sql | National\_Id | conditional | National\_Id != "" | Person Real Time query by National Identifier Number, Creation Date LIKE today% | medium | — |
| MR-002 | Request to OTBI SQL and result | sql | CandidateID | conditional | National\_Id empty | Worker Assignment Real Time query by PER\_ASG\_DF\_CANDIDATE\_ID\_, Creation Date LIKE today% | medium | Employee\_JobVite version only. |
| MR-003 | Request to OTBI SQL and result | Person\_Number | rowset Row\[1\]/Column1 | passthrough | — | — | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Lookup key priority | National identifier first; candidate ID when no national identifier is given. | Medium |
| BR-002 | Created today only | Only persons whose Creation Date starts with the current server date. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive lookup request** — From dat-profile-generation-rest
2. **National ID supplied?**
3. **Query by national ID** — Person Real Time, created today
4. **Query by candidate ID** — Assignment flexfield, created today
5. **Notify error** — EmailDL notification; empty result
6. **Return person number** — First row, may be empty

## Data Model

_This interface owns only its process component; it uses the OTBI rowset profile of hcm-otbi-sql-query and a small inline request schema._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback | Sub-process start + Return Documents |
| If1 + bpelx:append SQL | Decision + Set Properties (escaped values) |
| HCMExtractSyncProv | Process Call hcm-otbi-sql-query |
| catchAll | Try/Catch + util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| proc-sub-hcm-get-person-number | Main sub-process |

## Client questions

- **Q-001** Should the person-number lookup only consider people created in HCM today, and if so in which time zone is "today"?
  **If this is not answered:** Look back three days in UTC.
- **Q-002** If several HCM people share the same national ID or candidate ID, which one should be used?
  **If this is not answered:** Use the most recently created person and report the duplicate.
