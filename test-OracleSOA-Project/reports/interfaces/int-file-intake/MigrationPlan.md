# Migration Plan — ProcessINTReadFileABCSImpl (file poll)

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.380Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessINTReadFileABCSImpl (file poll) |
| Identifier | int-file-intake |
| Trigger | Scheduled |
| Starts with | Disk |
| Complexity | complex (66) |
| Components to build | 6 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/BPEL/ProcessINTReadFileABCSImpl.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/fault-policies.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/ProcessINTReadFileABCSImpl_prod_cfgplan.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessINTReadFileABCSImpl/SOA/BPEL/ProcessINTReadFileABCSImpl.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/WebServiceHandler/SOA/BPEL/WebServiceHandlerBPEL.bpel |
| Depends on | hcm-employee-provisioning, dat-profile-generation-rest, util-send-notification, hcm-dat-generation, newhire-emp-sch |

## 2. End-to-End Process Architecture

BPEL composite started by a pick on either the File adapter (ReadINTFile) or the REST service CreateEmployee; sequential forEach with validation, an HCM REST lookup, a call to ProcessEmployeeHCMProvABCSImpl, a 3-minute wait and a REST call to ProcessDatProfileGeneration; fault policy File_Rejection_Handler -> WebServiceHandler for unreadable files. A ProcessHCMExtractSyncProv reference is wired but not invoked. The Employee copy is the older 113-column layout keyed on SSS.

Boomi scheduled process "INT New Hire File Intake": conn-int-file-share + op-int-file-get (Disk or SFTP, get and delete, archive), prof-int-newhire-csv, split per row, map-int-csv-to-ebm, Process Call proc-sub-int-validate-create (shared with jobvite-create-employee-rest) which validates, checks HCM via op-hcm-emps-get, calls hcm-employee-provisioning and then dat-profile-generation-rest when the worker exists; profile errors reported via util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| File adapter ReadINTFile + archive | conn-int-file-share + op-int-file-get | Archive by moving the file. |
| NXSD ReadFileService.xsd | prof-int-newhire-csv |  |
| Fault policy -> WebServiceHandler | Try/Catch around parsing + util-send-notification |  |
| forEach validation and calls | proc-sub-int-validate-create | Fix per-row SSS check. |
| xform_create_emp_Int_to_hcm | map-int-csv-to-ebm |  |
| wait PT3M + Rest_DatProfile | Process Call dat-profile-generation-rest after confirmed creation | Pass the hire data. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Poll folder | Read *.csv from the Interviewed folder (recursive), delete and archive. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Adapters/ReadINTFile_file.jca |
| 2 | Parse file | NXSD ReadFileService (header line, comma-separated). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Schemas/ReadFileService.xsd |
| 3 | Any employees? | count(Employee) > 0, else EmailDL notification "Input Request Empty". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/BPEL/ProcessINTReadFileABCSImpl.bpel |
| 4 | Validate action | Hiring with no EmployeeID or Rehiring with EmployeeID; else "ActionType ... Empty" or "Action type not Valide". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/BPEL/ProcessINTReadFileABCSImpl.bpel |
| 5 | Validate candidate and job | ID (candidate) and JobCode must be present; else notification. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/BPEL/ProcessINTReadFileABCSImpl.bpel |
| 6 | PH SSS checks | CountryCode PH: SSSNo required ("The SSSNo is empty"); HCM emps?q=NationalId=SSSNo must be empty ("Employee Exists in HCM with given SSSNo"). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/WADLs/GetEmployeeData.wadl |
| 7 | Provision | Map to EmployeeEBM and call ProcessEmployeeHCMProvABCSImpl. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl |
| 8 | Wait and follow up | Wait 3 minutes; POST DatProfileGeneration JobVite_Rest. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/WADLs/Rest_DatProfile.wadl |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| INT New Hire CSV Prof | flatfile | Employee (one per line after header) | record | yes |
| INT New Hire CSV Prof | flatfile | ID, ActionType, ActionReason, EmployeeID, Status, Source | string | no |
| INT New Hire CSV Prof | flatfile | FirstName, MiddleName, LastName, PrefixName, SuffixName, PreferredName, Email, EmailAddressType, Gender, CivilStatus, Dateofbirth, Birthplace, Nationality, Religion | string | no |
| INT New Hire CSV Prof | flatfile | SSSNo, TINno, Philhealth, HDMFPagibig, NIDIssuingCountry | string | no |
| INT New Hire CSV Prof | flatfile | Address, AddressLine1, AddressLine2, AddressType, City, Region, PostalCode, Country, CountryCode, AreaCode, PhoneType, PhoneNumber, Phonecountrycode | string | no |
| INT New Hire CSV Prof | flatfile | Visa*/Passport*, CountryofIssueVISA | string | no |
| INT New Hire CSV Prof | flatfile | JobCode, JobName, OrganizationDepartment, BusinessUnit, LegalEntity, LineofBusiness, Location, Campaign, SalaryGrade, Schedule, WorkScheduleStartTime/EndTime, ExpectedStartDate, EffectiveDate, AssignmentCategory, FulltimeParttime, NumOfWorkHours, ContractType, ContractEndDate, PersonType | string | no |
| INT New Hire CSV Prof | flatfile | Salary, Currency, TaxableAllowance, NonTaxableAllowance, SupervisorIdentifier | string | no |
| INT New Hire CSV Prof | flatfile | EducationDetails[], PreviousEmploymentDetails[] | nested | no |
| INT New Hire CSV Prof | flatfile | SeparationDate, SeparationActionType, SeparationReason, FirstAppliedTime | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | Employee/CandidateID and Person/NationalID_Type/NationalIDNumber | ID | transformed |  | substring-before(ID, "-") | low | Candidate ID prefix becomes the HDL key in this variant. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT new-hire record to Employee EBM |
| MR-002 | Employee/EmployeeID, Campaign | EmployeeID, Campaign | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT new-hire record to Employee EBM |
| MR-003 | Person names, prefix/suffix, preferred name, email | FirstName, MiddleName, LastName, PrefixName, SuffixName, PreferredName, Email, EmailAddressType | transformed |  | normalize-space on Prefix | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT new-hire record to Employee EBM |
| MR-004 | NationalID_Type SSS/TIN/Philhealth/HDMF, IssuingCountry | SSSNo, TINno, Philhealth, HDMFPagibig, NIDIssuingCountry | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT new-hire record to Employee EBM |
| MR-005 | Visa / passport | Visa*, Passport*, CountryofIssueVISA | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT new-hire record to Employee EBM |
| MR-006 | CountryBirth, MaritalStatus, Nationality, Religion, Gender, DateOfBirth | Birthplace, CivilStatus, Nationality, Religion, Gender, Dateofbirth | transformed |  | lower-case + trim of CivilStatus | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT new-hire record to Employee EBM |
| MR-007 | ContactAddress / Phone | AddressType, AddressLine1/2, PostalCode, City, Region, Country, CountryCode, AreaCode, PhoneType, PhoneNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT new-hire record to Employee EBM |
| MR-008 | EducationalDetails / PreviousEmploymentDetails | EducationDetails[], PreviousEmploymentDetails[] | looped |  |  | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT new-hire record to Employee EBM |
| MR-009 | EmploymentAssignments (department, job, grade, BU, LOB, hours, location, dates, schedule, contract end, reason) | OrganizationDepartment, JobCode, SalaryGrade, BusinessUnit, LineofBusiness, NumOfWorkHours, Location, ExpectedStartDate, Schedule, WorkScheduleStartTime/EndTime, ContractEndDate, ActionReason, AssignmentCategory, ContractType | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT new-hire record to Employee EBM |
| MR-010 | IsFullPartTime | FulltimeParttime | conditional | "full time" / "part time" | FULL_TIME / PART_TIME | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT new-hire record to Employee EBM |
| MR-011 | Compensation / allowances / supervisor | Currency, Salary, TaxableAllowance, NonTaxableAllowance, SupervisorIdentifier | passthrough |  | AllowanceType = TaxableAllowance, Allowance = NonTaxableAllowance | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT new-hire record to Employee EBM |
| MR-012 | Action | ActionType | conditional | lower-case value | hiring -> HIRE; rehiring -> REHIRE; internal hiring -> INTERNALHIRE | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT new-hire record to Employee EBM |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Action validity | Hiring requires an empty EmployeeID; Rehiring requires an EmployeeID. | High | Decision in proc-sub-int-validate-create. |
| BR-002 | Mandatory candidate and job | Candidate ID and JobCode must be present. | High | Decision. |
| BR-003 | PH SSS uniqueness | Philippine hires need an SSS number not already on an HCM person. | High | op-hcm-emps-get by NationalId (per row). |
| BR-004 | Regional notification | US issues to UShrisJobvite, others to hris_Jobvite. | Medium | Cross reference EmailDL. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound File (SOA server) | /u01/data/domains/SOACS-Pr_domain/Interviewed/*.csv | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/ProcessINTReadFileABCSImpl_prod_cfgplan.xml |
| outbound REST over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCoreApi/resources/latest/emps?q=NationalId=<SSSNo> | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/WADLs/GetEmployeeData.wadl |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/processemployeehcmprovabcsimpl_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/composite.xml |
| outbound REST over HTTPS (composite) | https://129.144.156.28:443/soa-infra/resources/default/ProcessDatProfileGeneration!2.0/JobVite_Rest | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/ProcessINTReadFileABCSImpl_prodJobVite_cfgplan.xml |
| File adapter ReadINTFile (inbound) | eis/FileAdapter; LogicalDirectory LogicalPath ([redacted]); LogicalArchiveDirectory ([redacted]); IncludeFiles .*\.csv; Recursive true; PollingFrequency 60; MinimumAge 0; DeleteFile true; UseHeaders false. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Adapters/ReadINTFile_file.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-employee-provisioning | Sub: HCM Employee Validate and Provision | synchronous |
| dat-profile-generation-rest | DAT profile generation REST | synchronous |
| newhire-emp-sch | HCM REST emps GET (shared) | synchronous |
| hcm-dat-generation | Employee EBM profile | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl | INT record to EmployeeEBM, incl. action-type mapping and date helper templates. | map-int-csv-to-ebm. | Medium |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Per-hire scope catchAll with notification; fault policy for rejected files. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| WebServiceHandler composite | other | yes |
| EmailDL.dvm | dvm | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud REST | HTTPS REST | https://eetm.hcm.us6.oraclecloud.com:443/hcmCoreApi/resources/latest/emps | Oracle (TaskUs tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Input folder | [redacted] | [redacted].xml |
| PROD | Archive folder | [redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

Every minute this integration looks in a folder on the integration server for new-hire files from the recruiting system. Each file holds one line per hire with around 75 columns: personal details, IDs, address, education, previous employment, job, department, location, schedule, salary, allowances and supervisor. For each hire it checks the basic data and, for the Philippines, that the SSS number is present and not already used by someone in HCM. Valid hires are passed to HCM provisioning. Problems are emailed to the HRIS Jobvite list for the US or the rest of the world.

After each hire it waits three minutes, presumably to let HCM finish creating the worker, and then calls the DAT profile generation service. On the file path it passes the REST request, which is empty, instead of the hire's data, so that follow-up step likely receives nothing. The SSS check also looks at the first hire in the file rather than the current one. Because of the per-hire wait, a file of 20 hires takes at least an hour. Files that cannot be read are reported to the "Rejection" email list by a separate fault handler.

In Boomi this becomes a scheduled process with a Disk or SFTP connector, a flat-file profile and a validation sub-process shared with the Jobvite REST listener. The fixed wait should be replaced by checking the worker-creation result.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | INT New Hire CSV Prof | prof-int-newhire-csv | profile.flatfile | — |
| 2 | INT File Drop Conn | conn-int-file-share | connector-settings | — |
| 3 | INT New Hire CSV GET n DEL Op | op-int-file-get | connector-action | conn-int-file-share |
| 4 | Map \| INT New Hire to Employee EBM | map-int-csv-to-ebm | transform.map | prof-int-newhire-csv, prof-employee-ebm-xml |
| 5 | Sub: INT Employee Validate and Create | proc-sub-int-validate-create | process | op-hcm-emps-get, map-int-csv-to-ebm, proc-sub-hcm-employee-provisioning, proc-sub-send-notification |
| 6 | INT New Hire File Intake | proc-int-file-intake | process | op-int-file-get, prof-int-newhire-csv, proc-sub-int-validate-create, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/BPEL/ProcessINTReadFileABCSImpl.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/fault-policies.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/fault-bindings.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/ProcessINTReadFileABCSImpl_prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/ProcessINTReadFileABCSImpl_prodJobVite_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessINTReadFileABCSImpl/SOA/BPEL/ProcessINTReadFileABCSImpl.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/WebServiceHandler/SOA/BPEL/WebServiceHandlerBPEL.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Adapters/ReadINTFile_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Schemas/ReadFileService.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/Transformations/xform_create_emp_Int_to_hcm.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/WADLs/Rest_DatProfile.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/SOA/WADLs/GetEmployeeData.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/BPEL/ProcessEmployeeHCMProvABCSImpl.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/EnterpriseObjectLibrary/Core/EBO/Employee/V1/EmployeeEBM.xsd

## Open questions

### Still outstanding

REVIEW-NEEDED: Which new-hire file layout is live: the older 113-column layout keyed on the SSS number, or the Jobvite layout keyed on the candidate ID? (Two input layouts exist. Confirm which is live. The HDL SourceSystemId scheme must not change for existing persons.)
  - If this is not answered: The Employee_JobVite layout, keyed on the candidate-ID prefix.

REVIEW-NEEDED: Which system produces the new-hire files, and where should they be dropped for Boomi (a folder on the Boomi server or an SFTP location)? (The producer of the CSV files in /u01/data/domains/SOACS-Pr_domain/Interviewed and the target drop location for Boomi are not defined in the export.)
  - If this is not answered: An SFTP folder managed by TaskUs IT.

REVIEW-NEEDED: Is the three-minute pause after each hire needed, or can the follow-up step run once HCM confirms the worker is created? (The pause makes large files slow (20 hires take over an hour).)
  - If this is not answered: Replace the pause with a check that the worker exists.

REVIEW-NEEDED: Who should be told when a whole file cannot be read? (Unreadable files are sent to the "Rejection" distribution list today.)
  - If this is not answered: Keep the "Rejection" list.

