# Migration Plan — ProcessEmployeeINTReqABCSImpl

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.355Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessEmployeeINTReqABCSImpl |
| Identifier | emp-int-rest-intake |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | simple (16) |
| Components to build | 4 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/BPEL/ProcessEmployeeINTReqABCSImpl.bpel |
| Depends on | hcm-employee-provisioning, util-send-notification, hcm-dat-generation |

## 2. End-to-End Process Architecture

BPEL composite exposed through a REST binding with an OWSM HTTP-token policy; one XSLT map; synchronous call to ProcessEmployeeHCMProvABCSImpl via MDS WSDL; XXSendNotification on error.

Boomi listener "INT Employee Create REST Listener" (Web Services Server, JSON, basic auth): Decision on EmployeeId/Action, map-int-json-to-ebm, Process Call hcm-employee-provisioning, map ack to prof-int-emp-resp-json, Return Documents; Try/Catch returning an error response and notifying.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| REST binding /emp + OWSM policy | Web Services Server listener with JSON profiles and basic auth |  |
| xform_create_emp_Int_to_hcm | map-int-json-to-ebm | Loop over all employees. |
| ProcessEmployeeHCMProvABCSImpl | Process Call hcm-employee-provisioning |  |
| catchAll | Try/Catch + error response + util-send-notification |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive hire JSON | POST /emp with EmployeeRequest. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/WADLs/CreateEmployee.wadl |
| 2 | New or rehire? | EmployeeId empty, or EmployeeId given and Action REHIRE -> continue; else "Employee Already Exist". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/BPEL/ProcessEmployeeINTReqABCSImpl.bpel |
| 3 | Map to Employee EBM | xform_create_emp_Int_to_hcm. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl |
| 4 | Provision in HCM | ProcessEmployeeHCMProvABCSImpl (request/reply). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/composite.xml |
| 5 | Reply | employeeID, candidateID, attribute1, attribute2, status from TransmissionAck. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/BPEL/ProcessEmployeeINTReqABCSImpl.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| INT Employee Create Req JSON Prof | json | Employee[] | array | yes |
| INT Employee Create Req JSON Prof | json | Employee.EmployeeId, CandidateIDorApplicantID, Action | string | no |
| INT Employee Create Req JSON Prof | json | Employee.FirstName, MiddleName, LastName, Prefix, Suffix, EmailAddress, EmailAddressType | string | no |
| INT Employee Create Req JSON Prof | json | Employee.SSSNumber, TINNumber, PhilhealthNumber, HDMFNumber, IssuingCountry | string | no |
| INT Employee Create Req JSON Prof | json | Employee.Address1, Address2, City, Region, ZipCode, Country, CountryCode, AreaCode, PhoneType, PrimaryPhoneNumber | string | no |
| INT Employee Create Req JSON Prof | json | Employee.Visa*/Passport*, College*/Masteral*, PreviousEmployer1-3 fields | string | no |
| INT Employee Create Req JSON Prof | json | Employee.DepartmentName, JobCode, GradeCode, BusinessUnitName, LOB, LocationCode, EffectiveStartDate, StartDateOrHireDate, WorkSchedule, Starttime, Endtime, ProbationEndDate, ActionReason | string | no |
| INT Employee Create Req JSON Prof | json | Employee.Currency, BasicSalary, AllowanceType, Allowance, SupervisorEID, Campaign, LegalEntityName, PersonTypeOrContractType | string | no |
| INT Employee Create Resp JSON Prof | json | employeeID | string | no |
| INT Employee Create Resp JSON Prof | json | candidateID | string | no |
| INT Employee Create Resp JSON Prof | json | attribute1 | string | no |
| INT Employee Create Resp JSON Prof | json | attribute2 | string | no |
| INT Employee Create Resp JSON Prof | json | status | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | Employee/EmployeeID, CandidateID, Campaign | EmployeeId, CandidateIDorApplicantID, Campaign | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT employee JSON to Employee EBM |
| MR-002 | Person (names, prefix/suffix, email, legal entity, person type) | FirstName, MiddleName, LastName, Prefix, Suffix, EmailAddress, EmailAddressType, LegalEntityName, PersonTypeOrContractType | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT employee JSON to Employee EBM |
| MR-003 | Person/NationalID_Type/NationalIDNumber | SSSNumber | passthrough |  |  | low | Becomes the HDL key. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT employee JSON to Employee EBM |
| MR-004 | Person/NationalID_Type SSS/TIN/Philhealth/HDMF, IssuingCountry | SSSNumber, TINNumber, PhilhealthNumber, HDMFNumber, IssuingCountry | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT employee JSON to Employee EBM |
| MR-005 | Person/Visa/* | Visa*/Passport* fields | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT employee JSON to Employee EBM |
| MR-006 | CountryBirth, MaritalStatus, Nationality, Religion, Gender, DateOfBirth | same names (CountryOfBirth) | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT employee JSON to Employee EBM |
| MR-007 | ContactDetails/ContactAddress, Phone | AddressType, Address1/2, ZipCode, City, Region, Country, CountryCode, AreaCode, PhoneType, PrimaryPhoneNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT employee JSON to Employee EBM |
| MR-008 | EducationalDetails / PreviousEmploymentDetails (1-3) | College*/Masteral*, PreviousEmployer*/Position*/Salary* | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT employee JSON to Employee EBM |
| MR-009 | EmploymentAssignments (department, job, grade, BU, LOB, probation, location, dates, schedule, action reason) | DepartmentName, JobCode, GradeCode, BusinessUnitName, LOB, ProbationEndDate, LocationCode, EffectiveStartDate, WorkSchedule, Starttime, Endtime, StartDateOrHireDate, ActionReason | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT employee JSON to Employee EBM |
| MR-010 | EmployeeCompensation / EmployeeAllowanceType / SupervisorEmpID / Action | Currency, BasicSalary, AllowanceType, Allowance, SupervisorEID, Action | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT employee JSON to Employee EBM |
| MR-011 | employeeID, candidateID, attribute1, attribute2, status | TransmissionAck same fields | passthrough |  |  | low | "Employee Already Exist" when skipped. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/BPEL/ProcessEmployeeINTReqABCSImpl.bpel | TransmissionAck to response JSON |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Duplicate guard | A request with an EmployeeId is only processed when Action = REHIRE. | Medium | Decision before the map. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound REST / JSON over HTTP(S) | /emp (service CreateEmployee) | HTTP Basic (OWSM wss_http_token_service_policy) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/wsm-assembly.xml |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/processemployeehcmprovabcsimpl_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/composite.xml |
| REST service binding CreateEmployee | WADL resource /emp, POST, JSON request EmployeeRequest and response EmployeeResponse (NXSD JSON schemas). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/WADLs/CreateEmployee.wadl |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-employee-provisioning | Sub: HCM Employee Validate and Provision | synchronous |
| hcm-dat-generation | Employee EBM / ack profiles | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT JSON fields to EmployeeEBM. | map-int-json-to-ebm. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll notify only; no reply. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| EmployeeEBM.xsd (PROD_MDS snapshot) | mds | yes |

## 12. External Services

> **Not available.** No external services were recorded for this interface.

## 13. Environment Configuration Matrix

> **Not available.** No environment-specific configuration was recorded.

## 14. Boomi Migration Readiness Summary

This is a web service that lets an outside recruiting system send a new hire to TaskUs. The system posts the hire's details (personal data, address, national IDs, visa, education, previous employers, job, department, location, schedule, salary, allowance, supervisor) as JSON. The service converts them to the standard TaskUs employee format and passes them to HCM provisioning, then answers with the result. If the request already has an employee number and is not a rehire, it answers "Employee Already Exist".

The request format allows several employees per call, but only the first is processed. The service is protected with an HTTP username/password policy. On an unexpected error it emails the notification list in a format the notification service cannot read and gives the caller no answer. The export does not say which system calls it, so TaskUs needs to confirm whether it is still used.

In Boomi this becomes a Web Services Server listener with JSON request and response profiles and one map, calling the shared provisioning sub-process.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | INT Employee Create Req JSON Prof | prof-int-emp-req-json | profile.json | — |
| 2 | INT Employee Create Resp JSON Prof | prof-int-emp-resp-json | profile.json | — |
| 3 | Map \| INT Employee JSON to Employee EBM | map-int-json-to-ebm | transform.map | prof-int-emp-req-json, prof-employee-ebm-xml |
| 4 | INT Employee Create REST Listener | proc-emp-int-rest-intake | process | prof-int-emp-resp-json, map-int-json-to-ebm, proc-sub-hcm-employee-provisioning, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/BPEL/ProcessEmployeeINTReqABCSImpl.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/WADLs/CreateEmployee.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Schemas/create_employee_request.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Schemas/create_employee_response.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/WADLs/create_employee_response_EmployeeResponse.json
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeINTReqABCSImpl/SOA/wsm-assembly.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/BPEL/ProcessEmployeeHCMProvABCSImpl.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmployeeHCMProvABCSImpl/SOA/BPEL/ProcessEmployeeHCMProvABCSImpl.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/EnterpriseObjectLibrary/Core/EBO/Employee/V1/EmployeeEBM.xsd

## Open questions

### Still outstanding

REVIEW-NEEDED: Which system sends new hires to the TaskUs "/emp" web service today, and is it still in use? (The caller is not identified in the export; the endpoint and its authentication must be preserved only if it is used.)
  - If this is not answered: Keep the endpoint with the same request/response format.

REVIEW-NEEDED: Should one request be able to carry several hires? (The format allows several, but only the first is processed today.)
  - If this is not answered: Process every hire in the request and return one result per hire.

