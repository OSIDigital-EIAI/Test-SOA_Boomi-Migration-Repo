# Migration Plan — ProcessINTReadFileABCSImpl (REST CreateEmployee)

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.400Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessINTReadFileABCSImpl (REST CreateEmployee) |
| Identifier | jobvite-create-employee-rest |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | medium (36) |
| Components to build | 2 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/WADLs/CreateEmployee.wadl, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/BPEL/ProcessINTReadFileABCSImpl.bpel |
| Depends on | int-file-intake, dat-profile-generation-rest |

## 2. End-to-End Process Architecture

Second onMessage branch of the pick in ProcessINTReadFileABCSImpl; XSLT to the INT record, reply, then the same forEach as the file path.

Boomi listener "Jobvite Create Employee REST Listener" (Web Services Server, JSON, basic auth): map-jobvite-json-to-int, Process Call proc-sub-int-validate-create (int-file-intake) per hire, Return Documents with the outcome; jobvite-csv-intake should call the sub-process directly.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| REST binding /create + OWSM | Web Services Server listener with JSON profile and basic auth |  |
| xform_rest_to_inputvar | map-jobvite-json-to-int |  |
| Shared forEach | Process Call proc-sub-int-validate-create (int-file-intake) |  |
| wait + Rest_DatProfile | Process Call dat-profile-generation-rest after confirmed creation |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive Jobvite JSON | POST /create (pick onMessage CreateEmployee). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/WADLs/CreateEmployee.wadl |
| 2 | Map to INT record | xform_rest_to_inputvar. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_rest_to_inputvar.xsl |
| 3 | Acknowledge | Reply candidateID and "Employee details received successfully". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/BPEL/ProcessINTReadFileABCSImpl.bpel |
| 4 | Validate and create each hire | Shared forEach as in int-file-intake. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/BPEL/ProcessINTReadFileABCSImpl.bpel |
| 5 | Follow-up | Wait 3 minutes, POST the original request to dat-profile-generation-rest. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/WADLs/Rest_DatProfile.wadl |

## 4. Payload Profiles

> **Not available.** No profile components owned here: the request uses the INT record layout of int-file-intake (ReadFileService.xsd in JSON form) and the response is the same small ack as emp-int-rest-intake.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | Employee/* (about 75 scalar fields) | Employees/Employee/* same names | looped | each Employee | copy | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_rest_to_inputvar.xsl | Jobvite Employees JSON to INT record |
| MR-002 | Employee/PhoneNumber | PhoneNumber | conditional | contains "+" | translate(PhoneNumber, "+", "") | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_rest_to_inputvar.xsl | Jobvite Employees JSON to INT record |
| MR-003 | EducationDetails[] / PreviousEmploymentDetails[] | same nested groups | looped |  | copy | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_rest_to_inputvar.xsl | Jobvite Employees JSON to INT record |
| MR-004 | Response candidateID / status | Employee[1]/ID | hardcoded |  | status "Employee details received successfully" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/BPEL/ProcessINTReadFileABCSImpl.bpel | Jobvite Employees JSON to INT record |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Early acknowledgement | The caller is told the details were received before any validation. | Medium | Return the validation outcome instead (if the caller can use it). |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound REST / JSON over HTTP (caller uses http://129.144.156.139:9073/soa-infra/resources/default/ProcessINTReadFileABCSImpl!2.0/CreateEmployee/create) | /create (service CreateEmployee) | HTTP Basic (OWSM wss_http_token_service_policy) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/wsm-assembly.xml |
| outbound REST over HTTPS (composite) | https://129.144.156.28:443/soa-infra/resources/default/ProcessDatProfileGeneration!2.0/JobVite_Rest | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/ProcessINTReadFileABCSImpl_prodJobVite_cfgplan.xml |
| REST service binding CreateEmployee | WADL resource /create, POST, JSON request Employees (ReadFileService.xsd) and response EmployeeResponse. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/WADLs/CreateEmployee.wadl |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| int-file-intake | Sub: INT Employee Validate and Create; INT record profile | synchronous |
| dat-profile-generation-rest | DAT profile generation REST | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_rest_to_inputvar.xsl | Jobvite JSON to INT record (copy, phone clean-up). | map-jobvite-json-to-int. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Early reply; per-hire handling as in int-file-intake. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

> **Not available.** No shared artefacts were recorded for this interface.

## 12. External Services

> **Not available.** No external services were recorded for this interface.

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | DatProfile endpoint | https://129.144.156.28:[redacted]!2.0/JobVite_Rest | [redacted].xml |

## 14. Boomi Migration Readiness Summary

This is the web-service door into the hiring process used by the Jobvite file intake. Each call carries one or more hires in Jobvite JSON. The service converts them to the standard TaskUs new-hire record, answers straight away that the details were received, and then processes each hire in the same way as the file intake: validation, the Philippine SSS duplicate check and creation in HCM. After each hire it waits three minutes and passes the original request to the DAT profile generation service, which handles salary, talent profile and allowances.

Because the reply is sent before any checks, the Jobvite side always sees success; problems are only emailed to the HRIS lists. In Boomi this becomes a Web Services Server listener that maps the JSON and calls the validation sub-process built for int-file-intake. Since the only known caller is another TaskUs integration, it should call that sub-process directly instead of making an HTTP call to itself.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Map \| Jobvite Employees JSON to INT Record | map-jobvite-json-to-int | transform.map | prof-jobvite-employees-json, prof-int-newhire-csv |
| 2 | Jobvite Create Employee REST Listener | proc-jobvite-create-employee-rest | process | prof-jobvite-employees-json, map-jobvite-json-to-int, proc-sub-int-validate-create, proc-sub-dat-profile-core |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/BPEL/ProcessINTReadFileABCSImpl.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/WADLs/CreateEmployee.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_rest_to_inputvar.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Schemas/ReadFileService.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Schemas/create_employee_response.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/wsm-assembly.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/ProcessINTReadFileABCSImpl_prodJobVite_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/WADLs/Rest_DatProfile.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessCallJobviteRestService/SOA/composite.xml

## Open questions

### Still outstanding

REVIEW-NEEDED: Does any system other than the Jobvite file intake call the TaskUs "/create" hire service? (If not, the endpoint can be dropped and the Jobvite intake can call the hire validation directly in Boomi.)
  - If this is not answered: Only the Jobvite file intake uses it; no external endpoint is published.

REVIEW-NEEDED: Should the Jobvite side be told which hires were rejected, rather than always receiving "received successfully"? (Today the reply is sent before any validation.)
  - If this is not answered: Return per-hire outcomes.

