# Migration Plan — ProcessEmployeeADPProvABCS

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.562Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessEmployeeADPProvABCS |
| Identifier | adp-employee-masterdata |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | complex (95) |
| Components to build | 6 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/BPEL/ProcessEmployeeADPProvABCS.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl |
| Depends on | hcm-otbi-sql-query, hcm-dat-generation, util-send-notification |

## 2. End-to-End Process Architecture

Async BPEL service: forEach employee with nested scopes calling the async OTBI provider, XSLT to NXSD, File adapter write, Java/shell PGP, skipped FTP put.

Boomi sub-process "ADP Employee Master Data File" (proc-adp-employee-masterdata) called by proc-adp-hcm-event-detector-sch: per-employee calls to the hcm-otbi-sql-query sub-process (consider fewer, joined queries), map-ebm-to-adp-md with Cross Reference Tables for the ADP DVMs, fn-adp-trailer-count, PGP Encrypt, op-adp-put on conn-adp-sftp; errors via util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| SOAP async service | Process Call target (sub-process) or Web Services Server listener | Caller is migrated too. |
| HCMExtract async provider | Process Call hcm-otbi-sql-query |  |
| xFORM_NEWHIRE_HCM_TO_FILE + 14 DVMs | map-ebm-to-adp-md + Cross Reference Tables | Fix or reproduce defects per payroll decision. |
| RECCT assign | fn-adp-trailer-count |  |
| FileEncryption shell | PGP Encrypt shape | Needs ADP public key. |
| eis/Ftp/ADP put (disabled) | op-adp-put on conn-adp-sftp |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive employee batch | Async SOAP process(EmployeeRequest). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/WSDLs/ProcessEmployeeADPProvABCS.wsdl |
| 2 | Per employee: read HCM data | Up to eleven OTBI queries by person number; status filter depends on terminate vs other actions. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/BPEL/ProcessEmployeeADPProvABCS.bpel |
| 3 | Derive codes | Employee group/subgroup (EmpGrpSGrp), location sub-area (LocationNameCode), rehire same-employer flag (CompCode). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/BPEL/ProcessEmployeeADPProvABCS.bpel |
| 4 | Build HRMD records | xFORM_NEWHIRE_HCM_TO_FILE by action. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl |
| 5 | Set trailer count | RECCT = number of records + 2. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/BPEL/ProcessEmployeeADPProvABCS.bpel |
| 6 | Write file | PP3208_<ts>_PHOHCM_HRMD01_DUT8G2I.SAP to the ADP folder. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Adapters/NewHireWrite_file.jca |
| 7 | PGP encrypt | FileEncryption runs a shell script producing .pgp. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/SCA-INF/lib/com/adp/FileEncryption.class |
| 8 | Upload to ADP (disabled) | Read .pgp and FTP put to /put — skipped. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Adapters/ADPFileWrite_ftp.jca |
| 9 | Callback | processResponse with the script output. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/BPEL/ProcessEmployeeADPProvABCS.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| ADP G2 HRMD Master Data Flat File Prof | flatfile | HEADER | record | no |
| ADP G2 HRMD Master Data Flat File Prof | flatfile | P0000, P0001, P0002, P0003, P0006, P0007, P0008, P0009, P0021, P0032, P0041, P0105, P0411, P0422, P0423, P0596, P0634, P0709 | record | no |
| ADP G2 HRMD Master Data Flat File Prof | flatfile | P_0000 … P_0634 | record | no |
| ADP G2 HRMD Master Data Flat File Prof | flatfile | TRAILER | record | no |
| ADP G2 HRMD Master Data Flat File Prof | flatfile | (all fields) | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | HEADER | (constants), current date/time | hardcoded | First employee | RECTY HEADR, USRID GVETASKUS, CLNAM TaskUs, CNAME DMEKA, FNAME MD, RUNDT, RUNTM, RUNTY P, VERNO 1 | low | Confirm with ADP. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl | Employee EBM to ADP HRMD records |
| MR-002 | BEGDA (B1 records) | EmploymentAssignments/EffectiveStartDate | transformed |  | yyyyMMdd; if earlier than 20170926 use 20170926 | low | Hardcoded floor. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl | Employee EBM to ADP HRMD records |
| MR-003 | P0000 MASSN | CustomAttributes/Description | conditional | Action | newhire 01, rehire 12, terminate 10 (BEGDA termination+1), companytransfer/costcenterchange 02 | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl | Employee EBM to ADP HRMD records |
| MR-004 | P0001 BUKRS, WERKS, PERSG, PERSK, BTRTL, KOSTL, ANSVH, ABKRS | Legal employer, location, employee group text, project id | transformed | newhire, rehire, costcenterchange | DVM LegalEntity, PersonnelArea, EmployeeGroup, PersonnelSubArea, CCenter, ProjectID; ABKRS 7M | high |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl | Employee EBM to ADP HRMD records |
| MR-005 | P0002 ANRED, GESCH, FAMST | /EmployeeRequest/Employee/... (absolute path) | transformed | newhire, rehire, namechange, taxdependentschange | DVM Title, Gender, MaritalStatus | medium | Defect: absolute path returns the first employee's value for every employee. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl | Employee EBM to ADP HRMD records |
| MR-006 | P0021 FGBDT, FASEX | ../../ContactDetails (employee level) | looped | Per dependant (Child / non-Child sets) | DVM FamilyType, Gender | medium | Defect: values are employee-level, not per dependant. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl | Employee EBM to ADP HRMD records |
| MR-007 | P0411 TAXFN | Person/NationalID_Type/TINNumber | conditional | TIN present | remove dashes; else select='000000000' | low | Defect: numeric literal yields 0. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl | Employee EBM to ADP HRMD records |
| MR-008 | P0411 NODEP / TXRDO | count(Person/Dependent); Organization name | calculated |  | count; DVM LegalEntity rdc | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl | Employee EBM to ADP HRMD records |
| MR-009 | P0008 wage lines | Person/Visa (wage type rows) | looped | newhire, rehire, basicpay | Wage codes 1000, 3001–3004 into LGA/BET pairs | medium | Wage data carried in the Visa structure. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl | Employee EBM to ADP HRMD records |
| MR-010 | P0634 | Person/PreEmployerDetail | looped | newhire, rehire, previousemployer |  | medium | Defect: for previousemployer the CLIID path ../../Employee/EmployeeID resolves to nothing, so CLIID is empty. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl | Employee EBM to ADP HRMD records |
| MR-011 | TRAILER RECCT | generated records | calculated | Last employee | XSL writes 15*employees+2; BPEL overwrites with actual record count + 2 | low | fn-adp-trailer-count. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/BPEL/ProcessEmployeeADPProvABCS.bpel | Employee EBM to ADP HRMD records |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Action drives record set | Each action produces a defined set of infotype records (see MigrationManifest 8.1 table). | High | Decision/route by action inside the map or before it. |
| BR-002 | Payroll-eligible status filter | Terminations read "Inactive - Payroll Eligible" assignments; all other actions "Active - Payroll Eligible". | High | Parameterised OTBI SQL. |
| BR-003 | Rehire to same employer | If the last two legal-employer history rows are the same company on rehire, P0709 is sent. | Medium | Script on CompCode rowset. |
| BR-004 | Code translation | TaskUs values are translated to ADP codes through 14 DVMs (e.g. LegalEntity 6 rows, CCenter 290 rows). | High | Cross Reference Tables. |
| BR-005 | Encrypt before send | File is PGP-encrypted for ADP. | High | PGP Encrypt shape. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP async | processemployeeadpprovabcs_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/composite.xml |
| outbound SOAP async (composite) | oramds:/apps/Provider/HCM/processhcmextractprovabcs_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/composite.xml |
| outbound File | /u01/app/oracle/tools/paas/state/homes/oracle/ADP_OSI_Taskus/PP3208_<ts>_PHOHCM_HRMD01_DUT8G2I.SAP | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Adapters/NewHireWrite_file.jca |
| outbound FTP (disabled) | eis/Ftp/ADP, /put | defined in the server connection factory (not in export) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Adapters/ADPFileWrite_ftp.jca |
| File NewHireWrite | [redacted], name set at runtime, Append false, NXSD write_new_hire.xsd. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Adapters/NewHireWrite_file.jca |
| File ADPFileRead (skipped) | Reads the .pgp file, DeleteFile true. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Adapters/ADPFileRead_file.jca |
| FTP ADPFileWrite (skipped) | eis/Ftp/ADP, /put, ascii. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Adapters/ADPFileWrite_ftp.jca |
| FTP ADPTransactionalWrite (skipped) | Transactional file put; unused here. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Adapters/ADPTransactionalWrite_ftp.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |
| hcm-dat-generation | Shared EmployeeEBM contract | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/SCA-INF/lib/com/adp/FileEncryption.class | com.adp.FileEncryption.readFile runs /bin/sh <script> with passphrase, GPG user, recipient, in/out paths. | PGP Encrypt shape with a PGP certificate component. | High |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl | EBM to ~35 G2 record types with DVM lookups. | map-ebm-to-adp-md with cross-reference lookups; complex conditions may need scripting. | High |
| other | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/BPEL/ProcessEmployeeADPProvABCS.bpel | RECCT = sum of record counts + 2. | fn-adp-trailer-count. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Per-employee catchAll with email; no process-level handler. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds EmployeeEBM.xsd / EmployeeEBO.xsd | mds | yes |
| ADP DVMs (14) and MaritalStatus.dvm | dvm | yes |
| Credentials.dvm (PGP keys) | dvm | yes |
| PGP shell script and ADP public key | other | no |
| eis/Ftp/ADP connection factory | other | no |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| ADP GlobalView GVS | FTP (legacy, disabled) | eis/Ftp/ADP connection factory, /put | ADP |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Output folder | [redacted] | [redacted].jca |

## 14. Boomi Migration Readiness Summary

This service builds the employee master-data file that ADP needs to run TaskUs's Philippines payroll. It is given a list of employees and what happened to each (hired, rehired, terminated, transferred, changed name and so on). For each employee it collects personal, job, pay, bank, tax and government-ID details from HCM and turns them into the record types ADP's G2 interface expects, translating TaskUs values into ADP codes through fourteen lookup tables (legal entity, personnel area, cost centre, employee group and others).

The finished file is saved on the integration server and encrypted with ADP's PGP key by a script. The upload to ADP is switched off in the code, so nothing in the export shows how ADP receives the file today.

Several defects in the mapping would put wrong data into payroll if copied as-is: with more than one employee in a file, title, gender and marital status come from the first employee; the "no TIN" value becomes 0 instead of nine zeros; dependants' birth date and gender are taken from the employee; and one query filter is missing for the first employee. Payroll must decide whether each is fixed, and the result should be validated against ADP's Philippines specification. In Boomi this becomes a sub-process with a large flat-file map, cross-reference tables, a PGP-encrypt step and an SFTP send.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | ADP G2 HRMD Master Data Flat File Prof | prof-adp-md-flatfile | profile.flatfile | — |
| 2 | ADP GVS File Transfer Conn | conn-adp-sftp | connector-settings | — |
| 3 | ADP Payroll File SEND Op | op-adp-put | connector-action | conn-adp-sftp |
| 4 | ADP File Trailer Record Count Script | fn-adp-trailer-count | transform.function | — |
| 5 | Map \| Employee EBM to ADP HRMD Records | map-ebm-to-adp-md | transform.map | prof-employee-ebm-xml, prof-adp-md-flatfile |
| 6 | ADP Employee Master Data File | proc-adp-employee-masterdata | process | op-adp-put, map-ebm-to-adp-md, fn-adp-trailer-count, proc-sub-otbi-query, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/BPEL/ProcessEmployeeADPProvABCS.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Transformations/xFORM_NEWHIRE_HCM_TO_FILE.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Schemas/write_new_hire.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/WSDLs/ProcessEmployeeADPProvABCS.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Adapters/NewHireWrite_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Adapters/ADPFileWrite_ftp.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Adapters/ADPFileRead_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/Adapters/ADPTransactionalWrite_ftp.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessEmployeeADPProvABCS/SOA/SCA-INF/lib/com/adp/FileEncryption.class
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Credentials.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/ADP/LegalEntity.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/ADP/CCenter.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/ADP/EmpGrpSGrp.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/MaritalStatus.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TimeCard/ProcessNewHireEmpBIReq_Sch/SOA/composite.xml

## Open questions

### Still outstanding

REVIEW-NEEDED: How does the employee master-data file reach ADP today, and how should Boomi deliver it? (The FTP upload to ADP is disabled in the source; the file is only written and PGP-encrypted on the SOA host. How the files reach ADP today is unknown (manual pickup? another job?). Before building delivery, confirm the delivery method, the ADP GVS endpoint and protocol, and the PGP public key; account details will be configured directly in Boomi.)
  - If this is not answered: Boomi PGP-encrypts the file and sends it by SFTP to the ADP GVS endpoint ADP specifies.

REVIEW-NEEDED: For each mapping defect found (first employee's title, gender and marital status reused; TIN default 0 instead of nine zeros; dependants' birth date and gender taken from the employee; empty previous-employer client ID; missing status filter on the first employee), should Boomi fix it or reproduce today's output? (Defects in xFORM_NEWHIRE_HCM_TO_FILE.xsl and the BPEL affect payroll data. Confirm with payroll whether each should be fixed or reproduced, and validate against the ADP Philippines specification.)
  - If this is not answered: Fix all of them and validate the file with ADP before go-live.

REVIEW-NEEDED: Are the ADP identifiers still current: client user GVETASKUS, company DMEKA, payroll area 7M, earliest start date 26 Sep 2017 and the file name PP3208_…_PHOHCM_HRMD01_DUT8G2I.SAP? (The BEGDA floor 20170926, ABKRS 7M, header USRID GVETASKUS, CNAME DMEKA and the file name are hardcoded. Confirm the current values with ADP.)
  - If this is not answered: Keep the current values as environment-configurable properties.

REVIEW-NEEDED: Who should be told when an employee cannot be included in the ADP file? (Today a generic notification list gets an email with the subject "Test" and the raw error.)
  - If this is not answered: Send a clear per-employee error summary to the payroll team list.

