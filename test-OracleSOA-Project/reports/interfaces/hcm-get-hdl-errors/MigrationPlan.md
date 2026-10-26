# Migration Plan — getHDLErrorsProvABCSImpl

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.073Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | getHDLErrorsProvABCSImpl |
| Identifier | hcm-get-hdl-errors |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | simple (18) |
| Components to build | 5 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/BPEL/getHDLErrorsProvABCSImpl.bpel |
| Depends on | Nothing |

## 2. End-to-End Process Architecture

Async BPEL composite with one SOAP reference to BI Publisher PublicReportService (MDS WSDL) and a Java embedding for Base64 decoding; no notification on error.

Boomi sub-process "Sub: HCM Get HDL Errors": map ContentId into prof-bip-runreport-req, op-bip-run-report on conn-hcm-bip-soap, Data Process Base64 decode, strip comment, return prof-hdl-status-report; Try/Catch returning an error status.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback | Sub-process start + Return Documents |  |
| xform_input_report.xsl | Map into prof-bip-runreport-req | No credentials in the body. |
| ErrorReportService runReport | conn-hcm-bip-soap + op-bip-run-report |  |
| Java Base64DecoderClass | Data Process Base64 decode |  |
| (no fault handler) | Try/Catch | New behaviour. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive content ID | Async SOAP process (createInstance) with ContentId. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/Schemas/getHDLErrorsProvABCSImpl.xsd |
| 2 | Build report request | xform_input_report: report path, ContentID parameter, userID/password from Credentials.dvm. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/Transformations/xform_input_report.xsl |
| 3 | Run report | PublicReportService.runReport with HTTP Authorization header. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/PublicReportService.wsdl |
| 4 | Decode output | Java Base64 decode of reportBytes; strip text up to the first "-->"; parseEscapedXML. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/SCA-INF/src/com/osi/Base64DecoderClass.java |
| 5 | Return errors | Callback with DATA_DS. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/BPEL/getHDLErrorsProvABCSImpl.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| HCM BIP runReport Req XML Prof | xml | runReport/reportRequest/reportAbsolutePath | string | yes |
| HCM BIP runReport Req XML Prof | xml | runReport/reportRequest/parameterNameValues/item/name | string | yes |
| HCM BIP runReport Req XML Prof | xml | runReport/reportRequest/parameterNameValues/item/values/item | string | yes |
| HCM BIP runReport Req XML Prof | xml | runReport/userID | string | no |
| HCM BIP runReport Req XML Prof | xml | runReport/password | string | no |
| HCM HDL Status Report XML Prof | xml | DATA_DS/CONTENTID | string | no |
| HCM HDL Status Report XML Prof | xml | DATA_DS/G_1 | repeating | no |
| HCM HDL Status Report XML Prof | xml | DATA_DS/G_1/KEY_SOURCE_OWNER | string | no |
| HCM HDL Status Report XML Prof | xml | DATA_DS/G_1/KEY_SOURCE_ID | string | no |
| HCM HDL Status Report XML Prof | xml | DATA_DS/G_1/MSG_TEXT | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | reportRequest/reportAbsolutePath | (none) | hardcoded |  | "/Human Capital Management/TaskUs HDL Status Report.xdo" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/Transformations/xform_input_report.xsl | Content ID to runReport request |
| MR-002 | parameterNameValues/item (name, label ContentID; UIType Text; dataType xsd:string) | (none) | hardcoded |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/Transformations/xform_input_report.xsl | Content ID to runReport request |
| MR-003 | parameterNameValues/item/defaultValue and values/item | process/ContentId | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/Transformations/xform_input_report.xsl | Content ID to runReport request |
| MR-004 | userID / password | Credentials.dvm | transformed |  | DVM lookup | low | Drop in Boomi; use connection authentication. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/Transformations/xform_input_report.xsl | Content ID to runReport request |
| MR-005 | DATA_DS | runReportReturn/reportBytes | calculated |  | Base64 decode, strip leading comment, parse XML | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/BPEL/getHDLErrorsProvABCSImpl.bpel | Report output to response |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | HDL error source | Errors for a load are read from the custom BI Publisher report /Human Capital Management/TaskUs HDL Status Report.xdo keyed by UCM ContentID. | Medium | op-bip-run-report with prof-bip-runreport-req. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/getHDLErrorsProvABCSImpl/gethdlerrorsprovabcsimpl_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/gethdlerrorsprovabcsimpl_client_ep.wsdl |
| outbound SOAP 1.1 over HTTPS | https://eetm.bi.us6.oraclecloud.com/xmlpserver/services/PublicReportService | HTTP Authorization header and userID/password in the SOAP body (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/PublicReportService.wsdl |

## 8. Sub-Process Call Hierarchy

> **Not available.** This interface calls no sub-processes.

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/SCA-INF/src/com/osi/Base64DecoderClass.java | Base64 decode of the report bytes (platform default charset). | Data Process Base64 decode, then XML profile. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/Transformations/xform_input_report.xsl | runReport request with ContentID parameter and credentials. | Map into prof-bip-runreport-req without credential fields. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | None (no faultHandlers). |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/HCM/PublicReportService.wsdl | mds | yes |
| Credentials.dvm | dvm | yes |
| BI Publisher report /Human Capital Management/TaskUs HDL Status Report.xdo | other | no |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud BI Publisher | SOAP / HTTPS | https://eetm.bi.us6.oraclecloud.com/xmlpserver/services/PublicReportService | Oracle (TaskUs tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | BI Publisher endpoint | https://eetm.bi.us6.oraclecloud.[redacted] | [redacted].wsdl |

## 14. Boomi Migration Readiness Summary

After data is loaded into HCM, the integrations need to know which records failed. This service asks Oracle's reporting engine (BI Publisher) to run the TaskUs "HDL Status Report" for a given load and returns the list of errors: which record and the error message.

It is simple, but the legacy version sends a stored user name and password inside the report request as well as in the request header, writes the decoded report (which contains employee identifiers) to the server log, and has no error handling: if the report call fails the caller never gets an answer. The Boomi version should use the connection's authentication only, keep data out of logs and always return a result.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | HCM BIP runReport Req XML Prof | prof-bip-runreport-req | profile.xml | — |
| 2 | HCM HDL Status Report XML Prof | prof-hdl-status-report | profile.xml | — |
| 3 | HCM BI Publisher PublicReportService Conn | conn-hcm-bip-soap | connector-settings | — |
| 4 | HCM BIP Report RUN Op | op-bip-run-report | connector-action | conn-hcm-bip-soap |
| 5 | Sub: HCM Get HDL Errors | proc-sub-hcm-get-hdl-errors | process | op-bip-run-report, prof-bip-runreport-req, prof-hdl-status-report |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/BPEL/getHDLErrorsProvABCSImpl.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/Schemas/getHDLErrorsProvABCSImpl.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/Transformations/xform_input_report.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/getHDLErrorsProvABCSImpl/SOA/SCA-INF/src/com/osi/Base64DecoderClass.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/PublicReportService.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/gethdlerrorsprovabcsimpl_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessUpdateExternalIdentifier/SOA/composite.xml

## Open questions

### Still outstanding

REVIEW-NEEDED: Will the custom BI Publisher report "TaskUs HDL Status Report" stay in HCM and remain available to the integration user after the migration? (The service depends entirely on this report, which is not part of the export.)
  - If this is not answered: Yes, unchanged at the same path.

REVIEW-NEEDED: Which HCM integration user should run the HDL status report from Boomi? (Legacy sends a stored user name and password inside the report request; the Boomi connection needs a named integration user with access to the report.)
  - If this is not answered: The same HCM integration user as the other HCM connections.

