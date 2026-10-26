# Migration Plan — ProcessHCMExtractSyncProv + ProcessHCMExtractProvABCS

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.723Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessHCMExtractSyncProv + ProcessHCMExtractProvABCS |
| Identifier | hcm-otbi-sql-query |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | medium (35) |
| Components to build | 7 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/BPEL/ProcessHCMExtractSyncProv.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessHCMExtractProvABCS/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessHCMExtractProvABCS/SOA/BPEL/ProcessHCMExtractProvABCS.bpel |
| Depends on | Nothing |

## 2. End-to-End Process Architecture

Two near-identical composites. TeleOptiWorkforce/ProcessHCMExtractSyncProv exposes a synchronous SOAP operation; TimeCard/ProcessHCMExtractProvABCS exposes an asynchronous one with a callback. Each BPEL reads a session ID from a node-local Coherence cache, builds executeSQLQuery with an XSLT, calls the HCM BI XmlViewService through a WSDL held in MDS (oramds:/apps/Provider/HCM/hcmextract.wsdl), and on "Invalid session ID" logs on with credentials from Credentials.dvm, stores the new session and retries. Callers bind via MDS client WSDLs pointing at the SOA host and parse the escaped rowset themselves with oraext:parseXML.

One shared Boomi sub-process "Sub: HCM OTBI Execute SQL" (proc-sub-otbi-query), invoked synchronously by Process Call with DPP_SQL. It reads the last session ID from a persisted process property, runs op-otbi-execsql on conn-hcm-otbi-soap, and in a Try/Catch re-logs on with op-otbi-logon and retries once when the error text contains "Invalid session ID". A Groovy step (fn-otbi-unescape-rowset) turns the escaped rowset string into the prof-otbi-rowset structure returned to the caller. All other errors are raised. Credentials live on the connection; endpoints are environment extensions.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| Sync and async composites (two SOAP services) | One sub-process proc-sub-otbi-query called synchronously | Async callback pattern is not needed in Boomi. |
| Coherence adapter get/put (adapter-local cache) | Persisted process property holding the session ID (or logon per execution) | No Boomi equivalent of a node-local Coherence cache; decision in questions. |
| HCMExtract reference (XmlViewService.executeSQLQuery) | op-otbi-execsql on conn-hcm-otbi-soap |  |
| HCMSessionLogon reference (SAWSessionService.logon) | op-otbi-logon on conn-hcm-otbi-soap |  |
| xform_hcm_extract_to_sqlquery.xsl / Transformation1_1.xsl | Map or Message into prof-otbi-execsql-req |  |
| Caller-side oraext:parseXML of return/rowset | fn-otbi-unescape-rowset inside the sub-process | Simplifies every caller. |
| Credentials.dvm username/password | Connection / environment extension fields |  |
| catchAll "Invalid session ID" -> logon, put, retry | Try/Catch + Decision on error message + single retry |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive SQL request | Operation "process" receives executeSQLQuery (namespace urn://oracle.bi.webservices/v7). Only the sql element is used; outputFormat, executionOptions and sessionID supplied by the caller are ignored. Sync version replies; async version calls back. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/BPEL/ProcessHCMExtractSyncProv.bpel |
| 2 | Read cached BI session | Coherence Get on cache "adapter-local" with key 'SessionID' (sync) or 'Session' (async); the stored value becomes the session ID for the call. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Adapters/GetSessionID_coherence.jca |
| 3 | Build executeSQLQuery request | XSLT copies sql, sets sessionID from the cached session and emits empty outputFormat and executionOptions (async, maxRowsPerPage, refresh, presentationInfo, type). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Transformations/xform_hcm_extract_to_sqlquery.xsl |
| 4 | Run the logical SQL | Invoke XmlViewService.executeSQLQuery on the HCM analytics endpoint; the executeSQLQueryResult (return/rowset, queryID, finished) is copied to the response. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/BPEL/ProcessHCMExtractSyncProv.bpel |
| 5 | On "Invalid session ID": log on | Fault handler checks ora:getFaultAsString() for "Invalid session ID"; if so, calls SAWSessionService.logon with name and password looked up from Credentials.dvm (keys username, password). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/BPEL/ProcessHCMExtractSyncProv.bpel |
| 6 | Store new session | Coherence Put of the new sessionID under the same key (TimeToLive 0). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Adapters/PutSessionID_coherence.jca |
| 7 | Retry the query once | Rebuilds the request with the new session ID, re-invokes executeSQLQuery and copies the result to the response. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/BPEL/ProcessHCMExtractSyncProv.bpel |
| 8 | Return result | Sync: reply with executeSQLQueryResult. Async: callback processResponse with executeSQLQueryResult. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessHCMExtractProvABCS/SOA/BPEL/ProcessHCMExtractProvABCS.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| HCM OTBI SQL EXECUTE Req XML Prof | xml | executeSQLQuery/sql | string | yes |
| HCM OTBI SQL EXECUTE Req XML Prof | xml | executeSQLQuery/outputFormat | XMLQueryOutputFormat | yes |
| HCM OTBI SQL EXECUTE Req XML Prof | xml | executeSQLQuery/executionOptions/async | boolean | no |
| HCM OTBI SQL EXECUTE Req XML Prof | xml | executeSQLQuery/executionOptions/maxRowsPerPage | int | no |
| HCM OTBI SQL EXECUTE Req XML Prof | xml | executeSQLQuery/executionOptions/refresh | boolean | no |
| HCM OTBI SQL EXECUTE Req XML Prof | xml | executeSQLQuery/executionOptions/presentationInfo | boolean | no |
| HCM OTBI SQL EXECUTE Req XML Prof | xml | executeSQLQuery/executionOptions/type | string | no |
| HCM OTBI SQL EXECUTE Req XML Prof | xml | executeSQLQuery/sessionID | string | no |
| HCM OTBI SQL EXECUTE Req XML Prof | xml | executeSQLQueryResult/return/rowset | string | yes |
| HCM OTBI SQL EXECUTE Req XML Prof | xml | executeSQLQueryResult/return/queryID | string | yes |
| HCM OTBI SQL EXECUTE Req XML Prof | xml | executeSQLQueryResult/return/finished | boolean | yes |
| HCM OTBI Rowset XML Prof | xml | rowset/Row | repeating element | yes |
| HCM OTBI Rowset XML Prof | xml | rowset/Row/Column0 | string | yes |
| HCM OTBI Rowset XML Prof | xml | rowset/Row/Column1 .. Column70 | string | yes |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | executeSQLQuery/sql | /executeSQLQuery/sql | passthrough |  |  | low | Callers typically prefix "SET VARIABLE PARAM_EFFECTIVE_DATE ='4000-12-31';". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Transformations/xform_hcm_extract_to_sqlquery.xsl | Caller request to executeSQLQuery |
| MR-002 | executeSQLQuery/sessionID | cached session (Coherence) or logonResult/sessionID | transformed |  | parameter $HCMSessionLogon_Response.parameters/logonResult/sessionID | low | Caller-supplied sessionID is ignored. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Transformations/xform_hcm_extract_to_sqlquery.xsl | Caller request to executeSQLQuery |
| MR-003 | executeSQLQuery/outputFormat | (none) | hardcoded |  | empty element | low | Schema type XMLQueryOutputFormat; left empty. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Transformations/xform_hcm_extract_to_sqlquery.xsl | Caller request to executeSQLQuery |
| MR-004 | executeSQLQuery/executionOptions/(async, maxRowsPerPage, refresh, presentationInfo, type) | (none) | hardcoded |  | empty elements | low | Schema types are boolean/int; empty values are sent. Paging decision affects maxRowsPerPage. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/hcmextract.wsdl | Caller request to executeSQLQuery |
| MR-005 | logon/name | Credentials.dvm key=username | transformed | only after "Invalid session ID" | dvm:lookupValue | low | Moves to the Boomi connection. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/BPEL/ProcessHCMExtractSyncProv.bpel | Logon request |
| MR-006 | logon/password | Credentials.dvm key=password | transformed | only after "Invalid session ID" | dvm:lookupValue | low | Value not reproduced. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/BPEL/ProcessHCMExtractSyncProv.bpel | Logon request |
| MR-007 | executeSQLQueryResult (to caller) | HCM executeSQLQueryResult | passthrough |  | whole element copied | low | Legacy returns the raw result; each caller then applies oraext:parseXML on return/rowset. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/BPEL/ProcessHCMExtractSyncProv.bpel | executeSQLQueryResult to caller / rowset |
| MR-008 | rowset/Row/ColumnN | return/rowset (escaped XML string) | transformed |  | un-escape and parse (replaces caller-side oraext:parseXML) | medium | New in Boomi: done once in the sub-process so callers receive a structured rowset. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel | executeSQLQueryResult to caller / rowset |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Reuse BI session | Use the cached session ID for every query; only log on when HCM answers "Invalid session ID". | Medium | Persisted process property DPP_OTBI_SessionID read at the start; Try/Catch around the query with a Decision on the error text; or simply log on per execution if session limits allow (see questions). |
| BR-002 | Single retry after re-logon | After a successful re-logon the query is retried exactly once. | Medium | Catch path: logon op, set property, re-run execsql op once; any further error is raised. |
| BR-003 | Result column order | Rows are returned as Column0..ColumnN in the order of the select list (alias s_0..s_N); callers depend on these positions. | High | Keep the generic Column0..Column70 profile; callers map by position. |
| BR-004 | Caller credentials not accepted | Session IDs and options sent by callers are ignored; the service always uses its own session. | Low | Sub-process only accepts DPP_SQL. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 synchronous | http://129.144.156.28:80/soa-infra/services/default/ProcessHCMExtractSyncProv/processhcmextractsyncprov_client_ep (PROD MDS client WSDL) | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl |
| inbound SOAP 1.1 asynchronous (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessHCMExtractProvABCS/processhcmextractprovabcs_client_ep (PROD MDS client WSDL) | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/processhcmextractprovabcs_client_ep.wsdl |
| outbound SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com/analytics-ws/saw.dll?SoapImpl=nQSessionService | BI username/password in the logon request body | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/hcmextract.wsdl |
| outbound SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com/analytics-ws/saw.dll?SoapImpl=xmlViewService | BI session ID in the request body | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/hcmextract.wsdl |
| Oracle Coherence adapter (Get) | JNDI eis/Coherence/Local; CacheName adapter-local; CacheOperation get; KeyType java.lang.String; key set at runtime via jca.coherence.Key = 'SessionID' (sync) / 'Session' (async). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Adapters/GetSessionID_coherence.jca |
| Oracle Coherence adapter (Put) | JNDI eis/Coherence/Local; CacheName adapter-local; CacheOperation put; TimeToLive 0; KeyType java.lang.String. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Adapters/PutSessionID_coherence.jca |
| SOAP reference HCMExtract / HCMSessionLogon (HCMExtractSession in async) | binding.ws to oramds:/apps/Provider/HCM/hcmextract.wsdl ports [redacted] and [redacted]; weblogic.wsee.wsat.transaction.flowOption=WSDLDriven. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/composite.xml |

## 8. Sub-Process Call Hierarchy

> **Not available.** This interface calls no sub-processes.

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Transformations/xform_hcm_extract_to_sqlquery.xsl | Builds executeSQLQuery: sql passthrough, session ID from a parameter, empty execution options. | Map or Message shape building prof-otbi-execsql-req from DPP_SQL and the session property. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessHCMExtractProvABCS/SOA/Transformations/Transformation1_1.xsl | Same as the sync XSLT (only the parameter name differs). | Same map; not duplicated. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Scope-level catchAll around the query: if the fault text contains "Invalid session ID", log on, store the session and retry once; otherwise rethrow (sync). The async composite adds a process-level catchAll that always sends the callback, which hides errors. |
| Retries | One retry, only for "Invalid session ID". |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/HCM/hcmextract.wsdl (BI v7 web services) | mds | yes |
| oramds:/apps/Utilities/DVM/Credentials.dvm (keys username, password) | dvm | yes |
| oramds:/apps/Utilities/AD_XSD/RowSet.xsd / Utilities/XSD/RowSet.xsd (rowset schema used by callers) | mds | yes |
| oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl and processhcmextractprovabcs_client_ep.wsdl (client WSDLs) | mds | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud BI - XmlViewService | SOAP 1.1 / HTTPS | https://eetm.hcm.us6.oraclecloud.com/analytics-ws/saw.dll?SoapImpl=xmlViewService | Oracle (TaskUs HCM Cloud tenant) |
| Oracle HCM Cloud BI - SAWSessionService | SOAP 1.1 / HTTPS | https://eetm.hcm.us6.oraclecloud.com/analytics-ws/saw.dll?SoapImpl=nQSessionService | Oracle (TaskUs HCM Cloud tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | HCM analytics base | https://eetm.hcm.us6.oraclecloud.com/analytics-ws/saw.dll | [redacted].wsdl |
| DEV | HCM analytics base | https://ebly-dev1.hcm.us2.oraclecloud.com/analytics-ws/saw.dll | [redacted].wsdl |
| PROD | Sync service endpoint | http://129.144.156.28:[redacted] | [redacted].wsdl |

## 14. Boomi Migration Readiness Summary

This is the integration estate's window into Oracle HCM reporting data. A calling integration writes a query in Oracle BI "logical SQL", sends it here, and receives the matching rows. The service handles the HCM analytics logon itself: it keeps the session identifier in a server cache, reuses it on later calls, and logs on again automatically when HCM reports that the session has expired.

The same service was built twice: one version answers immediately, the other replies later through a callback. They have the same logic, but each keeps its own cached session. In Boomi they become a single shared sub-process that returns the rows directly to the calling process, so callers no longer need to unpack the result themselves.

About 26 interfaces depend on this service, so it is built early, straight after the email utilities. Three behaviours carry over as risks unless they are addressed. Queries are not paged, so very large results may be cut off. The asynchronous version hides errors by returning an empty answer. And callers build their queries by joining values such as person numbers directly into the query text.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | HCM OTBI Rowset XML Prof | prof-otbi-rowset | profile.xml | — |
| 2 | HCM OTBI SQL EXECUTE Req XML Prof | prof-otbi-execsql-req | profile.xml | — |
| 3 | HCM OTBI Analytics SOAP Conn | conn-hcm-otbi-soap | connector-settings | — |
| 4 | HCM OTBI Session LOGON Op | op-otbi-logon | connector-action | conn-hcm-otbi-soap |
| 5 | HCM OTBI SQL EXECUTE Op | op-otbi-execsql | connector-action | conn-hcm-otbi-soap |
| 6 | OTBI Rowset Unescape Script | fn-otbi-unescape-rowset | transform.function | — |
| 7 | Sub: HCM OTBI Execute SQL | proc-sub-otbi-query | process | op-otbi-logon, op-otbi-execsql, prof-otbi-execsql-req, prof-otbi-rowset, fn-otbi-unescape-rowset |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Credentials.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/BPEL/ProcessHCMExtractSyncProv.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Transformations/xform_hcm_extract_to_sqlquery.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/WSDLs/ProcessHCMExtractSyncProv.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Schemas/ProcessHCMExtractSyncProv.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Adapters/GetSessionID_coherence.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Adapters/PutSessionID_coherence.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessHCMExtractSyncProv/SOA/Schemas/GetSessionIDRequest.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessHCMExtractProvABCS/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessHCMExtractProvABCS/SOA/BPEL/ProcessHCMExtractProvABCS.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessHCMExtractProvABCS/SOA/Transformations/Transformation1_1.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessHCMExtractProvABCS/SOA/WSDLs/ProcessHCMExtractProvABCS.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessHCMExtractProvABCS/ProcessHCMExtractProvABCS_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessHCMExtractProvABCS/SOA/Adapters/GetSessionID_coherence.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessHCMExtractProvABCS/SOA/Adapters/PutSessionID_coherence.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/hcmextract.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/DEV_MDS/apps/Provider/HCM/hcmextract.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/processhcmextractprovabcs_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/XSD/RowSet.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/BPEL/ProcessTerminateEmpReq_Sch.bpel

## Open questions

### Still outstanding

REVIEW-NEEDED: Which Oracle HCM reporting (BI) service account should the migrated integrations use to query HCM data? (The current account is stored in a shared lookup table in plain text; the Boomi connection cannot be configured until a supported account is named and issued.)
  - If this is not answered: Reuse the existing integration BI account with a newly rotated password stored only on the Boomi connection.

REVIEW-NEEDED: Can any of these HCM queries return more than a few thousand rows (for example the full list of active employees), and must every row be returned? (The current service never asks for further pages, so large results may be silently cut off; this decides whether paging must be built.)
  - If this is not answered: Implement paging so that all rows are always returned.

REVIEW-NEEDED: If HCM cannot be queried (for example HCM is unavailable or a query is wrong), should the calling integration stop with an error rather than continue as if no employees were found? (One of the two current versions returns an empty answer on any error, so scheduled jobs may skip employees without anyone knowing.)
  - If this is not answered: Raise the error to the calling integration, which then notifies support.

REVIEW-NEEDED: Is it acceptable for the integrations to log on to HCM reporting once per run instead of keeping a long-lived shared session? (The current session cache has no direct equivalent in Boomi; logging on per run is simpler but adds logon calls and may hit HCM session limits.)
  - If this is not answered: Keep reusing a stored session and log on again only when HCM reports it has expired, as today.

REVIEW-NEEDED: Does TaskUs plan to keep using the Oracle BI web services for HCM queries, or should integrations move to HCM REST APIs or BI Publisher reports where possible? (The BI SOAP web services are an older interface; moving would change how all callers query HCM.)
  - If this is not answered: Keep the BI web services (executeSQLQuery) as today.

