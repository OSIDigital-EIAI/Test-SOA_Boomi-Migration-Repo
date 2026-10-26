# Migration Plan — ProcessUpdateEmployeeEmail

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.015Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessUpdateEmployeeEmail |
| Identifier | hcm-update-employee-email |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | simple (12) |
| Components to build | 4 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/BPEL/ProcessUpdateEmployeeEmail.bpel |
| Depends on | util-send-notification |

## 2. End-to-End Process Architecture

Synchronous BPEL composite with one SOAP reference to HCM WorkerService (MDS WSDL) and XXSendNotification.

Boomi sub-process "Sub: HCM Update Work Email" called by newhire-emp-sch: map-email-to-mergeperson, op-hcm-merge-person on conn-hcm-worker-soap, return a success/failure flag; Try/Catch with util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / reply | Sub-process start + Return Documents | Return a real status. |
| Assign_UpdateEmail | map-email-to-mergeperson |  |
| UpdateEmail (WorkerService mergePerson) | conn-hcm-worker-soap + op-hcm-merge-person | Verify V2 availability. |
| catchAll | Try/Catch + util-send-notification |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive request | Synchronous process with PersonNumber, DateFrom, EmailAddress. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/Schemas/ProcessUpdateEmployeeEmail.xsd |
| 2 | Build mergePerson | PersonNumber, PersonEmail DateFrom/PersonNumber/EmailType W1/PrimaryFlag true/EmailAddress. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/BPEL/ProcessUpdateEmployeeEmail.bpel |
| 3 | Update HCM | WorkerService.mergePerson with HTTP Authorization header. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/WorkerService_wsdl.wsdl |
| 4 | Reply | Empty processResponse (result not set). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/BPEL/ProcessUpdateEmployeeEmail.bpel |

## 4. Payload Profiles

> **Not available.** This interface owns no profile components; the mergePerson request/response shapes come from the WorkerService WSDL imported into the connector operation.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | attributeList/PersonNumber | PersonNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/BPEL/ProcessUpdateEmployeeEmail.bpel | Email request to mergePerson |
| MR-002 | attributeList/PersonEmail/DateFrom | DateFrom | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/BPEL/ProcessUpdateEmployeeEmail.bpel | Email request to mergePerson |
| MR-003 | attributeList/PersonEmail/PersonNumber | PersonNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/BPEL/ProcessUpdateEmployeeEmail.bpel | Email request to mergePerson |
| MR-004 | attributeList/PersonEmail/EmailType | (none) | hardcoded |  | "W1" | low | Work email. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/BPEL/ProcessUpdateEmployeeEmail.bpel | Email request to mergePerson |
| MR-005 | attributeList/PersonEmail/PrimaryFlag | (none) | hardcoded |  | "true" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/BPEL/ProcessUpdateEmployeeEmail.bpel | Email request to mergePerson |
| MR-006 | attributeList/PersonEmail/EmailAddress | EmailAddress | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/BPEL/ProcessUpdateEmployeeEmail.bpel | Email request to mergePerson |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Primary work email | The supplied address is stored as email type W1 with PrimaryFlag true from DateFrom. | Medium | map-email-to-mergeperson + op-hcm-merge-person. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 synchronous | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateEmployeeEmail/processupdateemployeeemail_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/ProcessUpdateEmployeeEmail.wsdl |
| outbound SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmEmploymentCoreWorkerV2/WorkerService | HTTP Authorization header (Credentials.dvm); WSDL advertises WS-Security | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/WorkerService_wsdl.wsdl |
| outbound SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/composite.xml |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

> **Not available.** No custom code was found in this interface.

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll: notify EmailDL notification, reply with empty response. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/HCM/WorkerService_wsdl.wsdl | mds | yes |
| Credentials.dvm (Authorization) | dvm | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud WorkerService | SOAP / HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmEmploymentCoreWorkerV2/WorkerService | Oracle (TaskUs tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | WorkerService endpoint | https://eetm.hcm.us6.oraclecloud.com:[redacted] | [redacted].wsdl |

## 14. Boomi Migration Readiness Summary

This small service writes a new employee's company email address into Oracle HCM. The new-hire job calls it with the employee number, the date the address takes effect and the address itself, and it updates the employee's primary work email in HCM.

It answers the caller with an empty reply whether it worked or not, so the new-hire job cannot tell a failure from a success; failures are only emailed to the notification list. It also uses an older version of Oracle's worker web service, which should be checked against the current HCM release before the rebuild.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | HCM WorkerService SOAP Conn | conn-hcm-worker-soap | connector-settings | — |
| 2 | HCM Worker Email MERGE Op | op-hcm-merge-person | connector-action | conn-hcm-worker-soap |
| 3 | Map \| Email Request to HCM MergePerson | map-email-to-mergeperson | transform.map | — |
| 4 | Sub: HCM Update Work Email | proc-sub-hcm-update-email | process | op-hcm-merge-person, map-email-to-mergeperson, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/BPEL/ProcessUpdateEmployeeEmail.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateEmployeeEmail/SOA/Schemas/ProcessUpdateEmployeeEmail.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/WorkerService_wsdl.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/ProcessUpdateEmployeeEmail.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessDatProfileGeneration/SOA/composite.xml

## Open questions

### Still outstanding

REVIEW-NEEDED: Is the Oracle HCM SOAP worker service (version 2) still enabled in your HCM release, or should the new process use the HCM REST API to update work emails? (The legacy service may be deprecated; the connection cannot be built until this is confirmed.)
  - If this is not answered: Keep the SOAP WorkerService mergePerson as today.

REVIEW-NEEDED: If the work email cannot be saved in HCM, should the new-hire process stop for that person or continue and report it? (Today the failure is invisible to the new-hire job.)
  - If this is not answered: Continue the new hire and report the failure to the notification list.

