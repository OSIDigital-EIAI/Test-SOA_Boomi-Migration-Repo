# ProcessDATGenerationABCSImpl

Test-OracleSOA-Project · Listener · complex complexity

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

Turns a new hire or rehire received as an Employee EBM into an HCM Data Loader Worker.dat file and loads it into Oracle HCM. For a hire it produces the full worker record (Worker, PersonName, PersonAddress, PersonPhone, PersonEmail, PersonNationalIdentifier, PersonLegislativeData, PersonPassport, PersonVisa, WorkRelationship, WorkTerms, Contract, Assignment with descriptive flexfields, AssignmentSupervisor); for a rehire it produces WorkRelationship, WorkTerms, Assignment and Contract records with action REHIRE. It then calls the HDL loader and records candidateID -\> UCM contentID in a cross-reference table.

| Field | Value |
| --- | --- |
| Interface | ProcessDATGenerationABCSImpl |
| Type | Listener |
| Complexity | complex |
| Components to build | 7 |
| Mapping rules | 18 |
| Business rules | 5 |

## Interface Summary

When a new hire is ready to be created in Oracle HCM, this service builds the HCM Data Loader file that creates them and hands it to the HCM loading service. The file covers the person (name, date of birth, address, phone, email, national identifiers such as SSS, TIN, PhilHealth and Pag-IBIG, marital status, gender, passport and visa), the employment relationship, terms and assignment (business unit, job, department, location, grade, project, line of business, work schedule, probation) and the line manager. A rehire produces a smaller file that restarts the employment.  All HCM records are keyed on the national ID number supplied by the caller, so that number must never change for an existing person. The rehire file uses different record keys and assignment numbers from the hire file (for example "E\<id\>" and "ET\_\<id\>" instead of "ASSIGNMENT\_\<id\>" and "WT\_ASS\_\<id\>"), which should be checked with the HCM team.  The service writes the file to the shared server folder used by the HCM loader, then records which HCM load belongs to which candidate in a database cross-reference table. Errors from that step are ignored, and the caller never receives a result: it does not wait. Salary, salary basis and talent-profile files are switched off here. In Boomi the file is built in memory by two maps and passed straight to the shared HCM loading sub-process; the candidate-to-load link needs a persistent store because Boomi cross-reference tables are static.

**Complexity:** complex (score 70)

## Functional Analysis

1. **Receive employee** — Async SOAP process with EmployeeEBM EmployeeRequest.
2. **Rehire?** — Action = REHIRE -\> NationalIDNumber := EmployeeID and rehire map; else hire map.
3. **Write Worker.dat** — BasicWorker or WriteRehireWorker File adapter; pipe-delimited METADATA/MERGE lines.
4. **Load into HCM** — xform\_hdl\_provider\_req (CHECKIN\_UNIVERSAL, dDocTitle Worker, Worker.zip) -\> ProcessHDLProvABCSImpl; wait for callback.
5. **Record content ID** — If contentID returned: xref:populateXRefRow XRef\_UCMID candidateID -\> contentID (ADD); fault ignored.
6. **Callback** — processResponse (empty in this variant).

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | processdatgenerationabcsimpl\_client\_ep | none declared |
| outbound | File (SOA server) | /u01/data/domains/SOACS-Pr\_domain/osi\_file/put/Worker.dat | n/a |
| outbound | SOAP async (composite) | http://129.144.156.28:80/soa-infra/services/default/ProcessHDLProvABCSImpl/processhdlprovabcsimplprocess\_client\_ep | none declared |
| outbound | Database (SOA XREF) | XREF\_UCMID\_TABLE | SOA infrastructure data source |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Employee EBM to HDL Worker.dat (hire) | Worker: SourceSystemId / EffectiveStartDate / StartDate / DateOfBirth | NationalIDNumber, EffectiveStartDate, OriginalHireDate, DateOfBirth | calculated | — | K + "\_PERSON"; dates passthrough | low | — |
| MR-002 | Employee EBM to HDL Worker.dat (hire) | Worker: CountryOfBirth / ActionCode / ReasonCode / WaiveDataProtectFlag | CountryBirth, /EmployeeRequest/Action, ActionReason | transformed | — | Countries DVM Country-\>Code; "N" | low | — |
| MR-003 | Employee EBM to HDL Worker.dat (hire) | PersonName: First/Middle/Last, KnownAs, Title, NameType | Person/FirstName, MiddleName, LastName, PreferredName, Prefix | passthrough | — | NameType "GLOBAL"; LegislationCode via LegislationCode DVM | low | Title = salutation LookupCode set by hcm-employee-provisioning. |
| MR-004 | Employee EBM to HDL Worker.dat (hire) | PersonAddress | ContactDetails/ContactAddress/\* | transformed | — | AddressLine1/2, TownOrCity=City, Region1=Region, PostalCode=ZipCode, Country via Countries DVM | low | — |
| MR-005 | Employee EBM to HDL Worker.dat (hire) | PersonPhone | ContactDetails/Phone, ContactAddress/AreaCode | conditional | Country != "United States" | PhoneType via PhoneTypes DVM; CountryCodeNumber via CountryCodeNumber DVM; PrimaryFlag Y | medium | — |
| MR-006 | Employee EBM to HDL Worker.dat (hire) | PersonEmail | Person/EmailAddress, EmailAddressType | transformed | — | EmailType via EmailTypes DVM; PrimaryFlag Y | low | — |
| MR-007 | Employee EBM to HDL Worker.dat (hire) | PersonNationalIdentifier (up to 4 rows) | Person/NationalID\_Type/SSSNumber, TINNumber, PhilhealthNumber, HDMFNumber | conditional | value not empty; TIN/PhilHealth/HDMF only for Philippines | ORA\_HRX\_SSS (primary), TAX\_ID, ORA\_HRX\_PHILHEALTH\_NUM, ORA\_HRX\_COM\_REF\_NUM | medium | — |
| MR-008 | Employee EBM to HDL Worker.dat (hire) | PersonLegislativeData: MaritalStatus / Sex | MaritalStatus, Gender | transformed | — | MaritalStatus DVM Meaning-\>LookupCode; Male-\>M, Female-\>F | low | — |
| MR-009 | Employee EBM to HDL Worker.dat (hire) | PersonPassport / PersonVisa | Person/Visa/\* | conditional | Country != "United States" | Passport number/dates; visa type/number/dates; IssuingCountry via Countries; CurrentVisaPermit Y | medium | — |
| MR-010 | Employee EBM to HDL Worker.dat (hire) | WorkRelationship | EffectiveStartDate, Person/LegalEntityName | calculated | — | K + "\_WorkRelationship"; WorkerType E; PrimaryFlag Y | low | — |
| MR-011 | Employee EBM to HDL Worker.dat (hire) | WorkTerms | BusinessUnitName, JobCode, Department/Name, EmployeeType | calculated | — | AssignmentNumber "WT\_ASS\_" + K; AssignmentType ET; ACTIVE\_PROCESS; PersonTypeCode Employee; SystemPersonType EMP | medium | — |
| MR-012 | Employee EBM to HDL Worker.dat (hire) | Contract | LatestCSWDate, EmploymentType | conditional | Country != "United States" | EffectiveEndDate = LatestCSWDate; ContractType = EmploymentType | low | — |
| MR-013 | Employee EBM to HDL Worker.dat (hire) | Assignment (+ PER\_ASG\_DF flexfield) | Campaign, CandidateID, LineOfBusiness, WorkSchedule, JobGrade, WorkLocation, IsFullPartTime, ProbationEndDate | calculated | wsRule only when Department is in Departments.dvm (else S540006) | AssignmentNumber "ASSIGNMENT\_" + K; projectId, candidateId, lob, wsRule; DateProbationEnd +179 days | high | — |
| MR-014 | Employee EBM to HDL Worker.dat (hire) | AssignmentSupervisor | EmployeePSFTType/SupervisorEmpID | calculated | — | ManagerType LINE\_MANAGER; SourceSystemId "SUP\_ASSN\_" + K | low | — |
| MR-015 | Employee EBM to HDL Worker.dat (rehire) | WorkRelationship | EffectiveStartDate, LegalEntityName | calculated | Action = REHIRE | ActionCode REHIRE; SourceSystemId "WR\_" + K | low | K = EmployeeID for rehires. |
| MR-016 | Employee EBM to HDL Worker.dat (rehire) | WorkTerms | BusinessUnitName, EffectiveStartDate | calculated | — | "WT\_" + K; AssignmentNumber "ET\_" + K; EffectiveEndDate 4712/12/31 | low | Differs from hire keys. |
| MR-017 | Employee EBM to HDL Worker.dat (rehire) | Assignment | JobCode, Department, WorkLocation, BusinessUnitName | calculated | — | "ASSN\_" + K; AssignmentNumber "E" + K; FULL\_TIME; HourlySalariedCode \#NULL | medium | — |
| MR-018 | Employee EBM to HDL Worker.dat (rehire) | Contract | PersonOrContractType | calculated | — | "PER\_CONTRACT\_" + K; EffectiveEndDate 4712/12/31 | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | HDL key scheme | All objects keyed TASKUS + national ID number + object suffix. | High |
| BR-002 | US exclusions | Phone, passport, visa and contract records are not sent for US addresses. | Medium |
| BR-003 | PH identifiers | TIN, PhilHealth and HDMF identifiers only for the Philippines; SSS always when present. | High |
| BR-004 | Work schedule rule | wsRule from the input only for departments in Departments.dvm; otherwise S540006. | Medium |
| BR-005 | Probation | ProbationPeriod 180 days; probation end = given date + 179 days. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive employee EBM** — From hcm-employee-provisioning
2. **Rehire?** — Action = REHIRE
3. **Build rehire Worker.dat** — 4 objects
4. **Build hire Worker.dat** — 14 objects
5. **Load into HCM** — hcm-hdl-loader
6. **Notify service desk** — No callback
7. **Content ID returned?**
8. **Store candidate-content link** — XREF; errors ignored
9. **Callback** — Empty output

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| Employee EBM Request XML Prof | xml | prof-employee-ebm-xml |
| Employee Transmission Ack XML Prof | xml | prof-transmission-ack-xml |
| HCM HDL Worker DAT Prof | flatfile | prof-hdl-worker-dat |
| HCM HDL Rehire Worker DAT Prof | flatfile | prof-hdl-rehire-dat |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback | Sub-process start + Return Documents |
| xform\_hcmEBM\_to\_basicworker / rehireworker | map-ebm-to-hdl-worker / map-ebm-to-hdl-rehire |
| File adapters + shared folder | In-memory document |
| ProcessHDLProvABCSImpl | Process Call hcm-hdl-loader |
| XRef\_UCMID (XREF\_UCMID\_TABLE) | Database table or other persistent store |
| DVMs | Cross Reference Tables |

**Components**

| Component | Role |
| --- | --- |
| prof-employee-ebm-xml | Employee EBM request profile |
| prof-transmission-ack-xml | Response profile |
| prof-hdl-worker-dat | Hire Worker.dat profile |
| prof-hdl-rehire-dat | Rehire Worker.dat profile |
| map-ebm-to-hdl-worker | Hire map |
| map-ebm-to-hdl-rehire | Rehire map |
| proc-sub-hcm-dat-generation | Main sub-process |

## Client questions

- **Q-001** Where should the link between each candidate and their HCM load be kept, now that the legacy SOA database table goes away?
  **If this is not answered:** A small table in a TaskUs-managed database reached through a Boomi Database connection.
- **Q-002** Which employee data layout is used in production for new hires: the original (2017) layout or the Jobvite layout with repeating education and previous-employment details?
  **If this is not answered:** The PROD\_MDS Employee/V1 layout.
- **Q-003** Should rehires reuse the original assignment and work-terms numbers, or are the different numbers used today ("E\<id\>", "ET\_\<id\>") intended?
  **If this is not answered:** Keep the legacy patterns unchanged.
- **Q-004** Are salary, salary basis and talent-profile records for new hires created elsewhere, so that they can stay out of this service?
  **If this is not answered:** Yes, they stay in dat-profile-generation-rest.
