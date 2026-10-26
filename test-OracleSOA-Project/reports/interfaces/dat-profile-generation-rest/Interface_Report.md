# ProcessDatProfileGeneration (REST JobVite\_Rest)

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

Post-load reconciliation and follow-up for recruited hires, exposed as REST service JobVite\_Rest (POST, Employees JSON). For each candidate it looks up the HCM person number; if found it updates the Jobvite candidate with hcm\_integration\_status=Success and the employee ID, otherwise it fetches the HDL load errors (via the candidateID-\>contentID cross reference) and updates Jobvite with status Error and the message. It writes and emails HCM and HRIS list reports, appends success/failure rows to the daily Jobvite consolidated file, and for the batch builds Salary.dat and TalentProfile.dat, starts the HCM Data Loader and calls the payroll allowance loader.

| Field | Value |
| --- | --- |
| Interface | ProcessDatProfileGeneration (REST JobVite\_Rest) |
| Type | Listener |
| Complexity | complex |
| Components to build | 15 |
| Mapping rules | 16 |
| Business rules | 5 |

## Interface Summary

After a hire has been sent into HCM, this service checks whether it worked and finishes the job. For each candidate it asks HCM for the new person number. If HCM has the person, it marks the candidate in Jobvite as successfully integrated with their employee number. If not, it looks up the HCM Data Loader errors for that candidate's load and writes the error back to Jobvite. Either way it emails HR lists of the hires (one detailed list, one HRIS summary) and adds a success or failure line to the day's consolidated file, which a nightly job sends out at 23:30.  For the hires that succeeded it then creates two more HCM data files (salary, using a salary-basis table by location, and talent profile with up to three previous employers) and starts the HCM load, and it sends the allowances to the payroll allowance service.  Several points need attention. The Jobvite API key and secret are written in the service address and must be replaced. The detailed HR list contains salary, visa and tax ID data and is emailed as an attachment. The country check sends lists to both the Philippine and US recipients whenever both country fields are filled, which is almost always. The payroll call identifies employees by SSS number while the real HCM person number is available. Error text is carried in an unrelated field ("ContractEndDate"). Four versions of this service exist; one of them also creates Google accounts, so TaskUs must confirm which is deployed.

**Complexity:** complex (score 80)

## Functional Analysis

1. **Receive candidates** — REST POST (JobVite\_Rest) -\> InputVariable; reply immediately.
2. **Look up person number** — Per candidate: ProcessGetPersonNumber, wait for callback.
3. **Success path** — Set EmployeeID, clear message, delete XREF row; Jobvite PUT Success.
4. **Failure path** — Look up contentID in XRef\_UCMID; get HDL errors; message into ContractEndDate; count failure; Jobvite PUT Error.
5. **Write and email lists** — HCMList\_\<ts\>.csv and HCMList\_hris\_\<ts\>.csv; FileNotification to list recipients by country.
6. **Salary and talent profile** — If any candidate succeeded: Salary.dat and TalentProfile.dat for candidates with EmployeeID.
7. **Load and allowances** — ProcessHDLProvABCSImpl and ProcessPBLProvABCSImpl (no wait).
8. **Consolidated file** — Append rows to JobviteConsolidate\_PH.csv or \_US.csv by country of the first candidate.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | REST / JSON | https://129.144.156.28:443/soa-infra/resources/default/ProcessDatProfileGeneration!2.0/JobVite\_Rest | none declared (no OWSM policy in the project) |
| outbound | REST over HTTPS | https://api.jobvite.com/api/v2/candidate | API key and secret in query string \[redacted\] |
| outbound | SOAP async (composites) | oramds:/apps/Provider/HCM/ProcessGetPersonNumber.wsdl, gethdlerrorsprovabcsimpl\_client\_ep.wsdl | none declared |
| outbound | SOAP async (composites, no wait) | oramds:/apps/Provider/HCM/processhdlprovabcsimplprocess\_client\_ep.wsdl, processpblprovabcsimpl\_client\_ep.wsdl | none declared |
| outbound | SOAP (composite) | oramds:/apps/Utilities/Notification/filenotification\_client\_ep.wsdl | none declared |
| outbound | File (SOA server) | Domain.dvm Email folder (/u01/data/domains/SOACS-Pr\_domain/osi\_file/EmailAttachment) and DatFiles folder (/u01/data/domains/SOACS-Pr\_domain/osi\_file/put) | n/a |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Candidate to Jobvite status update | candidate.application.eId | Employee/ID | transformed | — | substring-before(ID, "-") | low | — |
| MR-002 | Candidate to Jobvite status update | customField hcm\_integration\_status | (outcome) | conditional | person number found | "Success" / "Error" | low | — |
| MR-003 | Candidate to Jobvite status update | customField hcm\_integration\_details | ContractEndDate (error message) | conditional | Error | — | low | Error text carried in ContractEndDate. |
| MR-004 | Candidate to Jobvite status update | customField hcm\_employee\_id | EmployeeID | passthrough | — | — | low | — |
| MR-005 | Candidates to HCM and HRIS lists | HCMList row (all input fields + Message) | Employee/\* | looped | each Employee | — | low | — |
| MR-006 | Candidates to HCM and HRIS lists | HRIS row Date, InstanceID, User, EID, Department, ProjectID, JobTitle, LocationName, Error | now, FirstAppliedTime, FirstName+LastName, EmployeeID, OrganizationDepartment, Campaign, JobCode, Location, ContractEndDate | looped | each Employee | — | low | — |
| MR-007 | Candidates to HCM and HRIS lists | HRIS Message | EmployeeID | conditional | EmployeeID empty | "Failed" else "Success" | low | — |
| MR-008 | Candidates to Salary and TalentProfile DAT | Salary: SourceSystemId / AssignmentNumber | ID | calculated | EmployeeID != "" | "Worker\_Salary\_" + K / "ASSIGNMENT\_" + K (K = ID prefix) | low | — |
| MR-009 | Candidates to Salary and TalentProfile DAT | Salary: DateFrom / SalaryAmount | ExpectedStartDate / Salary | passthrough | — | — | low | — |
| MR-010 | Candidates to Salary and TalentProfile DAT | Salary: SalaryBasisName | Location | transformed | — | Salary\_Basis.dvm on substring-after(Location, " ") | medium | — |
| MR-011 | Candidates to Salary and TalentProfile DAT | Salary: MultipleComponents / SalaryApproved / ActionCode | (none) | hardcoded | — | "N" / "Y" / "HIRE" | low | — |
| MR-012 | Candidates to Salary and TalentProfile DAT | TalentProfile: ProfileCode, PersonNumber, codes | ID, EmployeeID | calculated | EmployeeID != "" | K + "\_PER\_PROFILE"; ProfileTypeCode PERSON; status A; usage P | low | — |
| MR-013 | Candidates to Salary and TalentProfile DAT | ProfileItem PREV\_EMPLOY (1..3) | PreviousEmploymentDetails\[1..3\] | conditional | start and end dates present | ItemText2401 employer, 2402 address, 2403 city, 2404 postcode, 2407 position, ItemText308 salary; DateFrom/DateTo | medium | — |
| MR-014 | Candidates to consolidated row | EmployeeId, CandidateId, FirstName, MiddleName, LastName | EmployeeID, ID, names | looped | each Employee | — | low | — |
| MR-015 | Candidates to consolidated row | Status / Message | EmployeeID, ContractEndDate | conditional | EmployeeID != "" | Success + "Employee Created Successfully" else Failure + error text | low | — |
| MR-016 | Candidates to PBL request | EmployeeID, NationalIDNumber/SSSNumber, CountryCode, EffectiveStartDate, AllowanceType, Allowance | EmployeeID, SSSNo, Country/CountryCode, ExpectedStartDate, TaxableAllowance, NonTaxableAllowance | looped | EmployeeID != "" | CountryCode PH/US from Country name | low | hcm-pbl-allowances uses SSSNumber as PersonNumber. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Success definition | A hire is successful when HCM returns a person number for the candidate. | High |
| BR-002 | Error message source | Failure message = first HDL MSG\_TEXT for the candidate's load, else a generic text. | Medium |
| BR-003 | Follow-up loads only for successes | Salary, talent profile and PBL only for candidates with an EmployeeID. | High |
| BR-004 | Country routing | PH lists to HCMListEmail/hris\_Jobvite, US lists to USHCMListEmail/UShrisJobvite (legacy sends to both when both fields are filled). | Medium |
| BR-005 | Salary basis by location | Salary basis from Salary\_Basis table keyed on the location name after the first space. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive candidates** — REST JobVite\_Rest
2. **Next candidate** — Sequential
3. **Look up person number** — hcm-get-person-number
4. **Found in HCM?**
5. **Notify error** — EmailDL notification
6. **Jobvite: Success** — PUT candidate
7. **Fetch HDL errors** — XREF + hcm-get-hdl-errors
8. **Jobvite: Error** — PUT candidate
9. **Write and email lists** — HCMList / HRIS by country
10. **More candidates?**
11. **Any success?**
12. **Salary, talent, HDL, PBL** — No wait
13. **Append consolidated rows** — PH or US file
14. **End**

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| Jobvite Employees Create Req JSON Prof | json | prof-jobvite-employees-json |
| Jobvite Candidate Update Req JSON Prof | json | prof-jobvite-candidate-json |
| HCM List Report CSV Prof | flatfile | prof-hcmlist-csv |
| HRIS List Report CSV Prof | flatfile | prof-hris-list-csv |
| HCM HDL Salary DAT Prof | flatfile | prof-hdl-salary-dat |
| HCM HDL TalentProfile DAT Prof | flatfile | prof-hdl-talentprofile-dat |
| Jobvite Consolidated Row CSV Prof | flatfile | prof-jobvite-consolidated-csv |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| REST JobVite\_Rest | Web Services Server listener (proc-dat-profile-generation-rest) |
| Loop body and batch tail | proc-sub-dat-profile-core |
| JobvitePutCandidate | conn-jobvite-api + op-jobvite-put-candidate |
| XRef\_UCMID lookup / markForDelete | Agreed persistent store (see hcm-dat-generation) |
| File adapters + FileNotification | In-memory CSV + util-file-notification |
| Consolidated file (append) | Staging store read by jobvite-consolidated-notification-sch |
| Salary/TalentProfile files + HDL call | map-int-to-salary-talent-dat + Process Call hcm-hdl-loader |
| PBL call | Process Call hcm-pbl-allowances |

**Components**

| Component | Role |
| --- | --- |
| prof-jobvite-employees-json | Request profile |
| conn-jobvite-api | Jobvite API connection |
| op-jobvite-put-candidate | PUT candidate |
| prof-jobvite-candidate-json | Candidate update profile |
| prof-hcmlist-csv | HCM list profile |
| prof-hris-list-csv | HRIS list profile |
| prof-hdl-salary-dat | Salary DAT profile |
| prof-hdl-talentprofile-dat | TalentProfile DAT profile |
| prof-jobvite-consolidated-csv | Consolidated row profile |
| map-int-to-jobvite-status | Jobvite status map |
| map-int-to-hcmlist | List maps |
| map-int-to-salary-talent-dat | DAT maps |
| map-int-to-consolidated | Consolidated map |
| proc-sub-dat-profile-core | Core sub-process |
| proc-dat-profile-generation-rest | REST listener |

## Client questions

- **Q-001** Which version of the new-hire follow-up service is running in production: the Jobvite version, or the version that also creates Google accounts and updates the work email?
  **If this is not answered:** The Employee\_JobVite version (no Google creation here).
- **Q-002** Where should the day's hire results be kept for the 23:30 summary job once the server files go away?
  **If this is not answered:** A small database table, written per hire and read and cleared at 23:30.
- **Q-003** Can TaskUs issue new Jobvite API credentials for Boomi and revoke the ones in the current integration?
  **If this is not answered:** New keys issued and stored in the Boomi Jobvite connection.
- **Q-004** Should the HR hire lists go only to the team for the hire's country?
  **If this is not answered:** Send only to the hire's country team.
- **Q-005** Does the detailed HR list need salary, visa and tax ID data, given that it is emailed to a distribution list?
  **If this is not answered:** Reduce the list to identification, job and status fields.
- **Q-006** Should allowances be loaded using the HCM person number rather than the SSS number?
  **If this is not answered:** Use the HCM person number.
