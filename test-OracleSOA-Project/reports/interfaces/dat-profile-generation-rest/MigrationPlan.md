# Migration Plan — ProcessDatProfileGeneration (REST JobVite_Rest)

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.433Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessDatProfileGeneration (REST JobVite_Rest) |
| Identifier | dat-profile-generation-rest |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | complex (80) |
| Components to build | 15 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/BPEL/ProcessDatProfileGeneration.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/ProcessDatProfileGeneration_prod_cfgplan.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessDatProfileGeneration/SOA/BPEL/ProcessDatProfileGeneration.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessDatProfileGeneration/SOA/BPEL/ProcessDatProfileGeneration.bpel |
| Depends on | hcm-get-person-number, hcm-get-hdl-errors, hcm-hdl-loader, hcm-pbl-allowances, gsuite-user-lifecycle, hcm-update-employee-email, util-file-notification, util-send-notification |

## 2. End-to-End Process Architecture

BPEL composite with a pick on REST JobVite_Rest or the inbound File adapter, sequential loop with async calls to GetPersonNumber and getHDLErrors, REST PUT to Jobvite, five File adapters, FileNotification, and fire-and-forget calls to the HDL and PBL loaders; SOA XREF for candidate->content. Variants: Employee (older, no Jobvite update or consolidated file), GSuite_Java_Process (adds Google user creation, work-email update and an attachment file), Employee_JobVite (baseline).

Boomi listener "DAT Profile Generation REST Listener" (Web Services Server, JSON, authenticated) calling "Sub: DAT Profile Generation Core" (shared with dat-profile-archive-poll): per candidate hcm-get-person-number, hcm-get-hdl-errors via the agreed content-ID store, map-int-to-jobvite-status + op-jobvite-put-candidate on conn-jobvite-api, map-int-to-hcmlist + util-file-notification; batch maps map-int-to-salary-talent-dat -> Process Call hcm-hdl-loader, map to EBM -> Process Call hcm-pbl-allowances; map-int-to-consolidated rows written to the agreed staging store for the 23:30 job.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| REST JobVite_Rest | Web Services Server listener (proc-dat-profile-generation-rest) | Add authentication. |
| Loop body and batch tail | proc-sub-dat-profile-core |  |
| JobvitePutCandidate | conn-jobvite-api + op-jobvite-put-candidate | New keys. |
| XRef_UCMID lookup / markForDelete | Agreed persistent store (see hcm-dat-generation) |  |
| File adapters + FileNotification | In-memory CSV + util-file-notification |  |
| Consolidated file (append) | Staging store read by jobvite-consolidated-notification-sch |  |
| Salary/TalentProfile files + HDL call | map-int-to-salary-talent-dat + Process Call hcm-hdl-loader |  |
| PBL call | Process Call hcm-pbl-allowances | Use HCM person number. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive candidates | REST POST (JobVite_Rest) -> InputVariable; reply immediately. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/WADLs/JobVite_Rest.wadl |
| 2 | Look up person number | Per candidate: ProcessGetPersonNumber, wait for callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/BPEL/ProcessDatProfileGeneration.bpel |
| 3 | Success path | Set EmployeeID, clear message, delete XREF row; Jobvite PUT Success. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_hcm_to_jobvite.xsl |
| 4 | Failure path | Look up contentID in XRef_UCMID; get HDL errors; message into ContractEndDate; count failure; Jobvite PUT Error. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_hcm_to_jobvite_error.xsl |
| 5 | Write and email lists | HCMList_<ts>.csv and HCMList_hris_<ts>.csv; FileNotification to list recipients by country. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_writeEmailfile.xsl |
| 6 | Salary and talent profile | If any candidate succeeded: Salary.dat and TalentProfile.dat for candidates with EmployeeID. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_IntReq_to_WriteSalatyDat.xsl |
| 7 | Load and allowances | ProcessHDLProvABCSImpl and ProcessPBLProvABCSImpl (no wait). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_int_to_pbl.xsl |
| 8 | Consolidated file | Append rows to JobviteConsolidate_PH.csv or _US.csv by country of the first candidate. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_PH_Write.xsl |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| Jobvite Employees Create Req JSON Prof | json | Employee[] | array | yes |
| Jobvite Employees Create Req JSON Prof | json | Employee.ID | string | yes |
| Jobvite Employees Create Req JSON Prof | json | Employee.SSSNo, EmployeeID, ExpectedStartDate, Location, Salary, TaxableAllowance, NonTaxableAllowance, Country, CountryCode | string | no |
| Jobvite Candidate Update Req JSON Prof | json | candidate.application.eId | string | yes |
| Jobvite Candidate Update Req JSON Prof | json | candidate.application.customField[].fieldCode | string | yes |
| Jobvite Candidate Update Req JSON Prof | json | candidate.application.customField[].value | string | no |
| HCM List Report CSV Prof | flatfile | EmailList (EmployeeID, Message, and all input fields) | string | no |
| HRIS List Report CSV Prof | flatfile | Email/Date, InstanceID, User, EID, Department, ProjectID, JobTitle, LocationName, Error, Message | string | no |
| HCM HDL Salary DAT Prof | flatfile | Salary METADATA/MERGE: SourceSystemOwner, SourceSystemId, AssignmentNumber, DateFrom, SalaryAmount, SalaryBasisName, MultipleComponents, SalaryApproved, ActionCode, ... | string | yes |
| HCM HDL TalentProfile DAT Prof | flatfile | TalentProfile METADATA/MERGE: ProfileCode, ProfileStatusCode, ProfileUsageCode, PersonNumber, ProfileTypeCode, ... | string | yes |
| HCM HDL TalentProfile DAT Prof | flatfile | ProfileItem METADATA/MERGE: DateFrom, DateTo, ContentType, ItemText2401..2407, ItemText308, ... | string | no |
| Jobvite Consolidated Row CSV Prof | flatfile | Employee/EmployeeId, CandidateId, FirstName, MiddleName, LastName, Status, Message | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | candidate.application.eId | Employee/ID | transformed |  | substring-before(ID, "-") | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_hcm_to_jobvite.xsl | Candidate to Jobvite status update |
| MR-002 | customField hcm_integration_status | (outcome) | conditional | person number found | "Success" / "Error" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_hcm_to_jobvite_error.xsl | Candidate to Jobvite status update |
| MR-003 | customField hcm_integration_details | ContractEndDate (error message) | conditional | Error |  | low | Error text carried in ContractEndDate. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_hcm_to_jobvite_error.xsl | Candidate to Jobvite status update |
| MR-004 | customField hcm_employee_id | EmployeeID | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_hcm_to_jobvite.xsl | Candidate to Jobvite status update |
| MR-005 | HCMList row (all input fields + Message) | Employee/* | looped | each Employee |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_writeEmailfile.xsl | Candidates to HCM and HRIS lists |
| MR-006 | HRIS row Date, InstanceID, User, EID, Department, ProjectID, JobTitle, LocationName, Error | now, FirstAppliedTime, FirstName+LastName, EmployeeID, OrganizationDepartment, Campaign, JobCode, Location, ContractEndDate | looped | each Employee |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_hris_writeEmail.xsl | Candidates to HCM and HRIS lists |
| MR-007 | HRIS Message | EmployeeID | conditional | EmployeeID empty | "Failed" else "Success" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_hris_writeEmail.xsl | Candidates to HCM and HRIS lists |
| MR-008 | Salary: SourceSystemId / AssignmentNumber | ID | calculated | EmployeeID != "" | "Worker_Salary_" + K / "ASSIGNMENT_" + K (K = ID prefix) | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_IntReq_to_WriteSalatyDat.xsl | Candidates to Salary and TalentProfile DAT |
| MR-009 | Salary: DateFrom / SalaryAmount | ExpectedStartDate / Salary | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_IntReq_to_WriteSalatyDat.xsl | Candidates to Salary and TalentProfile DAT |
| MR-010 | Salary: SalaryBasisName | Location | transformed |  | Salary_Basis.dvm on substring-after(Location, " ") | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Salary_Basis.dvm | Candidates to Salary and TalentProfile DAT |
| MR-011 | Salary: MultipleComponents / SalaryApproved / ActionCode | (none) | hardcoded |  | "N" / "Y" / "HIRE" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_IntReq_to_WriteSalatyDat.xsl | Candidates to Salary and TalentProfile DAT |
| MR-012 | TalentProfile: ProfileCode, PersonNumber, codes | ID, EmployeeID | calculated | EmployeeID != "" | K + "_PER_PROFILE"; ProfileTypeCode PERSON; status A; usage P | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_IntReq_to_WriteTalentProfile.xsl | Candidates to Salary and TalentProfile DAT |
| MR-013 | ProfileItem PREV_EMPLOY (1..3) | PreviousEmploymentDetails[1..3] | conditional | start and end dates present | ItemText2401 employer, 2402 address, 2403 city, 2404 postcode, 2407 position, ItemText308 salary; DateFrom/DateTo | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_IntReq_to_WriteTalentProfile.xsl | Candidates to Salary and TalentProfile DAT |
| MR-014 | EmployeeId, CandidateId, FirstName, MiddleName, LastName | EmployeeID, ID, names | looped | each Employee |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_PH_Write.xsl | Candidates to consolidated row |
| MR-015 | Status / Message | EmployeeID, ContractEndDate | conditional | EmployeeID != "" | Success + "Employee Created Successfully" else Failure + error text | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_US_Write.xsl | Candidates to consolidated row |
| MR-016 | EmployeeID, NationalIDNumber/SSSNumber, CountryCode, EffectiveStartDate, AllowanceType, Allowance | EmployeeID, SSSNo, Country/CountryCode, ExpectedStartDate, TaxableAllowance, NonTaxableAllowance | looped | EmployeeID != "" | CountryCode PH/US from Country name | low | hcm-pbl-allowances uses SSSNumber as PersonNumber. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_int_to_pbl.xsl | Candidates to PBL request |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Success definition | A hire is successful when HCM returns a person number for the candidate. | High | Process Call hcm-get-person-number + Decision. |
| BR-002 | Error message source | Failure message = first HDL MSG_TEXT for the candidate's load, else a generic text. | Medium | Lookup contentID in the agreed store + hcm-get-hdl-errors. |
| BR-003 | Follow-up loads only for successes | Salary, talent profile and PBL only for candidates with an EmployeeID. | High | Filter before maps. |
| BR-004 | Country routing | PH lists to HCMListEmail/hris_Jobvite, US lists to USHCMListEmail/UShrisJobvite (legacy sends to both when both fields are filled). | Medium | Decision on CountryCode only. |
| BR-005 | Salary basis by location | Salary basis from Salary_Basis table keyed on the location name after the first space. | Medium | Cross reference Salary_Basis. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound REST / JSON | https://129.144.156.28:443/soa-infra/resources/default/ProcessDatProfileGeneration!2.0/JobVite_Rest | none declared (no OWSM policy in the project) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/ProcessINTReadFileABCSImpl_prodJobVite_cfgplan.xml |
| outbound REST over HTTPS | https://api.jobvite.com/api/v2/candidate | API key and secret in query string [redacted] | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/WADLs/JobvitePutCandidate.wadl |
| outbound SOAP async (composites) | oramds:/apps/Provider/HCM/ProcessGetPersonNumber.wsdl, gethdlerrorsprovabcsimpl_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/composite.xml |
| outbound SOAP async (composites, no wait) | oramds:/apps/Provider/HCM/processhdlprovabcsimplprocess_client_ep.wsdl, processpblprovabcsimpl_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/composite.xml |
| outbound SOAP (composite) | oramds:/apps/Utilities/Notification/filenotification_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/composite.xml |
| outbound File (SOA server) | Domain.dvm Email folder (/u01/data/domains/SOACS-Pr_domain/osi_file/EmailAttachment) and DatFiles folder (/u01/data/domains/SOACS-Pr_domain/osi_file/put) | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Domain.dvm |
| File WriteEmailFile / WriteHRISEmail | Append false; names set at runtime HCMList_<ts>.csv / HCMList_hris_<ts>.csv; directory Domain.dvm Email. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Adapters/WriteEmailFile_file.jca |
| File WriteSalaryDat / WriteTalentProfile | Salary.dat / TalentProfile.dat, Append false; directory Domain.dvm DatFiles (shared HDL folder). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Adapters/WriteSalaryDat_file.jca |
| File Write_Consolidated_File | Append true; names set at runtime JobviteConsolidate_PH.csv / _US.csv; directory Domain.dvm Email. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Adapters/Write_Consolidated_File_file.jca |
| REST JobvitePutCandidate | PUT / with Root-Element JSON (candidate.application.eId, customField[fieldCode,value]). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/WADLs/JobvitePutCandidate.wadl |
| Cross reference XRef_UCMID | lookupXRef candidateID -> contentID; markForDelete after processing. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/XRef_UCMID.xref |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-get-person-number | Sub: HCM Get Person Number | synchronous |
| hcm-get-hdl-errors | Sub: HCM Get HDL Errors | synchronous |
| hcm-hdl-loader | Sub: HCM HDL Zip Checkin and Load | synchronous |
| hcm-pbl-allowances | Sub: HCM PBL Allowance Load | synchronous |
| util-file-notification | Sub: File Notification | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |
| gsuite-user-lifecycle | Google account creation (GSuite variant only) | synchronous |
| hcm-update-employee-email | Work email update (GSuite variant only) | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_IntReq_to_WriteSalatyDat.xsl | Salary.dat for successful candidates; Salary_Basis.dvm (Location_Name -> Salary_Basis_Name, 6 rows) on the part of Location after the first space. | map-int-to-salary-talent-dat. | Medium |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_IntReq_to_WriteTalentProfile.xsl | TalentProfile.dat with PREV_EMPLOY items for up to three previous employers. | map-int-to-salary-talent-dat. | Medium |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_hcm_to_jobvite.xsl | Jobvite candidate update (success / error). | map-int-to-jobvite-status. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_writeEmailfile.xsl | HCM list (all input fields) and HRIS list. | map-int-to-hcmlist. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_PH_Write.xsl | Consolidated success/failure rows. | map-int-to-consolidated. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_int_to_pbl.xsl | INT record to EBM for PBL (PersonNumber from SSSNo). | Map to Employee EBM for hcm-pbl-allowances. | Medium |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Process catchAll emails the raw fault; per-candidate scopes swallow Jobvite errors. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| XRef_UCMID (XREF_UCMID_TABLE) | crossref | no |
| Domain.dvm, EmailDL.dvm, Salary_Basis.dvm | dvm | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Jobvite API v2 | HTTPS REST | https://api.jobvite.com/api/v2/candidate | Jobvite |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Inbound endpoint (from caller plan) | https://129.144.156.28:[redacted]!2.0/JobVite_Rest | [redacted].xml |

## 14. Boomi Migration Readiness Summary

After a hire has been sent into HCM, this service checks whether it worked and finishes the job. For each candidate it asks HCM for the new person number. If HCM has the person, it marks the candidate in Jobvite as successfully integrated with their employee number. If not, it looks up the HCM Data Loader errors for that candidate's load and writes the error back to Jobvite. Either way it emails HR lists of the hires (one detailed list, one HRIS summary) and adds a success or failure line to the day's consolidated file, which a nightly job sends out at 23:30.

For the hires that succeeded it then creates two more HCM data files (salary, using a salary-basis table by location, and talent profile with up to three previous employers) and starts the HCM load, and it sends the allowances to the payroll allowance service.

Several points need attention. The Jobvite API key and secret are written in the service address and must be replaced. The detailed HR list contains salary, visa and tax ID data and is emailed as an attachment. The country check sends lists to both the Philippine and US recipients whenever both country fields are filled, which is almost always. The payroll call identifies employees by SSS number while the real HCM person number is available. Error text is carried in an unrelated field ("ContractEndDate"). Four versions of this service exist; one of them also creates Google accounts, so TaskUs must confirm which is deployed.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Jobvite Candidate Update Req JSON Prof | prof-jobvite-candidate-json | profile.json | — |
| 2 | Jobvite Employees Create Req JSON Prof | prof-jobvite-employees-json | profile.json | — |
| 3 | HCM HDL Salary DAT Prof | prof-hdl-salary-dat | profile.flatfile | — |
| 4 | HCM HDL TalentProfile DAT Prof | prof-hdl-talentprofile-dat | profile.flatfile | — |
| 5 | HCM List Report CSV Prof | prof-hcmlist-csv | profile.flatfile | — |
| 6 | HRIS List Report CSV Prof | prof-hris-list-csv | profile.flatfile | — |
| 7 | Jobvite Consolidated Row CSV Prof | prof-jobvite-consolidated-csv | profile.flatfile | — |
| 8 | Jobvite API Conn | conn-jobvite-api | connector-settings | — |
| 9 | Jobvite Candidate UPDATE Op | op-jobvite-put-candidate | connector-action | conn-jobvite-api |
| 10 | Map \| INT Candidate to Consolidated Row | map-int-to-consolidated | transform.map | prof-jobvite-employees-json, prof-jobvite-consolidated-csv |
| 11 | Map \| INT Candidate to HCM and HRIS Lists | map-int-to-hcmlist | transform.map | prof-jobvite-employees-json, prof-hcmlist-csv, prof-hris-list-csv |
| 12 | Map \| INT Candidate to Jobvite Status Update | map-int-to-jobvite-status | transform.map | prof-jobvite-employees-json, prof-jobvite-candidate-json |
| 13 | Map \| INT Candidate to Salary and TalentProfile DAT | map-int-to-salary-talent-dat | transform.map | prof-jobvite-employees-json, prof-hdl-salary-dat, prof-hdl-talentprofile-dat |
| 14 | Sub: DAT Profile Generation Core | proc-sub-dat-profile-core | process | op-jobvite-put-candidate, map-int-to-jobvite-status, map-int-to-hcmlist, map-int-to-salary-talent-dat, map-int-to-consolidated, proc-sub-hcm-get-person-number, proc-sub-hcm-get-hdl-errors, proc-sub-hdl-load, proc-sub-hcm-pbl-allowances, proc-sub-file-notification, proc-sub-send-notification |
| 15 | DAT Profile Generation REST Listener | proc-dat-profile-generation-rest | process | prof-jobvite-employees-json, proc-sub-dat-profile-core |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/BPEL/ProcessDatProfileGeneration.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/ProcessDatProfileGeneration_prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessDatProfileGeneration/SOA/BPEL/ProcessDatProfileGeneration.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessDatProfileGeneration/SOA/BPEL/ProcessDatProfileGeneration.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/WADLs/JobVite_Rest.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/WADLs/JobvitePutCandidate.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/WADLs/ReadFileService_Employees.json
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Schemas/ReadFileService.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Schemas/UpdateCandidateReq.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Schemas/hcmemaillist_request.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Schemas/hris_email_list.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Schemas/write_SalaryDat_Request.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Schemas/write_talentProfile_Request.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Schemas/WriteConsolidateFile.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/XRef_UCMID.xref
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_hcm_to_jobvite.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_hcm_to_jobvite_error.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_writeEmailfile.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_hris_writeEmail.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_IntReq_to_WriteSalatyDat.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_IntReq_to_WriteTalentProfile.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_int_to_pbl.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_PH_Write.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_US_Write.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Transformations/xform_hdl_provider_req.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Adapters/Write_Consolidated_File_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Adapters/WriteSalaryDat_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Adapters/WriteTalentProfile_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Adapters/WriteEmailFile_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/Adapters/WriteHRISEmail_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Domain.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Salary_Basis.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessINTReadFileABCSImpl/ProcessINTReadFileABCSImpl_prodJobVite_cfgplan.xml

## Open questions

### Still outstanding

REVIEW-NEEDED: Which version of the new-hire follow-up service is running in production: the Jobvite version, or the version that also creates Google accounts and updates the work email? (Four variants exist. The GSuite variant additionally creates Google accounts here and updates the HCM work email; if that variant is live, Google accounts for recruited hires are created in this flow and not only in newhire-emp-sch. Confirm which variant is deployed.)
  - If this is not answered: The Employee_JobVite version (no Google creation here).

REVIEW-NEEDED: Where should the day's hire results be kept for the 23:30 summary job once the server files go away? (Legacy appends consolidated rows to JobviteConsolidate_PH.csv / _US.csv on the server for the 23:30 job to pick up. Decide the Boomi staging mechanism (a database table, or a document cache plus the day's executions).)
  - If this is not answered: A small database table, written per hire and read and cleared at 23:30.

REVIEW-NEEDED: Can TaskUs issue new Jobvite API credentials for Boomi and revoke the ones in the current integration? (The Jobvite api and sc keys are embedded in the reference URL in composite.xml (redacted) and must be rotated and re-issued for Boomi.)
  - If this is not answered: New keys issued and stored in the Boomi Jobvite connection.

REVIEW-NEEDED: Should the HR hire lists go only to the team for the hire's country? (The legacy country condition also sends to both PH and US lists whenever both Country and CountryCode are populated, which is probably a defect; confirm the routing.)
  - If this is not answered: Send only to the hire's country team.

REVIEW-NEEDED: Does the detailed HR list need salary, visa and tax ID data, given that it is emailed to a distribution list? (The list contains every field of the hire record.)
  - If this is not answered: Reduce the list to identification, job and status fields.

REVIEW-NEEDED: Should allowances be loaded using the HCM person number rather than the SSS number? (The payroll allowance load identifies employees by SSS number, while this service already knows the real person number.)
  - If this is not answered: Use the HCM person number.

