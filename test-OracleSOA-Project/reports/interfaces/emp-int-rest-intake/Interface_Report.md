# ProcessEmployeeINTReqABCSImpl

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

REST service (POST /emp, JSON) through which an external recruiting system submits a new hire or rehire. It converts the flat INT employee JSON into the Employee EBM and calls hcm-employee-provisioning, returning employeeID, candidateID, attribute1/2 and status. A request that already carries an EmployeeId (and is not a REHIRE) is answered "Employee Already Exist" without further processing.

| Field | Value |
| --- | --- |
| Interface | ProcessEmployeeINTReqABCSImpl |
| Type | Listener |
| Complexity | simple |
| Components to build | 4 |
| Mapping rules | 11 |
| Business rules | 1 |

## Interface Summary

This is a web service that lets an outside recruiting system send a new hire to TaskUs. The system posts the hire's details (personal data, address, national IDs, visa, education, previous employers, job, department, location, schedule, salary, allowance, supervisor) as JSON. The service converts them to the standard TaskUs employee format and passes them to HCM provisioning, then answers with the result. If the request already has an employee number and is not a rehire, it answers "Employee Already Exist".  The request format allows several employees per call, but only the first is processed. The service is protected with an HTTP username/password policy. On an unexpected error it emails the notification list in a format the notification service cannot read and gives the caller no answer. The export does not say which system calls it, so TaskUs needs to confirm whether it is still used.  In Boomi this becomes a Web Services Server listener with JSON request and response profiles and one map, calling the shared provisioning sub-process.

**Complexity:** simple (score 20)

## Functional Analysis

1. **Receive hire JSON** — POST /emp with EmployeeRequest.
2. **New or rehire?** — EmployeeId empty, or EmployeeId given and Action REHIRE -\> continue; else "Employee Already Exist".
3. **Map to Employee EBM** — xform\_create\_emp\_Int\_to\_hcm.
4. **Provision in HCM** — ProcessEmployeeHCMProvABCSImpl (request/reply).
5. **Reply** — employeeID, candidateID, attribute1, attribute2, status from TransmissionAck.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | REST / JSON over HTTP(S) | /emp (service CreateEmployee) | HTTP Basic (OWSM wss\_http\_token\_service\_policy) |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/processemployeehcmprovabcsimpl\_client\_ep.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | INT employee JSON to Employee EBM | Employee/EmployeeID, CandidateID, Campaign | EmployeeId, CandidateIDorApplicantID, Campaign | passthrough | — | — | low | — |
| MR-002 | INT employee JSON to Employee EBM | Person (names, prefix/suffix, email, legal entity, person type) | FirstName, MiddleName, LastName, Prefix, Suffix, EmailAddress, EmailAddressType, LegalEntityName, PersonTypeOrContractType | passthrough | — | — | low | — |
| MR-003 | INT employee JSON to Employee EBM | Person/NationalID\_Type/NationalIDNumber | SSSNumber | passthrough | — | — | low | Becomes the HDL key. |
| MR-004 | INT employee JSON to Employee EBM | Person/NationalID\_Type SSS/TIN/Philhealth/HDMF, IssuingCountry | SSSNumber, TINNumber, PhilhealthNumber, HDMFNumber, IssuingCountry | passthrough | — | — | low | — |
| MR-005 | INT employee JSON to Employee EBM | Person/Visa/\* | Visa\*/Passport\* fields | passthrough | — | — | low | — |
| MR-006 | INT employee JSON to Employee EBM | CountryBirth, MaritalStatus, Nationality, Religion, Gender, DateOfBirth | same names (CountryOfBirth) | passthrough | — | — | low | — |
| MR-007 | INT employee JSON to Employee EBM | ContactDetails/ContactAddress, Phone | AddressType, Address1/2, ZipCode, City, Region, Country, CountryCode, AreaCode, PhoneType, PrimaryPhoneNumber | passthrough | — | — | low | — |
| MR-008 | INT employee JSON to Employee EBM | EducationalDetails / PreviousEmploymentDetails (1-3) | College\*/Masteral\*, PreviousEmployer\*/Position\*/Salary\* | passthrough | — | — | low | — |
| MR-009 | INT employee JSON to Employee EBM | EmploymentAssignments (department, job, grade, BU, LOB, probation, location, dates, schedule, action reason) | DepartmentName, JobCode, GradeCode, BusinessUnitName, LOB, ProbationEndDate, LocationCode, EffectiveStartDate, WorkSchedule, Starttime, Endtime, StartDateOrHireDate, ActionReason | passthrough | — | — | low | — |
| MR-010 | INT employee JSON to Employee EBM | EmployeeCompensation / EmployeeAllowanceType / SupervisorEmpID / Action | Currency, BasicSalary, AllowanceType, Allowance, SupervisorEID, Action | passthrough | — | — | low | — |
| MR-011 | TransmissionAck to response JSON | employeeID, candidateID, attribute1, attribute2, status | TransmissionAck same fields | passthrough | — | — | low | "Employee Already Exist" when skipped. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Duplicate guard | A request with an EmployeeId is only processed when Action = REHIRE. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **POST /emp received** — JSON EmployeeRequest
2. **New hire or rehire?** — EmployeeId empty or Action REHIRE
3. **Map to Employee EBM**
4. **Reply already exists**
5. **Provision in HCM** — hcm-employee-provisioning
6. **Reply result**
7. **Notify error, no reply**

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| INT Employee Create Req JSON Prof | json | prof-int-emp-req-json |
| INT Employee Create Resp JSON Prof | json | prof-int-emp-resp-json |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| REST binding /emp + OWSM policy | Web Services Server listener with JSON profiles and basic auth |
| xform\_create\_emp\_Int\_to\_hcm | map-int-json-to-ebm |
| ProcessEmployeeHCMProvABCSImpl | Process Call hcm-employee-provisioning |
| catchAll | Try/Catch + error response + util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| prof-int-emp-req-json | Request profile |
| prof-int-emp-resp-json | Response profile |
| map-int-json-to-ebm | Request map |
| proc-emp-int-rest-intake | Listener process |

## Client questions

- **Q-001** Which system sends new hires to the TaskUs "/emp" web service today, and is it still in use?
  **If this is not answered:** Keep the endpoint with the same request/response format.
- **Q-002** Should one request be able to carry several hires?
  **If this is not answered:** Process every hire in the request and return one result per hire.
