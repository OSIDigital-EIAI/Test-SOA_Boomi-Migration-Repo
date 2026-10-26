# Migration Plan — ProcessPBLProvABCSImpl

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.111Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessPBLProvABCSImpl |
| Identifier | hcm-pbl-allowances |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | medium (42) |
| Components to build | 4 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/BPEL/ProcessPBLProvABCSImpl.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/ProcessPBLProvABCSImpl_prod_cfgplan.xml |
| Depends on | hcm-hdl-loader, emp-campaign-extract-sch, util-send-notification, hcm-dat-generation |

## 2. End-to-End Process Architecture

Async BPEL composite: XSLT to an NXSD flat file written by the File adapter, Java Base64 encoding with folder clean-up, UCM GenericSoapPort (MDS WSDL) and FlowActionsService (URL replaced by the config plan), XXSendNotification on error.

Boomi sub-process "Sub: HCM PBL Allowance Load": map-ebm-to-pbl into prof-pbl-dat in memory, Base64 encode, op-ucm-checkin (hcm-hdl-loader), map-ucm-to-pbl-submitflow, op-hcm-flow-submit (emp-campaign-extract-sch), optional flow status polling, return a result; Try/Catch with util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback | Sub-process start + Return Documents | Return a result on success and failure. |
| XForm_EBO_To_DAT + File adapter | map-ebm-to-pbl + prof-pbl-dat (in memory) | Fix guards. |
| Java FiletoBase64Class | Base64 encode step | No folder. |
| CheckIN_UCM | op-ucm-checkin (hcm-hdl-loader) |  |
| PBLSubmitFlowService submitFlow | map-ucm-to-pbl-submitflow + op-hcm-flow-submit (emp-campaign-extract-sch) | Environment extension for the URL. |
| catchAll | Try/Catch + util-send-notification |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive employees | Async SOAP process with EmployeeEBM EmployeeRequest (createInstance). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/WSDLs/ProcessPBLProvABCSImpl.wsdl |
| 2 | Build PBL rows | XForm_EBO_To_DAT: Taxable Allowance row per employee; PH de minimis tier rows. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_EBO_To_DAT.xsl |
| 3 | Any rows? | count(PBL) > 0, otherwise nothing is done (in practice always true). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/BPEL/ProcessPBLProvABCSImpl.bpel |
| 4 | Write PBL file | File adapter writes PBLFile.dat with header line to the LogicalPath (PROD: /u01/data/domains/SOACS-Pr_domain/osi_payroll/put/). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/ProcessPBLProvABCSImpl_prod_cfgplan.xml |
| 5 | Encode file | Java FiletoBase64Class: Base64 of PBLFile.dat (US-ASCII), then clean the folder; errors only audited. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/SCA-INF/src/com/osi/FiletoBase64Class.java |
| 6 | Check in to UCM | GenericSoapOperation CHECKIN_UNIVERSAL with Authorization header (Credentials.dvm). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_DAT_To_UCM.xsl |
| 7 | Submit payroll flow | FlowActionsService.submitFlow "Load and Transfer Batch" with the UCM content ID; Authorization header. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_UCM_To_PBL.xsl |
| 8 | Callback | processResponse with empty result. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/BPEL/ProcessPBLProvABCSImpl.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/TaskName | string | yes |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/TaskActionName | string | yes |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/Reference | string | yes |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/EffectiveStartDate | string | yes |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/EffectiveEndDate | string | no |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/PersonNumber | string | yes |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/AssignmentNumber | string | yes |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/TermsNumber | string | no |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/RelationshipNumber | string | no |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/Payroll | string | no |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/Reason | string | no |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/Subpriority | string | no |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/PayValue | string | no |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/Amount | string | yes |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/Periodicity | string | no |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/FullTimeEquivalent | string | no |
| HCM PBL Element Entry DAT Prof | flatfile | PBLDatRequest/PBL/OverrideEntry | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | TaskName / TaskActionName | (none) | hardcoded |  | "Element Entry" / "Create Element Entry" | low | All rows. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_EBO_To_DAT.xsl | Employee EBM to PBL element entries |
| MR-002 | EffectiveStartDate | Employee/EmploymentAssignments/EffectiveStartDate | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_EBO_To_DAT.xsl | Employee EBM to PBL element entries |
| MR-003 | PersonNumber | Employee/Person/NationalID_Type/SSSNumber | passthrough |  |  | low | SSS number used as person number. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_EBO_To_DAT.xsl | Employee EBM to PBL element entries |
| MR-004 | AssignmentNumber | Employee/Person/NationalID_Type/SSSNumber | calculated |  | "ASSIGNMENT_" + SSSNumber | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_EBO_To_DAT.xsl | Employee EBM to PBL element entries |
| MR-005 | PBL row: Reference "Taxable Allowance" / Amount | Employee/EmployeeAllowanceType/AllowanceType | conditional | AllowanceType != "" or != "0" (always true) | AllowanceType | medium | Every employee. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_EBO_To_DAT.xsl | Employee EBM to PBL element entries |
| MR-006 | PBL row: Reference "De Minimis Rice" / Amount | Employee/EmployeeAllowanceType/Allowance | conditional | CountryCode = PH | Allowance if <= 1500, else 1500 | medium | Guard on Allowance also always true. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_EBO_To_DAT.xsl | Employee EBM to PBL element entries |
| MR-007 | PBL row: Reference "De Minimis Christmas" / Amount | Employee/EmployeeAllowanceType/Allowance | conditional | PH and Allowance > 1500 | Allowance - 1500 if <= 1900, else 400 | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_EBO_To_DAT.xsl | Employee EBM to PBL element entries |
| MR-008 | PBL row: Reference "De Minimis Uniform" / Amount | Employee/EmployeeAllowanceType/Allowance | conditional | PH and Allowance > 1900 | Allowance - 1900 if <= 2300, else 400 | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_EBO_To_DAT.xsl | Employee EBM to PBL element entries |
| MR-009 | PBL row: Reference "De Minimis Laundry" / Amount | Employee/EmployeeAllowanceType/Allowance | conditional | PH and Allowance > 2300 | Allowance - 2300 | medium | No upper limit. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_EBO_To_DAT.xsl | Employee EBM to PBL element entries |
| MR-010 | EffectiveEndDate, TermsNumber, RelationshipNumber, Payroll, Reason, Subpriority, PayValue, Periodicity, FullTimeEquivalent, OverrideEntry | (none) | hardcoded |  | empty | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_EBO_To_DAT.xsl | Employee EBM to PBL element entries |
| MR-011 | flowName | (none) | hardcoded |  | "Load and Transfer Batch" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_UCM_To_PBL.xsl | UCM response to payroll flow submission |
| MR-012 | parameterValues[Batch] | current dateTime | calculated |  | "PFB" + yyyyMMddHHmmss | low | Server time zone. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_UCM_To_PBL.xsl | UCM response to payroll flow submission |
| MR-013 | parameterValues[Content Id] | GenericResponse Field[@name="dDocName"] | conditional | @name = dDocName |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_UCM_To_PBL.xsl | UCM response to payroll flow submission |
| MR-014 | flowInstanceName | current dateTime | calculated |  | "PF" + yyyyMMddHHmmss | low | Two calls in the same second collide. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_UCM_To_PBL.xsl | UCM response to payroll flow submission |
| MR-015 | legislativeDataGroupName / recurringFlag | (none) | hardcoded |  | "PH Legislative Data Group" / false | low | Non-PH Taxable Allowance rows also go to the PH LDG. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_UCM_To_PBL.xsl | UCM response to payroll flow submission |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Taxable allowance entry | Every employee gets a Taxable Allowance entry with AllowanceType as amount (intended only when non-empty and non-zero). | High | map-ebm-to-pbl with a corrected guard. |
| BR-002 | PH de minimis split | Rice up to 1500; Christmas up to 400 more; Uniform up to 400 more; remainder above 2300 as Laundry. | High | Thresholds in a cross reference table; map logic or Groovy function. |
| BR-003 | Payroll load flow | Load and Transfer Batch submitted for the PH Legislative Data Group with batch name PFB<timestamp>. | High | map-ucm-to-pbl-submitflow + op-hcm-flow-submit. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessPBLProvABCSImpl/processpblprovabcsimpl_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/processpblprovabcsimpl_client_ep.wsdl |
| outbound File (SOA server) | /u01/data/domains/SOACS-Pr_domain/osi_payroll/put/PBLFile.dat | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Adapters/FileWriteService_file.jca |
| outbound SOAP 1.1 over HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/GenericSoapPort.wsdl |
| outbound SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com/hcmProcFlowCoreController/FlowActionsService | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/ProcessPBLProvABCSImpl_prod_cfgplan.xml |
| File adapter FileWriteService | JNDI eis/FileAdapter; LogicalDirectory LogicalPath (PROD [redacted]); FileNamingConvention PBLFile.dat; Append false; NumberMessages 1; NXSD FileWrite_Request.xsd (pipe-delimited, header line, US-ASCII). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Adapters/FileWriteService_file.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-hdl-loader | UCM check-in components (op-ucm-checkin, prof-ucm-generic-req/resp) | synchronous |
| emp-campaign-extract-sch | FlowActionsService components (op-hcm-flow-submit, prof-hcm-flow-submit-req) | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/SCA-INF/src/com/osi/FiletoBase64Class.java | Base64-encode PBLFile.dat and empty the folder. | Base64 encode of the in-memory document (fn-zip-base64 without zipping, or Data Process). | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_EBO_To_DAT.xsl | Allowance rows with tiered de minimis split. | map-ebm-to-pbl (with corrected guards; thresholds in a cross reference). | High |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_UCM_To_PBL.xsl | submitFlow request with timestamped batch and instance names. | map-ucm-to-pbl-submitflow. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll: notify EmailDL itservicedesk; no callback. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/EnterpriseObjectLibrary/Core/EBO/Employee/V1/EmployeeEBM.xsd (PROD_MDS snapshot) | mds | yes |
| oramds:/apps/Provider/HCM/GenericSoapPort.wsdl | mds | yes |
| Credentials.dvm / EmailDL.dvm | dvm | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud UCM | SOAP / HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | Oracle (TaskUs tenant) |
| Oracle HCM Cloud FlowActionsService | SOAP / HTTPS | https://eetm.hcm.us6.oraclecloud.com/hcmProcFlowCoreController/FlowActionsService | Oracle (TaskUs tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| DEV | FlowActionsService | https://ebly-dev1.hcm.us2.oraclecloud.[redacted] | [redacted].xml |
| PROD | FlowActionsService | https://eetm.hcm.us6.oraclecloud.[redacted] | [redacted].xml |
| PROD | PBL folder | [redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

This service sets up a new employee's allowances in payroll. For every employee it receives, it creates a "Taxable Allowance" entry with the allowance amount. For employees in the Philippines it also takes the de minimis (tax-free) allowance and splits it across benefit types in a fixed order: Rice up to 1,500, then Christmas up to 400 more, then Uniform up to 400 more, and anything above 2,300 as Laundry. It writes these entries into a payroll batch file, uploads it to HCM and starts the payroll flow that loads and transfers the batch.

The payroll impact makes accuracy important, and there are defects to settle before rebuilding. The checks that should skip employees without an allowance are written so they are always true, so entries are created even when the amount is empty or zero. The employee number in the file is taken from the SSS number field. The thresholds and benefit names are fixed in the code. The service also never checks whether the payroll flow succeeded, and on error it emails the service desk but never answers the caller.

In Boomi this becomes a sub-process that builds the file in memory, uses the shared HCM upload and flow-submission components, and returns a real result.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | HCM PBL Element Entry DAT Prof | prof-pbl-dat | profile.flatfile | — |
| 2 | Map \| Employee EBM to PBL Allowance Rows | map-ebm-to-pbl | transform.map | prof-employee-ebm-xml, prof-pbl-dat |
| 3 | Map \| UCM ContentId to PBL SubmitFlow | map-ucm-to-pbl-submitflow | transform.map | prof-ucm-generic-resp, prof-hcm-flow-submit-req |
| 4 | Sub: HCM PBL Allowance Load | proc-sub-hcm-pbl-allowances | process | map-ebm-to-pbl, op-ucm-checkin, fn-zip-base64, map-ucm-to-pbl-submitflow, op-hcm-flow-submit, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/BPEL/ProcessPBLProvABCSImpl.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/ProcessPBLProvABCSImpl_prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Adapters/FileWriteService_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Schemas/FileWrite_Request.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Schemas/ProcessPBLProvABCSImpl.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/WSDLs/ProcessPBLProvABCSImpl.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/SCA-INF/src/com/osi/FiletoBase64Class.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_EBO_To_DAT.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_DAT_To_UCM.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/SOA/Transformations/XForm_UCM_To_PBL.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessPBLProvABCSImpl/samplePBL.dat
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/processpblprovabcsimpl_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/GenericSoapPort.wsdl

## Open questions

### Still outstanding

REVIEW-NEEDED: Are the Philippine de minimis ceilings still Rice 1,500, Christmas 400, Uniform 400 with the remainder as Laundry, and are these the correct element names in payroll? (The split is fixed in code and directly affects pay and tax; it must be confirmed before rebuilding.)
  - If this is not answered: Keep the current ceilings and names, held in a configurable table.

REVIEW-NEEDED: Should employees with no allowance or a zero allowance be skipped, rather than given an empty or zero allowance entry? (The current check meant to skip them never works, so empty or zero entries are sent to payroll.)
  - If this is not answered: Skip empty and zero amounts.

REVIEW-NEEDED: Is the employee's SSS number always the same as their HCM person number and assignment number suffix? (The payroll file identifies employees by SSS number; if they differ, entries load against the wrong or no person.)
  - If this is not answered: Yes, as today.

REVIEW-NEEDED: Should non-Philippine employees' taxable allowances also be loaded, given that every batch is submitted to the PH Legislative Data Group? (Taxable Allowance rows are created for all countries but loaded only into the PH data group.)
  - If this is not answered: Limit the service to PH employees.

REVIEW-NEEDED: Should the process wait for the payroll load to finish and report failures to payroll? (Today the flow is only submitted; failed loads are not reported.)
  - If this is not answered: Poll the flow status and email failures to the IT service desk list.

