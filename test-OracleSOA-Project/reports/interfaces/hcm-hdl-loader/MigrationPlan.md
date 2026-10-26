# Migration Plan — ProcessHDLProvABCSImpl

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.956Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessHDLProvABCSImpl |
| Identifier | hcm-hdl-loader |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | medium (40) |
| Components to build | 8 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/BPEL/ProcessHDLProvABCSImplProcess.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java |
| Depends on | util-send-notification |

## 2. End-to-End Process Architecture

Async BPEL composite that combines a Java embedding working on server folders with two HCM SOAP references (UCM GenericSoapPort and HCMDataLoader via MDS WSDLs) and XXSendNotification. Callers coordinate with it only through the shared folder.

Boomi sub-process "Sub: HCM HDL Zip Checkin and Load": receives the DAT document(s) and the document title/zip name from the caller, fn-zip-base64 builds the zip, op-ucm-checkin (prof-ucm-generic-req/resp) on conn-hcm-ucm-soap, op-hdl-import-load on conn-hcm-hdl-soap, optional getDataSetStatus polling, returns ContentId (and request ID); Try/Catch with util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback | Sub-process start + Return Documents |  |
| Java ZIPtoBASE64 + server folders | fn-zip-base64 on in-memory documents | Removes the race and destructive clean-up. |
| CHECKIN_UMS (GenericSoapPort) | conn-hcm-ucm-soap + op-ucm-checkin |  |
| HCMDataLoader importAndLoadData | conn-hcm-hdl-soap + op-hdl-import-load | Add status polling. |
| Credentials.dvm Authorization header | Connection authentication settings | No credential in process data. |
| catchAll | Try/Catch + util-send-notification |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive check-in request | Async SOAP process with a UCM GenericRequest (createInstance). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/WSDLs/ProcessHDLProvABCSImplProcess.wsdl |
| 2 | Zip and encode DAT files | Java embedding: all *.dat in /u01/data/domains/SOACS-Pr_domain/osi_file/put/ zipped to /u01/data/domains/SOACS-Pr_domain/osi_file/get/OSIWorker.zip; put folder emptied; zip Base64-encoded (chunked); get folder emptied. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java |
| 3 | Check in to UCM | xform_checkin_request_1 copies webKey, IdcService, all Fields and File name/href from the request and sets Contents to the encoded string; GenericSoapOperation with Authorization header. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/Transformations/xform_checkin_request_1.xsl |
| 4 | Import and load | xform_hcm_request: ContentId = response Field dDocName; HCMDataLoader.importAndLoadData with Authorization header. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/Transformations/xform_hcm_request.xsl |
| 5 | Return content ID | Callback processResponse/contentID = ContentId. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/BPEL/ProcessHDLProvABCSImplProcess.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| HCM UCM Generic Req XML Prof | xml | GenericRequest/@webKey | string | yes |
| HCM UCM Generic Req XML Prof | xml | GenericRequest/Service/@IdcService | string | yes |
| HCM UCM Generic Req XML Prof | xml | GenericRequest/Service/Document/Field | string (repeating, @name) | yes |
| HCM UCM Generic Req XML Prof | xml | GenericRequest/Service/Document/File/@name | string | yes |
| HCM UCM Generic Req XML Prof | xml | GenericRequest/Service/Document/File/@href | string | yes |
| HCM UCM Generic Req XML Prof | xml | GenericRequest/Service/Document/File/Contents | base64 | yes |
| HCM UCM Generic Resp XML Prof | xml | GenericResponse/Service/Document/Field | string (repeating, @name) | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | GenericRequest/@webKey | GenericRequest/@webKey | passthrough |  |  | low | Callers send "cs". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/Transformations/xform_checkin_request_1.xsl | Caller request to UCM check-in |
| MR-002 | Service/@IdcService | Service/@IdcService | passthrough |  |  | low | CHECKIN_UNIVERSAL. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hdl_provider_req.xsl | Caller request to UCM check-in |
| MR-003 | Document/Field[@name] | Document/Field | looped |  | copy name and value | low | dDocTitle, dSecurityGroup FAFusionImportExport, dDocType Document, dDocAccount hcm$/dataloader$/import$. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hdl_provider_req.xsl | Caller request to UCM check-in |
| MR-004 | File/@name, File/@href | File/@name, File/@href | passthrough |  |  | low | primaryFile, e.g. Worker.zip. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/Transformations/xform_checkin_request_1.xsl | Caller request to UCM check-in |
| MR-005 | File/Contents | Java output | calculated |  | Base64(zip of all .dat files) | medium | fn-zip-base64 in Boomi. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java | Caller request to UCM check-in |
| MR-006 | ContentId | GenericResponse/Service/Document/Field[@name="dDocName"] | conditional | @name = dDocName |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/Transformations/xform_hcm_request.xsl | UCM response to HDL import |
| MR-007 | contentID | ContentId | passthrough |  |  | low | HDL response (request ID) is not returned. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/BPEL/ProcessHDLProvABCSImplProcess.bpel | Result to response |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | HDL import location | Content is checked in with dSecurityGroup FAFusionImportExport and dDocAccount hcm$/dataloader$/import$ so HCM Data Loader can read it. | High | Fixed fields in prof-ucm-generic-req. |
| BR-002 | Load after check-in | importAndLoadData is called with the UCM dDocName as ContentId. | High | op-hdl-import-load. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessHDLProvABCSImpl/processhdlprovabcsimplprocess_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/processhdlprovabcsimplprocess_client_ep.wsdl |
| outbound SOAP 1.1 over HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | HTTP Authorization header (Credentials.dvm); WSDL advertises WS-Security | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/GenericSoapPort.wsdl |
| outbound SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCommonDataLoader/HCMDataLoader | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/HCMDataLoader.wsdl |
| outbound SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/composite.xml |
| File system (Java, not an adapter) | Reads and deletes [redacted]; writes and deletes [redacted].zip. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java | Zip all .dat files of a folder, Base64-encode (chunked, 76-char lines), delete folder contents. | fn-zip-base64 (Groovy) working on the DAT documents passed in; no file system. | Medium |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/Transformations/xform_checkin_request_1.xsl | Copy the caller's GenericRequest and insert the encoded contents. | Map or Message shape into prof-ucm-generic-req. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/Transformations/xform_hcm_request.xsl | ContentId = dDocName from the UCM response. | Map from prof-ucm-generic-resp. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll: notify EmailDL itservicedesk, callback with empty result. Java errors swallowed. |
| Retries | None. |
| Compensation | None; deleted files cannot be recovered. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/HCM/GenericSoapPort.wsdl, HCMDataLoader.wsdl | mds | yes |
| Credentials.dvm (Authorization) | dvm | yes |
| EmailDL.dvm | dvm | yes |
| commons-io / commons-codec | library | no |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud UCM | SOAP / HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | Oracle (TaskUs tenant) |
| Oracle HCM Cloud HCM Data Loader | SOAP / HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCommonDataLoader/HCMDataLoader | Oracle (TaskUs tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | UCM endpoint | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | [redacted].wsdl |
| PROD | HDL endpoint | https://eetm.hcm.us6.oraclecloud.com:[redacted] | [redacted].wsdl |
| PROD | DAT folder | [redacted] | [redacted].bpel |

## 14. Boomi Migration Readiness Summary

This is the shared service that sends data files into Oracle HCM. Other integrations prepare HCM Data Loader files (worker records, talent profiles and so on), drop them in a folder on the integration server and call this service. It packs every data file in that folder into one zip, uploads it to Oracle's content server in the area HCM Data Loader reads from, and then tells HCM to import and load it.

The weak point is the shared folder. The service takes every file it finds, not just the caller's, and then empties the folder. If two jobs run close together, one can upload the other's files or delete them before they are sent. If zipping fails, the error is only logged and an empty file is still uploaded. The service also never checks whether HCM actually loaded the data: it returns as soon as the load has been requested.

In Boomi the files are passed in memory from the calling process, zipped and encoded by a script, checked in and loaded, with no server folder. We recommend adding a status check of the load so that failures are reported.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | HCM UCM Generic Req XML Prof | prof-ucm-generic-req | profile.xml | — |
| 2 | HCM UCM Generic Resp XML Prof | prof-ucm-generic-resp | profile.xml | — |
| 3 | HCM Data Loader SOAP Conn | conn-hcm-hdl-soap | connector-settings | — |
| 4 | HCM UCM GenericSoap Conn | conn-hcm-ucm-soap | connector-settings | — |
| 5 | HCM Data Loader IMPORT Op | op-hdl-import-load | connector-action | conn-hcm-hdl-soap |
| 6 | HCM UCM File CHECKIN Op | op-ucm-checkin | connector-action | conn-hcm-ucm-soap |
| 7 | DAT Zip and Base64 Script | fn-zip-base64 | transform.function | — |
| 8 | Sub: HCM HDL Zip Checkin and Load | proc-sub-hdl-load | process | op-ucm-checkin, prof-ucm-generic-req, prof-ucm-generic-resp, op-hdl-import-load, fn-zip-base64, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/BPEL/ProcessHDLProvABCSImplProcess.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/Schemas/ProcessHDLProvABCSImplProcess.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/WSDLs/ProcessHDLProvABCSImplProcess.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/Transformations/xform_checkin_request_1.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/Transformations/xform_hcm_request.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/GenericSoapPort.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/HCMDataLoader.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/processhdlprovabcsimplprocess_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hdl_provider_req.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/ProcessDATGenerationABCSImpl_prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Adapters/writeTalentProfile_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/Adapters/WriteHDL_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/composite.xml

## Open questions

### Still outstanding

REVIEW-NEEDED: Which HCM integration user should Boomi use to upload files and start HCM Data Loader, and does your HCM tenant accept basic [redacted] for it? (Legacy sends one stored credential as a basic [redacted] header while Oracle's service description asks for message-level security; the Boomi connections cannot be configured without this.)
  - If this is not answered: A dedicated HCM integration user with basic [redacted] over HTTPS, as in legacy.

REVIEW-NEEDED: Should the new process wait for HCM Data Loader to finish and report rows that failed to load? (Today the load is only requested; failed loads are never reported.)
  - If this is not answered: Poll the load status and email failures to the IT service desk list.

REVIEW-NEEDED: Who should be told when an HCM data load fails? (Errors go to the "itservicedesk" list today; HRIS may be the right owner.)
  - If this is not answered: Keep the "itservicedesk" list.

