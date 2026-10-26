# Migration Plan — ProcessDATGenerationABCSImpl

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.309Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessDATGenerationABCSImpl |
| Identifier | hcm-dat-generation |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | complex (70) |
| Components to build | 7 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/BPEL/ProcessDATGenerationABCSImpl.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessDATGenerationABCSImpl/SOA/BPEL/ProcessDATGenerationABCSImpl.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/XRef_UCMID.xref |
| Depends on | hcm-hdl-loader, util-send-notification |

## 2. End-to-End Process Architecture

Async BPEL composite: two XSLT maps to NXSD flat files written by File adapters into the shared folder, async call to ProcessHDLProvABCSImpl, SOA cross-reference table for candidate -> content, XXSendNotification on error. The Employee copy is an older variant that returns the contentID.

Boomi sub-process "Sub: HCM Worker DAT Generation": Decision on Action, map-ebm-to-hdl-worker or map-ebm-to-hdl-rehire (cross reference tables for the DVMs), Process Call hcm-hdl-loader with the in-memory Worker.dat, persist candidateID -> contentID in an agreed store, return prof-transmission-ack-xml with status and contentID; Try/Catch with util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback | Sub-process start + Return Documents | Return the content ID. |
| xform_hcmEBM_to_basicworker / rehireworker | map-ebm-to-hdl-worker / map-ebm-to-hdl-rehire |  |
| File adapters + shared folder | In-memory document |  |
| ProcessHDLProvABCSImpl | Process Call hcm-hdl-loader |  |
| XRef_UCMID (XREF_UCMID_TABLE) | Database table or other persistent store | Boomi Cross Reference Tables are static. |
| DVMs | Cross Reference Tables |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive employee | Async SOAP process with EmployeeEBM EmployeeRequest. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/WSDLs/ProcessDATGenerationABCSImpl.wsdl |
| 2 | Rehire? | Action = REHIRE -> NationalIDNumber := EmployeeID and rehire map; else hire map. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/BPEL/ProcessDATGenerationABCSImpl.bpel |
| 3 | Write Worker.dat | BasicWorker or WriteRehireWorker File adapter; pipe-delimited METADATA/MERGE lines. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Adapters/BasicWorker_file.jca |
| 4 | Load into HCM | xform_hdl_provider_req (CHECKIN_UNIVERSAL, dDocTitle Worker, Worker.zip) -> ProcessHDLProvABCSImpl; wait for callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hdl_provider_req.xsl |
| 5 | Record content ID | If contentID returned: xref:populateXRefRow XRef_UCMID candidateID -> contentID (ADD); fault ignored. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/XRef_UCMID.xref |
| 6 | Callback | processResponse (empty in this variant). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/BPEL/ProcessDATGenerationABCSImpl.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| Employee EBM Request XML Prof | xml | EmployeeRequest/TransmissionHeader | complex | no |
| Employee EBM Request XML Prof | xml | EmployeeRequest/Action | string | no |
| Employee EBM Request XML Prof | xml | EmployeeRequest/Employee | EmployeeType (repeating) | yes |
| Employee EBM Request XML Prof | xml | Employee/EmployeeID, CandidateID, ReqNumber, Campaign, CountryBirth, MaritalStatus, Gender, DateOfBirth | string | no |
| Employee EBM Request XML Prof | xml | Employee/Person (PersonType) | complex | no |
| Employee EBM Request XML Prof | xml | Employee/Person/NationalID_Type (NationalType) | complex | no |
| Employee EBM Request XML Prof | xml | Employee/ContactDetails (ContactType) | complex (repeating) | no |
| Employee EBM Request XML Prof | xml | Employee/EmploymentAssignments (EmploymentAssignmentType) | complex | no |
| Employee EBM Request XML Prof | xml | Employee/EmployeeAllowanceType | complex | no |
| Employee EBM Request XML Prof | xml | Employee/EmployeePSFTType | complex | no |
| Employee Transmission Ack XML Prof | xml | EmployeeResponse/TransmissionAck | TransmissionAckType (repeating) | no |
| Employee Transmission Ack XML Prof | xml | TransmissionAck/TransmissionAckStatus | TransmissionMessageType | no |
| HCM HDL Worker DAT Prof | flatfile | <Object> METADATA line | string | yes |
| HCM HDL Worker DAT Prof | flatfile | <Object> MERGE line | string | yes |
| HCM HDL Rehire Worker DAT Prof | flatfile | WorkRelationship / WorkTerms / Assignment / Contract METADATA + MERGE lines | string | yes |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | Worker: SourceSystemId / EffectiveStartDate / StartDate / DateOfBirth | NationalIDNumber, EffectiveStartDate, OriginalHireDate, DateOfBirth | calculated |  | K + "_PERSON"; dates passthrough | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-002 | Worker: CountryOfBirth / ActionCode / ReasonCode / WaiveDataProtectFlag | CountryBirth, /EmployeeRequest/Action, ActionReason | transformed |  | Countries DVM Country->Code; "N" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-003 | PersonName: First/Middle/Last, KnownAs, Title, NameType | Person/FirstName, MiddleName, LastName, PreferredName, Prefix | passthrough |  | NameType "GLOBAL"; LegislationCode via LegislationCode DVM | low | Title = salutation LookupCode set by hcm-employee-provisioning. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-004 | PersonAddress | ContactDetails/ContactAddress/* | transformed |  | AddressLine1/2, TownOrCity=City, Region1=Region, PostalCode=ZipCode, Country via Countries DVM | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-005 | PersonPhone | ContactDetails/Phone, ContactAddress/AreaCode | conditional | Country != "United States" | PhoneType via PhoneTypes DVM; CountryCodeNumber via CountryCodeNumber DVM; PrimaryFlag Y | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-006 | PersonEmail | Person/EmailAddress, EmailAddressType | transformed |  | EmailType via EmailTypes DVM; PrimaryFlag Y | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-007 | PersonNationalIdentifier (up to 4 rows) | Person/NationalID_Type/SSSNumber, TINNumber, PhilhealthNumber, HDMFNumber | conditional | value not empty; TIN/PhilHealth/HDMF only for Philippines | ORA_HRX_SSS (primary), TAX_ID, ORA_HRX_PHILHEALTH_NUM, ORA_HRX_COM_REF_NUM | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-008 | PersonLegislativeData: MaritalStatus / Sex | MaritalStatus, Gender | transformed |  | MaritalStatus DVM Meaning->LookupCode; Male->M, Female->F | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-009 | PersonPassport / PersonVisa | Person/Visa/* | conditional | Country != "United States" | Passport number/dates; visa type/number/dates; IssuingCountry via Countries; CurrentVisaPermit Y | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-010 | WorkRelationship | EffectiveStartDate, Person/LegalEntityName | calculated |  | K + "_WorkRelationship"; WorkerType E; PrimaryFlag Y | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-011 | WorkTerms | BusinessUnitName, JobCode, Department/Name, EmployeeType | calculated |  | AssignmentNumber "WT_ASS_" + K; AssignmentType ET; ACTIVE_PROCESS; PersonTypeCode Employee; SystemPersonType EMP | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-012 | Contract | LatestCSWDate, EmploymentType | conditional | Country != "United States" | EffectiveEndDate = LatestCSWDate; ContractType = EmploymentType | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-013 | Assignment (+ PER_ASG_DF flexfield) | Campaign, CandidateID, LineOfBusiness, WorkSchedule, JobGrade, WorkLocation, IsFullPartTime, ProbationEndDate | calculated | wsRule only when Department is in Departments.dvm (else S540006) | AssignmentNumber "ASSIGNMENT_" + K; projectId, candidateId, lob, wsRule; DateProbationEnd +179 days | high |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-014 | AssignmentSupervisor | EmployeePSFTType/SupervisorEmpID | calculated |  | ManagerType LINE_MANAGER; SourceSystemId "SUP_ASSN_" + K | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | Employee EBM to HDL Worker.dat (hire) |
| MR-015 | WorkRelationship | EffectiveStartDate, LegalEntityName | calculated | Action = REHIRE | ActionCode REHIRE; SourceSystemId "WR_" + K | low | K = EmployeeID for rehires. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_rehireworker.xsl | Employee EBM to HDL Worker.dat (rehire) |
| MR-016 | WorkTerms | BusinessUnitName, EffectiveStartDate | calculated |  | "WT_" + K; AssignmentNumber "ET_" + K; EffectiveEndDate 4712/12/31 | low | Differs from hire keys. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_rehireworker.xsl | Employee EBM to HDL Worker.dat (rehire) |
| MR-017 | Assignment | JobCode, Department, WorkLocation, BusinessUnitName | calculated |  | "ASSN_" + K; AssignmentNumber "E" + K; FULL_TIME; HourlySalariedCode #NULL | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_rehireworker.xsl | Employee EBM to HDL Worker.dat (rehire) |
| MR-018 | Contract | PersonOrContractType | calculated |  | "PER_CONTRACT_" + K; EffectiveEndDate 4712/12/31 | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_rehireworker.xsl | Employee EBM to HDL Worker.dat (rehire) |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | HDL key scheme | All objects keyed TASKUS + national ID number + object suffix. | High | Same expressions in both maps; must not change for existing persons. |
| BR-002 | US exclusions | Phone, passport, visa and contract records are not sent for US addresses. | Medium | Conditional map sections. |
| BR-003 | PH identifiers | TIN, PhilHealth and HDMF identifiers only for the Philippines; SSS always when present. | High | Conditional rows. |
| BR-004 | Work schedule rule | wsRule from the input only for departments in Departments.dvm; otherwise S540006. | Medium | Cross reference Departments. |
| BR-005 | Probation | ProbationPeriod 180 days; probation end = given date + 179 days. | Medium | Map date function. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | processdatgenerationabcsimpl_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/composite.xml |
| outbound File (SOA server) | /u01/data/domains/SOACS-Pr_domain/osi_file/put/Worker.dat | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/ProcessDATGenerationABCSImpl_prod_cfgplan.xml |
| outbound SOAP async (composite) | http://129.144.156.28:80/soa-infra/services/default/ProcessHDLProvABCSImpl/processhdlprovabcsimplprocess_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/processhdlprovabcsimplprocess_client_ep.wsdl |
| outbound Database (SOA XREF) | XREF_UCMID_TABLE | SOA infrastructure data source | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/XRef_UCMID.xref |
| File adapter BasicWorker | eis/FileAdapter; LogicalDirectory LogicalPath (PROD [redacted]); FileNamingConvention Worker.dat; Append false; NXSD oramds:/apps/Utilities/XSD/BasicWorker.xsd. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Adapters/BasicWorker_file.jca |
| File adapter WriteRehireWorker | Same folder; Worker.dat; NXSD Schemas/RehireWorker.xsd. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Adapters/WriteRehireWorker_file.jca |
| File adapters WriteSalaryDat / WriteSalaryBasis / writeTalentProfile | Salary.dat, SalaryBasis.dat, TalentProfile.dat; all invocations skipped. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/composite.xml |
| Cross reference XRef_UCMID | Columns candidateID, contentID; dbtable XREF_UCMID_TABLE. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/XRef_UCMID.xref |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-hdl-loader | Sub: HCM HDL Zip Checkin and Load | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | EBM to the 14-object hire Worker.dat (1,585 lines). | map-ebm-to-hdl-worker into prof-hdl-worker-dat with cross references. | High |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_rehireworker.xsl | EBM to rehire Worker.dat. | map-ebm-to-hdl-rehire into prof-hdl-rehire-dat. | Medium |
| other | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl | xp20:add-dayTimeDuration-to-dateTime for probation end; dvm:lookupValue for eight tables. | Date functions and Cross Reference Table lookups in the map. | Medium |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll: notify EmailDL itservicedesk, no callback; XREF errors swallowed. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| EmployeeEBM / EmployeeEBO / PersonEBO / CommonEBO / AddressBookEBO (PROD_MDS snapshot) | mds | yes |
| oramds:/apps/Utilities/XSD/BasicWorker.xsd | mds | yes |
| XRef_UCMID data (XREF_UCMID_TABLE) | crossref | no |

## 12. External Services

> **Not available.** No external services were recorded for this interface.

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Worker.dat folder | [redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

When a new hire is ready to be created in Oracle HCM, this service builds the HCM Data Loader file that creates them and hands it to the HCM loading service. The file covers the person (name, date of birth, address, phone, email, national identifiers such as SSS, TIN, PhilHealth and Pag-IBIG, marital status, gender, passport and visa), the employment relationship, terms and assignment (business unit, job, department, location, grade, project, line of business, work schedule, probation) and the line manager. A rehire produces a smaller file that restarts the employment.

All HCM records are keyed on the national ID number supplied by the caller, so that number must never change for an existing person. The rehire file uses different record keys and assignment numbers from the hire file (for example "E<id>" and "ET_<id>" instead of "ASSIGNMENT_<id>" and "WT_ASS_<id>"), which should be checked with the HCM team.

The service writes the file to the shared server folder used by the HCM loader, then records which HCM load belongs to which candidate in a database cross-reference table. Errors from that step are ignored, and the caller never receives a result: it does not wait. Salary, salary basis and talent-profile files are switched off here. In Boomi the file is built in memory by two maps and passed straight to the shared HCM loading sub-process; the candidate-to-load link needs a persistent store because Boomi cross-reference tables are static.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Employee EBM Request XML Prof | prof-employee-ebm-xml | profile.xml | — |
| 2 | Employee Transmission Ack XML Prof | prof-transmission-ack-xml | profile.xml | — |
| 3 | HCM HDL Rehire Worker DAT Prof | prof-hdl-rehire-dat | profile.flatfile | — |
| 4 | HCM HDL Worker DAT Prof | prof-hdl-worker-dat | profile.flatfile | — |
| 5 | Map \| Employee EBM to HDL Rehire DAT | map-ebm-to-hdl-rehire | transform.map | prof-employee-ebm-xml, prof-hdl-rehire-dat |
| 6 | Map \| Employee EBM to HDL Worker DAT | map-ebm-to-hdl-worker | transform.map | prof-employee-ebm-xml, prof-hdl-worker-dat |
| 7 | Sub: HCM Worker DAT Generation | proc-sub-hcm-dat-generation | process | map-ebm-to-hdl-worker, map-ebm-to-hdl-rehire, proc-sub-hdl-load, prof-transmission-ack-xml, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/BPEL/ProcessDATGenerationABCSImpl.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessDATGenerationABCSImpl/SOA/BPEL/ProcessDATGenerationABCSImpl.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/XRef_UCMID.xref
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/ProcessDATGenerationABCSImpl_prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/WSDLs/ProcessDATGenerationABCSImpl.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/WSDLs/BasicWorker.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/WSDLs/WriteRehireWorker.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_basicworker.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hcmEBM_to_rehireworker.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Transformations/xform_hdl_provider_req.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/XSD/BasicWorker.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Schemas/BasicWorker.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Schemas/RehireWorker.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Adapters/BasicWorker_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDATGenerationABCSImpl/SOA/Adapters/WriteRehireWorker_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/EnterpriseObjectLibrary/Core/EBO/Employee/V1/EmployeeEBM.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/EnterpriseObjectLibrary/Core/EBO/Employee/V1/EmployeeEBO.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/EnterpriseObjectLibrary/Core/EBO/AddressBook/V1/PersonEBO.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/EnterpriseObjectLibrary/Core/Common/V1/CommonEBO.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/EnterpriseObjectLibrary/Core/EBO/AddressBook/V1/AddressBookEBO.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/processhdlprovabcsimplprocess_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessEmployeeHCMProvABCSImpl/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm

## Open questions

### Still outstanding

REVIEW-NEEDED: Where should the link between each candidate and their HCM load be kept, now that the legacy SOA database table goes away? (The XREF data (XREF_UCMID_TABLE) is not in the export, and Boomi Cross Reference Tables are static. Decide where the candidateID->contentID map lives at runtime; dat-profile-generation-rest reads it to fetch HDL errors.)
  - If this is not answered: A small table in a TaskUs-managed database reached through a Boomi Database connection.

REVIEW-NEEDED: Which employee data layout is used in production for new hires: the original (2017) layout or the Jobvite layout with repeating education and previous-employment details? (The PROD_MDS EmployeeEBM.xsd is the 2017 layout. Confirm which EBO the production DAT generation is compiled against, because the profile depends on it.)
  - If this is not answered: The PROD_MDS Employee/V1 layout.

REVIEW-NEEDED: Should rehires reuse the original assignment and work-terms numbers, or are the different numbers used today ("E<id>", "ET_<id>") intended? (Hire and rehire files use different key patterns; mismatches can create duplicate assignments in HCM.)
  - If this is not answered: Keep the legacy patterns unchanged.

REVIEW-NEEDED: Are salary, salary basis and talent-profile records for new hires created elsewhere, so that they can stay out of this service? (Those files are switched off here; dat-profile-generation-rest appears to produce them.)
  - If this is not answered: Yes, they stay in dat-profile-generation-rest.

