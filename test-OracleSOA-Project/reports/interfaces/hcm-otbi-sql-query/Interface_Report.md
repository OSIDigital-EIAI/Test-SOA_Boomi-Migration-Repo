# ProcessHCMExtractSyncProv + ProcessHCMExtractProvABCS

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

This shared service lets other integrations ask Oracle HCM Cloud a question in Oracle BI (OTBI) logical SQL, for example "which workers were terminated between these two times" or "what is this person's department, project and manager", and get the answer back as rows and columns. It handles logging on to the HCM analytics service and reuses the BI session between calls.

| Field | Value |
| --- | --- |
| Interface | ProcessHCMExtractSyncProv + ProcessHCMExtractProvABCS |
| Type | Listener |
| Complexity | medium |
| Components to build | 7 |
| Mapping rules | 8 |
| Business rules | 4 |

## Interface Summary

This is the integration estate's window into Oracle HCM reporting data. A calling integration writes a query in Oracle BI "logical SQL", sends it here, and receives the matching rows. The service handles the HCM analytics logon itself: it keeps the session identifier in a server cache, reuses it on later calls, and logs on again automatically when HCM reports that the session has expired.  The same service was built twice: one version answers immediately, the other replies later through a callback. They have the same logic, but each keeps its own cached session. In Boomi they become a single shared sub-process that returns the rows directly to the calling process, so callers no longer need to unpack the result themselves.  About 26 interfaces depend on this service, so it is built early, straight after the email utilities. Three behaviours carry over as risks unless they are addressed. Queries are not paged, so very large results may be cut off. The asynchronous version hides errors by returning an empty answer. And callers build their queries by joining values such as person numbers directly into the query text.

**Complexity:** medium (score 35)

## Functional Analysis

1. **Receive SQL request** — Operation "process" receives executeSQLQuery (namespace urn://oracle.bi.webservices/v7). Only the sql element is used; outputFormat, executionOptions and sessionID supplied by the caller are ignored. Sync version replies; async version calls back.
2. **Read cached BI session** — Coherence Get on cache "adapter-local" with key 'SessionID' (sync) or 'Session' (async); the stored value becomes the session ID for the call.
3. **Build executeSQLQuery request** — XSLT copies sql, sets sessionID from the cached session and emits empty outputFormat and executionOptions (async, maxRowsPerPage, refresh, presentationInfo, type).
4. **Run the logical SQL** — Invoke XmlViewService.executeSQLQuery on the HCM analytics endpoint; the executeSQLQueryResult (return/rowset, queryID, finished) is copied to the response.
5. **On "Invalid session ID": log on** — Fault handler checks ora:getFaultAsString() for "Invalid session ID"; if so, calls SAWSessionService.logon with name and password looked up from Credentials.dvm (keys username, password).
6. **Store new session** — Coherence Put of the new sessionID under the same key (TimeToLive 0).
7. **Retry the query once** — Rebuilds the request with the new session ID, re-invokes executeSQLQuery and copies the result to the response.
8. **Return result** — Sync: reply with executeSQLQueryResult. Async: callback processResponse with executeSQLQueryResult.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 synchronous | http://129.144.156.28:80/soa-infra/services/default/ProcessHCMExtractSyncProv/processhcmextractsyncprov\_client\_ep (PROD MDS client WSDL) | none declared |
| inbound | SOAP 1.1 asynchronous (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessHCMExtractProvABCS/processhcmextractprovabcs\_client\_ep (PROD MDS client WSDL) | none declared |
| outbound | SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com/analytics-ws/saw.dll?SoapImpl=nQSessionService | BI username/password in the logon request body |
| outbound | SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com/analytics-ws/saw.dll?SoapImpl=xmlViewService | BI session ID in the request body |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Caller request to executeSQLQuery | executeSQLQuery/sql | /executeSQLQuery/sql | passthrough | — | — | low | Callers typically prefix "SET VARIABLE PARAM\_EFFECTIVE\_DATE ='4000-12-31';". |
| MR-002 | Caller request to executeSQLQuery | executeSQLQuery/sessionID | cached session (Coherence) or logonResult/sessionID | transformed | — | parameter $HCMSessionLogon\_Response.parameters/logonResult/sessionID | low | Caller-supplied sessionID is ignored. |
| MR-003 | Caller request to executeSQLQuery | executeSQLQuery/outputFormat | (none) | hardcoded | — | empty element | low | Schema type XMLQueryOutputFormat; left empty. |
| MR-004 | Caller request to executeSQLQuery | executeSQLQuery/executionOptions/(async, maxRowsPerPage, refresh, presentationInfo, type) | (none) | hardcoded | — | empty elements | low | Schema types are boolean/int; empty values are sent. Paging decision affects maxRowsPerPage. |
| MR-005 | Logon request | logon/name | Credentials.dvm key=username | transformed | only after "Invalid session ID" | dvm:lookupValue | low | Moves to the Boomi connection. |
| MR-006 | Logon request | logon/password | Credentials.dvm key=password | transformed | only after "Invalid session ID" | dvm:lookupValue | low | Value not reproduced. |
| MR-007 | executeSQLQueryResult to caller / rowset | executeSQLQueryResult (to caller) | HCM executeSQLQueryResult | passthrough | — | whole element copied | low | Legacy returns the raw result; each caller then applies oraext:parseXML on return/rowset. |
| MR-008 | executeSQLQueryResult to caller / rowset | rowset/Row/ColumnN | return/rowset (escaped XML string) | transformed | — | un-escape and parse (replaces caller-side oraext:parseXML) | medium | New in Boomi: done once in the sub-process so callers receive a structured rowset. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Reuse BI session | Use the cached session ID for every query; only log on when HCM answers "Invalid session ID". | Medium |
| BR-002 | Single retry after re-logon | After a successful re-logon the query is retried exactly once. | Medium |
| BR-003 | Result column order | Rows are returned as Column0..ColumnN in the order of the select list (alias s\_0..s\_N); callers depend on these positions. | High |
| BR-004 | Caller credentials not accepted | Session IDs and options sent by callers are ignored; the service always uses its own session. | Low |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive SQL request** — process(executeSQLQuery); only sql used
2. **Read cached session ID** — Coherence get adapter-local, key SessionID / Session
3. **Run executeSQLQuery** — XmlViewService with cached session; empty execution options
4. **Invalid session ID?** — catchAll: contains(fault, "Invalid session ID")
5. **Log on to HCM BI** — SAWSessionService.logon with Credentials.dvm user
6. **Store new session ID** — Coherence put, TTL 0
7. **Retry executeSQLQuery once** — Same request with new session
8. **Return result rows** — Sync reply / async callback with executeSQLQueryResult
9. **Raise or hide error** — Sync rethrows; async calls back empty

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| HCM OTBI SQL EXECUTE Req XML Prof | xml | prof-otbi-execsql-req |
| HCM OTBI Rowset XML Prof | xml | prof-otbi-rowset |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| Sync and async composites (two SOAP services) | One sub-process proc-sub-otbi-query called synchronously |
| Coherence adapter get/put (adapter-local cache) | Persisted process property holding the session ID (or logon per execution) |
| HCMExtract reference (XmlViewService.executeSQLQuery) | op-otbi-execsql on conn-hcm-otbi-soap |
| HCMSessionLogon reference (SAWSessionService.logon) | op-otbi-logon on conn-hcm-otbi-soap |
| xform\_hcm\_extract\_to\_sqlquery.xsl / Transformation1\_1.xsl | Map or Message into prof-otbi-execsql-req |
| Caller-side oraext:parseXML of return/rowset | fn-otbi-unescape-rowset inside the sub-process |
| Credentials.dvm username/password | Connection / environment extension fields |
| catchAll "Invalid session ID" -\> logon, put, retry | Try/Catch + Decision on error message + single retry |

**Components**

| Component | Role |
| --- | --- |
| conn-hcm-otbi-soap | SOAP connection to HCM analytics (saw.dll) |
| op-otbi-logon | SAWSessionService.logon operation |
| op-otbi-execsql | XmlViewService.executeSQLQuery operation |
| prof-otbi-execsql-req | executeSQLQuery request / result profile |
| prof-otbi-rowset | Generic rowset profile Column0..Column70 |
| fn-otbi-unescape-rowset | Groovy un-escape/parse of the rowset string |
| proc-sub-otbi-query | Shared sub-process |

## Client questions

- **Q-001** Which Oracle HCM reporting (BI) service account should the migrated integrations use to query HCM data?
  **If this is not answered:** Reuse the existing integration BI account with a newly rotated password stored only on the Boomi connection.
- **Q-002** Can any of these HCM queries return more than a few thousand rows (for example the full list of active employees), and must every row be returned?
  **If this is not answered:** Implement paging so that all rows are always returned.
- **Q-003** If HCM cannot be queried (for example HCM is unavailable or a query is wrong), should the calling integration stop with an error rather than continue as if no employees were found?
  **If this is not answered:** Raise the error to the calling integration, which then notifies support.
- **Q-004** Is it acceptable for the integrations to log on to HCM reporting once per run instead of keeping a long-lived shared session?
  **If this is not answered:** Keep reusing a stored session and log on again only when HCM reports it has expired, as today.
- **Q-005** Does TaskUs plan to keep using the Oracle BI web services for HCM queries, or should integrations move to HCM REST APIs or BI Publisher reports where possible?
  **If this is not answered:** Keep the BI web services (executeSQLQuery) as today.
