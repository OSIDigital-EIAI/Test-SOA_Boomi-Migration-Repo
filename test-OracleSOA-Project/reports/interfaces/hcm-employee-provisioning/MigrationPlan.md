# Migration Plan — ProcessEmployeeHCMProvABCSImpl

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.333Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessEmployeeHCMProvABCSImpl |
| Identifier | hcm-employee-provisioning |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | medium (44) |
| Components to build | 3 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/BPEL/ProcessEmployeeHCMProvABCSImpl.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeHCMProvABCSImpl/SOA/BPEL/ProcessEmployeeHCMProvABCSImpl.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeHCMProvABCSImpl/ProcessEmployeeHCMProvABCSImpl_prod_cfgplan.xml |
| Depends on | newhire-emp-sch, hcm-dat-generation, util-send-notification |

## 2. End-to-End Process Architecture

BPEL composite with an early reply, HCM REST references (GetEmployee, GetEmployeeLov) and an async reference to ProcessDATGenerationABCSImpl; eight further references (Java HCM relay, assignments, email generation, Google create/update, PBL, Interviewed, HCMExtract) are present but skipped. The Employee copy waits for DAT generation before replying.

Boomi sub-process "Sub: HCM Employee Validate and Provision": op-hcm-emps-get (newhire-emp-sch) for the supervisor, Decision, op-hcm-lov-get + map-salutation-lov, Process Call hcm-dat-generation, return its TransmissionAck; Try/Catch with util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive + early reply | Sub-process start; Return Documents after processing | Return the real outcome. |
| GetEmployee (emps?q=PersonNumber) | op-hcm-emps-get on conn-hcm-rest (newhire-emp-sch) |  |
| GetEmployee limit=1 + GetEmployeeLov | op-hcm-lov-get (href from emps links) | Could be replaced by a direct LOV resource. |
| xform_SalutationLOV | map-salutation-lov |  |
| ProcessDATGeneration (no wait) | Process Call hcm-dat-generation (synchronous) |  |
| Disabled references | (not migrated) |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive hire | Synchronous-style SOAP process with EmployeeEBM. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/WSDLs/ProcessEmployeeHCMProvABCSImpl.wsdl |
| 2 | Acknowledge | Reply candidateID + "Employee details received successfully". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/BPEL/ProcessEmployeeHCMProvABCSImpl.bpel |
| 3 | Check supervisor | HCM REST emps?q=PersonNumber=<SupervisorEmpID>; count > 0 to continue. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/BPEL/ProcessEmployeeHCMProvABCSImpl.bpel |
| 4 | Supervisor missing | EmailDL hris: "SuperviserID#<id> is not existed in HCM" with candidate ID; exit. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/BPEL/ProcessEmployeeHCMProvABCSImpl.bpel |
| 5 | Get salutation list | GET emps?limit=1, href of link SalutationLOV, GET LOV items. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/Transformations/xform_get_employeeLov.xsl |
| 6 | Convert salutation | Prefix := LookupCode where Meaning = Prefix. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/Transformations/xform_SalutationLOV.xsl |
| 7 | Create worker | Invoke ProcessDATGenerationABCSImpl with the enriched EBM (no wait). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/BPEL/ProcessDATGenerationABCSImpl.bpel |

## 4. Payload Profiles

> **Not available.** No profile components owned here; uses the EBM and ack profiles of hcm-dat-generation and the HCM emps JSON profile of newhire-emp-sch.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | Employee/Person/Prefix | SalutationLOV items LookupCode | conditional | Meaning = input Prefix |  | low | Empty if no match. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/Transformations/xform_SalutationLOV.xsl | Salutation to HCM lookup code |
| MR-002 | TransmissionAck/candidateID | Employee/CandidateID | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/BPEL/ProcessEmployeeHCMProvABCSImpl.bpel | Acknowledgement |
| MR-003 | TransmissionAck/status | (none) | hardcoded |  | "Employee details received successfully" | low | Sent before processing. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/BPEL/ProcessEmployeeHCMProvABCSImpl.bpel | Acknowledgement |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Supervisor must exist | A hire is only forwarded when HCM has a person with PersonNumber = SupervisorEmpID. | High | op-hcm-emps-get (newhire-emp-sch) + Decision on count. |
| BR-002 | Salutation code | Salutation text converted to the HCM SalutationLOV LookupCode. | Low | op-hcm-lov-get + map-salutation-lov. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 (request/reply) | processemployeehcmprovabcsimpl_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/composite.xml |
| outbound REST over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCoreApi/resources/latest/emps?q=PersonNumber=<SupervisorEmpID> and ?limit=1 | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/ProcessEmployeeHCMProvABCSImpl_prod_cfgplan.xml |
| outbound REST over HTTPS | SalutationLOV href returned by HCM | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/Transformations/xform_get_employeeLov.xsl |
| outbound SOAP async (composite, no wait) | oramds:/apps/Provider/HCM/processdatgenerationabcsimpl_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/composite.xml |
| REST references GetEmployee / GetEmployeeLov | GetEmployee base from cfgplan (DEV ebly-dev1.hcm.us2 in composite, PROD eetm.hcm.us6); request URI overridden with Credentials.dvm HCMURL; GetEmployeeLov location "dummy" overridden by the LOV href. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/ProcessEmployeeHCMProvABCSImpl_prod_cfgplan.xml |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| newhire-emp-sch | HCM REST connection and emps GET operation (shared) | synchronous |
| hcm-dat-generation | Sub: HCM Worker DAT Generation | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/Transformations/xform_get_employeeLov.xsl | Extract the SalutationLOV link href. | JSON profile path on op-hcm-emps-get response. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/Transformations/xform_SalutationLOV.xsl | Meaning -> LookupCode for the salutation. | map-salutation-lov. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/Transformations/xform_getemployeemanager_assignments.xsl | Extract the assignments href (used only by the disabled path). | Not needed. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll: notify EmailDL notification, second reply. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| EmployeeEBM.xsd (PROD_MDS snapshot) | mds | yes |
| Credentials.dvm (HCMURL, Authorization) | dvm | yes |
| EmailDL.dvm | dvm | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud REST | HTTPS REST | https://eetm.hcm.us6.oraclecloud.com:443/hcmCoreApi/resources/latest/emps | Oracle (TaskUs tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| DEV | GetEmployee | https://ebly-dev1.hcm.us2.oraclecloud.com:[redacted]?limit=1 | [redacted].xml |
| PROD | GetEmployee | https://eetm.hcm.us6.oraclecloud.com:[redacted]?limit=1 | [redacted].xml |

## 14. Boomi Migration Readiness Summary

This service is the gate between the recruiting systems and HCM. For each new hire it immediately tells the caller "Employee details received successfully", then checks that the hire's supervisor exists in HCM. If not, it emails the HRIS list ("supervisor ID ... does not exist in HCM") and stops, so the hire is not created until someone fixes the data and resends it. Otherwise it converts the salutation into the code HCM expects and passes the hire to the worker-creation service.

Because the caller is answered before any work is done, it never learns whether the hire was created. Worker creation is also called without waiting, so failures there are only visible through emails. Most of the composite is switched-off code from earlier designs (direct HCM creation through a Java service, Google account creation, payroll allowances, updating the Interviewed system), and one of those switched-off references holds an Interviewed API key in its web address, which should be rotated.

In Boomi this becomes a small sub-process: supervisor check, salutation lookup and a Process Call to hcm-dat-generation, returning the real outcome.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | HCM LOV By Href GET Op | op-hcm-lov-get | connector-action | conn-hcm-rest |
| 2 | Map \| HCM Salutation LOV to Prefix Code | map-salutation-lov | transform.map | prof-employee-ebm-xml |
| 3 | Sub: HCM Employee Validate and Provision | proc-sub-hcm-employee-provisioning | process | op-hcm-emps-get, op-hcm-lov-get, map-salutation-lov, proc-sub-hcm-dat-generation, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/BPEL/ProcessEmployeeHCMProvABCSImpl.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeHCMProvABCSImpl/SOA/BPEL/ProcessEmployeeHCMProvABCSImpl.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeHCMProvABCSImpl/ProcessEmployeeHCMProvABCSImpl_prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/ProcessEmployeeHCMProvABCSImpl_prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/WSDLs/ProcessEmployeeHCMProvABCSImpl.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/Schemas/ProcessEmployeeHCMProvABCSImpl.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/Transformations/xform_SalutationLOV.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/Transformations/xform_get_employeeLov.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/Transformations/xform_getemployeemanager_assignments.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/EnterpriseObjectLibrary/Core/EBO/Employee/V1/EmployeeEBM.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/BPEL/ProcessDATGenerationABCSImpl.bpel

## Open questions

### Still outstanding

REVIEW-NEEDED: Should the recruiting systems be told whether each hire was actually created in HCM, rather than just "received"? (Today the caller is answered before any work is done and worker creation is not awaited, so failures are invisible to the recruiting side.)
  - If this is not answered: Return the real outcome (created, rejected for missing supervisor, or failed).

REVIEW-NEEDED: When a hire's supervisor is not yet in HCM, should the hire be held and retried automatically, or wait for HRIS to resend it? (Today the hire is dropped after an email to HRIS.)
  - If this is not answered: Email HRIS and wait for a resend, as today.

REVIEW-NEEDED: Can the Interviewed API key that appears in the integration configuration be rotated? (The key is visible in the exported configuration of a switched-off call and should be considered exposed.)
  - If this is not answered: TaskUs rotates it before go-live.

