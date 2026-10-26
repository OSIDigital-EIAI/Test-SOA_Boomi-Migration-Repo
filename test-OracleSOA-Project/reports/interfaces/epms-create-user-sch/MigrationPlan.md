# Migration Plan — ProcessCreateEPMSUser_sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.525Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessCreateEPMSUser_sch |
| Identifier | epms-create-user-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | medium (48) |
| Components to build | 5 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/BPEL/ProcessCreateEPMSUser_sch.bpel |
| Depends on | newhire-emp-sch, hcm-otbi-sql-query, util-send-notification |

## 2. End-to-End Process Architecture

Scheduler-started BPEL composite with XXTS window, async OTBI provider (sync variant skipped), XSLT to NXSD, File adapter, disabled Java lftp upload, XXSendNotification.

Boomi scheduled process "EPMS New Hire Profile Sch": window sub-process (newhire-emp-sch), hcm-otbi-sql-query for persons and per-person details (or one combined query), map-hcm-to-epms into prof-epms-profile, op-epms-put on conn-epms-ftp (SFTP preferred) if delivery is confirmed; notifications through util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| XXTS window | Sub: Schedule Window Get and Advance (newhire-emp-sch) |  |
| Async OTBI provider x4 | Process Call hcm-otbi-sql-query | Consider one joined query. |
| xform_HCM_EPMS | map-hcm-to-epms |  |
| File adapter + disabled Java lftp | conn-epms-ftp + op-epms-put | Delivery to confirm. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Scheduled start | Direct binding; read XXTS window (start, end). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |
| 2 | Find new persons | OTBI Person Real Time where Creation Date between window start and end. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/BPEL/ProcessCreateEPMSUser_sch.bpel |
| 3 | Per person: three queries | Assignment (64 columns), work relationship, person type by person number. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/BPEL/ProcessCreateEPMSUser_sch.bpel |
| 4 | Map to EPMS record | xform_HCM_EPMS; accumulate rows. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl |
| 5 | Warn on missing job data | Job Name or Job Manager Level empty -> EmailDL EPMSJOB. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/BPEL/ProcessCreateEPMSUser_sch.bpel |
| 6 | Write file | NewHireEPMS_<ts>.csv in the EPMS folder. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Adapters/EPMS_FileWrite_file.jca |
| 7 | FTP (disabled) | Java lftp put to /HCM/PROD skipped. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/SCA-INF/src/com/osi/FTPFileTransfer_Class.java |
| 8 | Advance window | Update start date and last run; callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/BPEL/ProcessCreateEPMSUser_sch.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| EPMS Employee Profile Flat File Prof | flatfile | Employee (68 fields) | record | yes |
| EPMS Employee Profile Flat File Prof | flatfile | FirstName\|MiddleName\|LastName\|PersonNumber\|PreviousLastName\|NameSuffix\|DisplayName\|WorkEmail\|PersonId\|EffectiveStartDate\|UserName | string | no |
| EPMS Employee Profile Flat File Prof | flatfile | PersonType*, Assignment*, BusinessUnit*, ProjectID, ProjectDescription, Location*, Job*, Grade*, Department* | string | no |
| EPMS Employee Profile Flat File Prof | flatfile | AssignmentCategory\|StartTime\|EndTime\|ActionCode\|ActionReasonCode\|AssignmentStatus\|Position*\|Manager*\|AssignmentNumber\|OriginalHireDate\|LegalEntityId\|CreationDate\|LastUpdateDate\|PeriodOfService* | string | no |
| EPMS Employee Profile Flat File Prof | flatfile | JobLevel\|JobCode\|JobManagementLevel*\|ADPCompanyCode\|FileNumber\|ADPPositionID | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | FirstName, MiddleName, LastName, PreviousLastName, NameSuffix, DisplayName, WorkEmail | Assignment Column29, 31, 30, 33, 32, 27, 28 | passthrough | Assignment status != INACTIVE |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI assignment to EPMS profile |
| MR-002 | PersonNumber, PersonId, UserName, AssignmentId, AssignmentNumber | Column38, 37, 40, 39/17, 25 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI assignment to EPMS profile |
| MR-003 | PersonTypeId / PersonTypeEffectiveDate / PersonTypeModifyDate | Person type query Column5, 1, 2 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI assignment to EPMS profile |
| MR-004 | BusinessUnit, Project, Location, Job, Grade, Department, Position fields with ids and dates | Assignment columns 1-15, 41-58 | passthrough |  |  | medium | GradeName and GradeCode both from Column4. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI assignment to EPMS profile |
| MR-005 | AssignmentCategory, StartTime, EndTime, ActionCode, ActionReasonCode, AssignmentStatus, assignment dates | Column24, 23, 22, 16, 21, 18, 20, 19 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI assignment to EPMS profile |
| MR-006 | ManagerId, ManagerPersonNumber, ManagerAssignmentId, OriginalHireDate, LegalEntityId, CreationDate, LastUpdateDate | Column36, 47, 35, 50, 8, 26, 34 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI assignment to EPMS profile |
| MR-007 | PeriodOfServiceId | Work relationship Column1 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI assignment to EPMS profile |
| MR-008 | JobLevel, JobManagementLevel ID/Name/dates | Column55, 61, 60, 54, 53 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI assignment to EPMS profile |
| MR-009 | ADPCompanyCode | Column62 (legal employer), Column11 (location), Column63 (country) | conditional | Country US | TaskUs USA LLC + Santa Monica -> MX4; TaskUs USA LLC -> 6PZ; TaskUs Inc. -> AK0 | medium | Same rule as hcm-external-identifier-sch. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI assignment to EPMS profile |
| MR-010 | FileNumber / PositionID | Column38 | calculated | Country US | substring(PersonNumber, 2) / ADP code + FileNumber | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI assignment to EPMS profile |
| MR-011 | CountryCode, CountryName, LegalEntityName, CreativeTitle | Column63, 64, 62, 7 | passthrough |  |  | low | Not in the PROD_MDS EPMSWrite.xsd; present in the DEV_MDS copy. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/DEV_MDS/apps/Utilities/XSD/EPMSWrite.xsd | OTBI assignment to EPMS profile |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | New persons in window | Persons whose Creation Date falls in the XXTS window. | High | Window sub-process + OTBI. |
| BR-002 | ADP codes for US | ADP company code, file number and position ID only for US workers by legal employer/location. | High | Cross reference (shared with hcm-external-identifier-sch). |
| BR-003 | Job data completeness | Warn when Job Name or Job Manager Level is empty. | Medium | Decision + util-send-notification to EPMSJOB list. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding / SOAP async | processcreateepmsuser_DirectBinding / _sch_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/composite.xml |
| outbound SOAP async (composite) | oramds:/apps/Provider/HCM/processhcmextractprovabcs_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/composite.xml |
| outbound File (SOA server) | /u01/data/domains/SOACS-Pr_domain/EPMSFolder/NewHireEPMS_<ts>.csv | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Domain.dvm |
| outbound FTP via shell lftp (disabled) | IP-addressed host [redacted], directory /HCM/PROD | user/password hardcoded in Java [redacted] | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/SCA-INF/src/com/osi/FTPFileTransfer_Class.java |
| File EPMS_FileWrite | Directory and file name set at runtime (Domain.dvm EPMS, NewHireEPMS_<ts>.csv); Append false; NXSD oramds:/apps/Utilities/XSD/EPMSWrite.xsd. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Adapters/EPMS_FileWrite_file.jca |
| DB adapters XXTS_SCHEDULING | Select window, advance start, update last run; jca.retry.count 4. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| newhire-emp-sch | Sub: Schedule Window Get and Advance | synchronous |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/SCA-INF/src/com/osi/FTPFileTransfer_Class.java | Run sh -c "lftp -e 'put -E <file>; exit' -u <user>,<password> <host>:<dir>" (deletes the local file after upload). Disabled in this composite. | conn-epms-ftp + op-epms-put (prefer SFTP). | High |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl | OTBI rows to EPMS profile with ADP code rules. | map-hcm-to-epms. | Medium |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Per-person catchAll with email; process catchAll with EmailDL notification. |
| Retries | DB adapter retries only. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Utilities/XSD/EPMSWrite.xsd (PROD_MDS) | mds | yes |
| Domain.dvm, EmailDL.dvm | dvm | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| EPMS FTP server | FTP (lftp) | IP-addressed host [redacted], /HCM/PROD | TaskUs EPMS |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | EPMS folder | [redacted] | [redacted].dvm |

## 14. Boomi Migration Readiness Summary

Twice a day this job collects the people newly created in HCM since the last run and writes a profile line for each into a file for EPMS. For each person it runs three HCM queries (assignment, work relationship, person type) and fills about 70 fields: names, person and assignment IDs, business unit, project, location, job, grade, department, manager, dates, job management level and, for US staff, the ADP company code (MX4, 6PZ or AK0), file number and position ID. If a person's job name or job level is empty it warns the EPMS job team.

The file is written to a folder on the integration server; the step that uploads it to EPMS by FTP is switched off. The export does not show whether something else picks the file up, so TaskUs must confirm how EPMS receives new hires today. The map also writes four fields (country code, country name, legal entity, creative title) that exist only in the development version of the file layout.

In Boomi this becomes a scheduled process using the shared window and OTBI sub-processes, a flat-file profile and an FTP/SFTP send if delivery is confirmed.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | EPMS Employee Profile Flat File Prof | prof-epms-profile | profile.flatfile | — |
| 2 | EPMS FTP Conn | conn-epms-ftp | connector-settings | — |
| 3 | EPMS Profile File SEND Op | op-epms-put | connector-action | conn-epms-ftp |
| 4 | Map \| OTBI Assignment to EPMS Profile | map-hcm-to-epms | transform.map | prof-otbi-rowset, prof-epms-profile |
| 5 | EPMS New Hire Profile Sch | proc-epms-create-user-sch | process | proc-sub-schedule-window, proc-sub-otbi-query, map-hcm-to-epms, op-epms-put, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/BPEL/ProcessCreateEPMSUser_sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/SCA-INF/src/com/osi/FTPFileTransfer_Class.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Adapters/EPMS_FileWrite_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/Transformations/xform_HCM_EPMS.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/EPMS/ProcessCreateEPMSUser_sch/SOA/WSDLs/EPMS_FileWrite.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/XSD/EPMSWrite.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/DEV_MDS/apps/Utilities/XSD/EPMSWrite.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Domain.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx

## Open questions

### Still outstanding

REVIEW-NEEDED: How does EPMS receive the new-hire file today, given that the upload step in the integration is switched off? (Its FTP push is disabled. Confirm the EPMS file layout (PROD vs DEV schema) and how the new-hire file is delivered to EPMS today.)
  - If this is not answered: Boomi uploads the file to the EPMS server by SFTP.

REVIEW-NEEDED: Should the EPMS profile file include country code, country name, legal entity and creative title? (The Create XSL emits CountryCode, CountryName, LegalEntityName and CreativeTitle, which exist only in the DEV_MDS EPMSWrite.xsd, and omits JobCode, which exists in PROD_MDS.)
  - If this is not answered: Use the PROD layout (with JobCode, without the four extra fields), as the Update job does.

REVIEW-NEEDED: Which EPMS server, account and protocol should Boomi use to deliver files? (The legacy account details are hardcoded and must be replaced; SFTP is preferred.)
  - If this is not answered: SFTP to the existing server with a new service account.

