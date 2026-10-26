# ProcessHDLProvABCSImpl

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

Loads HCM Data Loader (HDL) files into Oracle HCM Cloud. It zips every .dat file found in a shared folder on the SOA server, Base64-encodes the zip, checks it into Oracle WebCenter Content (UCM) in the HDL import account using the caller's check-in request, then asks HCM Data Loader to import and load that content item, and returns the UCM content ID.

| Field | Value |
| --- | --- |
| Interface | ProcessHDLProvABCSImpl |
| Type | Listener |
| Complexity | medium |
| Components to build | 8 |
| Mapping rules | 7 |
| Business rules | 2 |

## Interface Summary

This is the shared service that sends data files into Oracle HCM. Other integrations prepare HCM Data Loader files (worker records, talent profiles and so on), drop them in a folder on the integration server and call this service. It packs every data file in that folder into one zip, uploads it to Oracle's content server in the area HCM Data Loader reads from, and then tells HCM to import and load it.  The weak point is the shared folder. The service takes every file it finds, not just the caller's, and then empties the folder. If two jobs run close together, one can upload the other's files or delete them before they are sent. If zipping fails, the error is only logged and an empty file is still uploaded. The service also never checks whether HCM actually loaded the data: it returns as soon as the load has been requested.  In Boomi the files are passed in memory from the calling process, zipped and encoded by a script, checked in and loaded, with no server folder. We recommend adding a status check of the load so that failures are reported.

**Complexity:** medium (score 40)

## Functional Analysis

1. **Receive check-in request** — Async SOAP process with a UCM GenericRequest (createInstance).
2. **Zip and encode DAT files** — Java embedding: all \*.dat in /u01/data/domains/SOACS-Pr\_domain/osi\_file/put/ zipped to /u01/data/domains/SOACS-Pr\_domain/osi\_file/get/OSIWorker.zip; put folder emptied; zip Base64-encoded (chunked); get folder emptied.
3. **Check in to UCM** — xform\_checkin\_request\_1 copies webKey, IdcService, all Fields and File name/href from the request and sets Contents to the encoded string; GenericSoapOperation with Authorization header.
4. **Import and load** — xform\_hcm\_request: ContentId = response Field dDocName; HCMDataLoader.importAndLoadData with Authorization header.
5. **Return content ID** — Callback processResponse/contentID = ContentId.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessHDLProvABCSImpl/processhdlprovabcsimplprocess\_client\_ep | none declared |
| outbound | SOAP 1.1 over HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | HTTP Authorization header (Credentials.dvm); WSDL advertises WS-Security |
| outbound | SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCommonDataLoader/HCMDataLoader | HTTP Authorization header (Credentials.dvm) |
| outbound | SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification\_client\_ep.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Caller request to UCM check-in | GenericRequest/@webKey | GenericRequest/@webKey | passthrough | — | — | low | Callers send "cs". |
| MR-002 | Caller request to UCM check-in | Service/@IdcService | Service/@IdcService | passthrough | — | — | low | CHECKIN\_UNIVERSAL. |
| MR-003 | Caller request to UCM check-in | Document/Field\[@name\] | Document/Field | looped | — | copy name and value | low | dDocTitle, dSecurityGroup FAFusionImportExport, dDocType Document, dDocAccount hcm$/dataloader$/import$. |
| MR-004 | Caller request to UCM check-in | File/@name, File/@href | File/@name, File/@href | passthrough | — | — | low | primaryFile, e.g. Worker.zip. |
| MR-005 | Caller request to UCM check-in | File/Contents | Java output | calculated | — | Base64(zip of all .dat files) | medium | fn-zip-base64 in Boomi. |
| MR-006 | UCM response to HDL import | ContentId | GenericResponse/Service/Document/Field\[@name="dDocName"\] | conditional | @name = dDocName | — | low | — |
| MR-007 | Result to response | contentID | ContentId | passthrough | — | — | low | HDL response (request ID) is not returned. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | HDL import location | Content is checked in with dSecurityGroup FAFusionImportExport and dDocAccount hcm$/dataloader$/import$ so HCM Data Loader can read it. | High |
| BR-002 | Load after check-in | importAndLoadData is called with the UCM dDocName as ContentId. | High |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive check-in request** — From DAT generation services
2. **Zip and encode DAT files** — Java; empties the shared folder
3. **Check in to UCM** — CHECKIN\_UNIVERSAL
4. **Import and load in HCM** — importAndLoadData; not verified
5. **Notify service desk** — catchAll; empty result
6. **Return content ID** — Callback

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| HCM UCM Generic Req XML Prof | xml | prof-ucm-generic-req |
| HCM UCM Generic Resp XML Prof | xml | prof-ucm-generic-resp |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback | Sub-process start + Return Documents |
| Java ZIPtoBASE64 + server folders | fn-zip-base64 on in-memory documents |
| CHECKIN\_UMS (GenericSoapPort) | conn-hcm-ucm-soap + op-ucm-checkin |
| HCMDataLoader importAndLoadData | conn-hcm-hdl-soap + op-hdl-import-load |
| Credentials.dvm Authorization header | Connection authentication settings |
| catchAll | Try/Catch + util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| conn-hcm-ucm-soap | UCM GenericSoapPort connection |
| op-ucm-checkin | CHECKIN\_UNIVERSAL operation |
| prof-ucm-generic-req | UCM request profile |
| prof-ucm-generic-resp | UCM response profile |
| conn-hcm-hdl-soap | HCM Data Loader connection |
| op-hdl-import-load | importAndLoadData operation |
| fn-zip-base64 | Zip + Base64 script |
| proc-sub-hdl-load | Shared sub-process |

## Client questions

- **Q-001** Which HCM integration user should Boomi use to upload files and start HCM Data Loader, and does your HCM tenant accept basic \[redacted\] for it?
  **If this is not answered:** A dedicated HCM integration user with basic \[redacted\] over HTTPS, as in legacy.
- **Q-002** Should the new process wait for HCM Data Loader to finish and report rows that failed to load?
  **If this is not answered:** Poll the load status and email failures to the IT service desk list.
- **Q-003** Who should be told when an HCM data load fails?
  **If this is not answered:** Keep the "itservicedesk" list.
