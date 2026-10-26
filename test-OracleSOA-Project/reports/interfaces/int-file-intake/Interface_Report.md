# ProcessINTReadFileABCSImpl (file poll)

Test-OracleSOA-Project · Scheduled · complex complexity

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

Polls the Interviewed drop folder for new-hire CSV files, validates each hire (action type, candidate ID, job code, and for the Philippines an SSS number that does not already exist in HCM), sends valid hires to hcm-employee-provisioning, and after each hire waits three minutes and calls the DAT profile generation REST service. Invalid hires are reported to the HRIS Jobvite lists; unreadable files are reported by a fault policy.

| Field | Value |
| --- | --- |
| Interface | ProcessINTReadFileABCSImpl (file poll) |
| Type | Scheduled |
| Complexity | complex |
| Components to build | 6 |
| Mapping rules | 12 |
| Business rules | 4 |

## Interface Summary

Every minute this integration looks in a folder on the integration server for new-hire files from the recruiting system. Each file holds one line per hire with around 75 columns: personal details, IDs, address, education, previous employment, job, department, location, schedule, salary, allowances and supervisor. For each hire it checks the basic data and, for the Philippines, that the SSS number is present and not already used by someone in HCM. Valid hires are passed to HCM provisioning. Problems are emailed to the HRIS Jobvite list for the US or the rest of the world.  After each hire it waits three minutes, presumably to let HCM finish creating the worker, and then calls the DAT profile generation service. On the file path it passes the REST request, which is empty, instead of the hire's data, so that follow-up step likely receives nothing. The SSS check also looks at the first hire in the file rather than the current one. Because of the per-hire wait, a file of 20 hires takes at least an hour. Files that cannot be read are reported to the "Rejection" email list by a separate fault handler.  In Boomi this becomes a scheduled process with a Disk or SFTP connector, a flat-file profile and a validation sub-process shared with the Jobvite REST listener. The fixed wait should be replaced by checking the worker-creation result.

**Complexity:** complex (score 66)

## Functional Analysis

1. **Poll folder** — Read \*.csv from the Interviewed folder (recursive), delete and archive.
2. **Parse file** — NXSD ReadFileService (header line, comma-separated).
3. **Any employees?** — count(Employee) \> 0, else EmailDL notification "Input Request Empty".
4. **Validate action** — Hiring with no EmployeeID or Rehiring with EmployeeID; else "ActionType ... Empty" or "Action type not Valide".
5. **Validate candidate and job** — ID (candidate) and JobCode must be present; else notification.
6. **PH SSS checks** — CountryCode PH: SSSNo required ("The SSSNo is empty"); HCM emps?q=NationalId=SSSNo must be empty ("Employee Exists in HCM with given SSSNo").
7. **Provision** — Map to EmployeeEBM and call ProcessEmployeeHCMProvABCSImpl.
8. **Wait and follow up** — Wait 3 minutes; POST DatProfileGeneration JobVite\_Rest.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | File (SOA server) | /u01/data/domains/SOACS-Pr\_domain/Interviewed/\*.csv | n/a |
| outbound | REST over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCoreApi/resources/latest/emps?q=NationalId=\<SSSNo\> | HTTP Authorization header (Credentials.dvm) |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/processemployeehcmprovabcsimpl\_client\_ep.wsdl | none declared |
| outbound | REST over HTTPS (composite) | https://129.144.156.28:443/soa-infra/resources/default/ProcessDatProfileGeneration!2.0/JobVite\_Rest | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | INT new-hire record to Employee EBM | Employee/CandidateID and Person/NationalID\_Type/NationalIDNumber | ID | transformed | — | substring-before(ID, "-") | low | Candidate ID prefix becomes the HDL key in this variant. |
| MR-002 | INT new-hire record to Employee EBM | Employee/EmployeeID, Campaign | EmployeeID, Campaign | passthrough | — | — | low | — |
| MR-003 | INT new-hire record to Employee EBM | Person names, prefix/suffix, preferred name, email | FirstName, MiddleName, LastName, PrefixName, SuffixName, PreferredName, Email, EmailAddressType | transformed | — | normalize-space on Prefix | low | — |
| MR-004 | INT new-hire record to Employee EBM | NationalID\_Type SSS/TIN/Philhealth/HDMF, IssuingCountry | SSSNo, TINno, Philhealth, HDMFPagibig, NIDIssuingCountry | passthrough | — | — | low | — |
| MR-005 | INT new-hire record to Employee EBM | Visa / passport | Visa\*, Passport\*, CountryofIssueVISA | passthrough | — | — | low | — |
| MR-006 | INT new-hire record to Employee EBM | CountryBirth, MaritalStatus, Nationality, Religion, Gender, DateOfBirth | Birthplace, CivilStatus, Nationality, Religion, Gender, Dateofbirth | transformed | — | lower-case + trim of CivilStatus | low | — |
| MR-007 | INT new-hire record to Employee EBM | ContactAddress / Phone | AddressType, AddressLine1/2, PostalCode, City, Region, Country, CountryCode, AreaCode, PhoneType, PhoneNumber | passthrough | — | — | low | — |
| MR-008 | INT new-hire record to Employee EBM | EducationalDetails / PreviousEmploymentDetails | EducationDetails\[\], PreviousEmploymentDetails\[\] | looped | — | — | medium | — |
| MR-009 | INT new-hire record to Employee EBM | EmploymentAssignments (department, job, grade, BU, LOB, hours, location, dates, schedule, contract end, reason) | OrganizationDepartment, JobCode, SalaryGrade, BusinessUnit, LineofBusiness, NumOfWorkHours, Location, ExpectedStartDate, Schedule, WorkScheduleStartTime/EndTime, ContractEndDate, ActionReason, AssignmentCategory, ContractType | passthrough | — | — | low | — |
| MR-010 | INT new-hire record to Employee EBM | IsFullPartTime | FulltimeParttime | conditional | "full time" / "part time" | FULL\_TIME / PART\_TIME | low | — |
| MR-011 | INT new-hire record to Employee EBM | Compensation / allowances / supervisor | Currency, Salary, TaxableAllowance, NonTaxableAllowance, SupervisorIdentifier | passthrough | — | AllowanceType = TaxableAllowance, Allowance = NonTaxableAllowance | low | — |
| MR-012 | INT new-hire record to Employee EBM | Action | ActionType | conditional | lower-case value | hiring -\> HIRE; rehiring -\> REHIRE; internal hiring -\> INTERNALHIRE | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Action validity | Hiring requires an empty EmployeeID; Rehiring requires an EmployeeID. | High |
| BR-002 | Mandatory candidate and job | Candidate ID and JobCode must be present. | High |
| BR-003 | PH SSS uniqueness | Philippine hires need an SSS number not already on an HCM person. | High |
| BR-004 | Regional notification | US issues to UShrisJobvite, others to hris\_Jobvite. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Poll Interviewed folder** — Every 60 s, \*.csv
2. **Parse CSV** — Rejected -\> Rejection email
3. **Next hire** — Sequential
4. **Action, candidate, job valid?**
5. **Philippines?**
6. **SSS present and new?** — HCM emps by NationalId
7. **Provision in HCM** — hcm-employee-provisioning
8. **Notify HRIS of issue** — US / non-US lists
9. **Wait 3 min, DAT profile** — dat-profile-generation-rest
10. **More hires?**
11. **Done**

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| INT New Hire CSV Prof | flatfile | prof-int-newhire-csv |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| File adapter ReadINTFile + archive | conn-int-file-share + op-int-file-get |
| NXSD ReadFileService.xsd | prof-int-newhire-csv |
| Fault policy -\> WebServiceHandler | Try/Catch around parsing + util-send-notification |
| forEach validation and calls | proc-sub-int-validate-create |
| xform\_create\_emp\_Int\_to\_hcm | map-int-csv-to-ebm |
| wait PT3M + Rest\_DatProfile | Process Call dat-profile-generation-rest after confirmed creation |

**Components**

| Component | Role |
| --- | --- |
| conn-int-file-share | Drop location connection |
| op-int-file-get | Get and delete \*.csv |
| prof-int-newhire-csv | CSV profile |
| map-int-csv-to-ebm | Row to EBM map |
| proc-sub-int-validate-create | Shared validation and creation sub-process |
| proc-int-file-intake | Scheduled process |

## Client questions

- **Q-001** Which new-hire file layout is live: the older 113-column layout keyed on the SSS number, or the Jobvite layout keyed on the candidate ID?
  **If this is not answered:** The Employee\_JobVite layout, keyed on the candidate-ID prefix.
- **Q-002** Which system produces the new-hire files, and where should they be dropped for Boomi (a folder on the Boomi server or an SFTP location)?
  **If this is not answered:** An SFTP folder managed by TaskUs IT.
- **Q-003** Is the three-minute pause after each hire needed, or can the follow-up step run once HCM confirms the worker is created?
  **If this is not answered:** Replace the pause with a check that the worker exists.
- **Q-004** Who should be told when a whole file cannot be read?
  **If this is not answered:** Keep the "Rejection" list.
