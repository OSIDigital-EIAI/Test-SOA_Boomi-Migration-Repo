# getHDLErrorsProvABCSImpl

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

Returns the HCM Data Loader errors for a given UCM content ID. It runs the BI Publisher report "/Human Capital Management/TaskUs HDL Status Report.xdo" with parameter ContentID, decodes the Base64 report output and returns it as DATA\_DS (CONTENTID and repeating G\_1 rows with KEY\_SOURCE\_OWNER, KEY\_SOURCE\_ID, MSG\_TEXT).

| Field | Value |
| --- | --- |
| Interface | getHDLErrorsProvABCSImpl |
| Type | Listener |
| Complexity | simple |
| Components to build | 5 |
| Mapping rules | 5 |
| Business rules | 1 |

## Interface Summary

After data is loaded into HCM, the integrations need to know which records failed. This service asks Oracle's reporting engine (BI Publisher) to run the TaskUs "HDL Status Report" for a given load and returns the list of errors: which record and the error message.  It is simple, but the legacy version sends a stored user name and password inside the report request as well as in the request header, writes the decoded report (which contains employee identifiers) to the server log, and has no error handling: if the report call fails the caller never gets an answer. The Boomi version should use the connection's authentication only, keep data out of logs and always return a result.

**Complexity:** simple (score 18)

## Functional Analysis

1. **Receive content ID** — Async SOAP process (createInstance) with ContentId.
2. **Build report request** — xform\_input\_report: report path, ContentID parameter, userID/password from Credentials.dvm.
3. **Run report** — PublicReportService.runReport with HTTP Authorization header.
4. **Decode output** — Java Base64 decode of reportBytes; strip text up to the first "--\>"; parseEscapedXML.
5. **Return errors** — Callback with DATA\_DS.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/getHDLErrorsProvABCSImpl/gethdlerrorsprovabcsimpl\_client\_ep | none declared |
| outbound | SOAP 1.1 over HTTPS | https://eetm.bi.us6.oraclecloud.com/xmlpserver/services/PublicReportService | HTTP Authorization header and userID/password in the SOAP body (Credentials.dvm) |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Content ID to runReport request | reportRequest/reportAbsolutePath | (none) | hardcoded | — | "/Human Capital Management/TaskUs HDL Status Report.xdo" | low | — |
| MR-002 | Content ID to runReport request | parameterNameValues/item (name, label ContentID; UIType Text; dataType xsd:string) | (none) | hardcoded | — | — | low | — |
| MR-003 | Content ID to runReport request | parameterNameValues/item/defaultValue and values/item | process/ContentId | passthrough | — | — | low | — |
| MR-004 | Content ID to runReport request | userID / password | Credentials.dvm | transformed | — | DVM lookup | low | Drop in Boomi; use connection authentication. |
| MR-005 | Report output to response | DATA\_DS | runReportReturn/reportBytes | calculated | — | Base64 decode, strip leading comment, parse XML | medium | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | HDL error source | Errors for a load are read from the custom BI Publisher report /Human Capital Management/TaskUs HDL Status Report.xdo keyed by UCM ContentID. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive content ID** — From DAT profile / external ID jobs
2. **Run HDL status report** — BI Publisher runReport
3. **Decode report output** — Base64 + strip comment
4. **Unhandled fault** — No callback
5. **Return error rows** — DATA\_DS by callback

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| HCM BIP runReport Req XML Prof | xml | prof-bip-runreport-req |
| HCM HDL Status Report XML Prof | xml | prof-hdl-status-report |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback | Sub-process start + Return Documents |
| xform\_input\_report.xsl | Map into prof-bip-runreport-req |
| ErrorReportService runReport | conn-hcm-bip-soap + op-bip-run-report |
| Java Base64DecoderClass | Data Process Base64 decode |
| (no fault handler) | Try/Catch |

**Components**

| Component | Role |
| --- | --- |
| conn-hcm-bip-soap | BI Publisher connection |
| op-bip-run-report | runReport operation |
| prof-bip-runreport-req | Request profile |
| prof-hdl-status-report | Decoded report profile |
| proc-sub-hcm-get-hdl-errors | Main sub-process |

## Client questions

- **Q-001** Will the custom BI Publisher report "TaskUs HDL Status Report" stay in HCM and remain available to the integration user after the migration?
  **If this is not answered:** Yes, unchanged at the same path.
- **Q-002** Which HCM integration user should run the HDL status report from Boomi?
  **If this is not answered:** The same HCM integration user as the other HCM connections.
