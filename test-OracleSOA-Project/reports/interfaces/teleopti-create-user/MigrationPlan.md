# Migration Plan — ProcessCreateTeleOptiUser

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.256Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessCreateTeleOptiUser |
| Identifier | teleopti-create-user |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | medium (34) |
| Components to build | 6 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/BPEL/ProcessCreateTeleOptiUser.bpel |
| Depends on | hcm-otbi-sql-query, util-send-notification, hcm-dat-generation |

## 2. End-to-End Process Architecture

Async BPEL composite calling ProcessHCMExtractSyncProv, the TeleOpti SDK OrganizationService (MDS WSDL) with a SOAP header built from DVMs, and XXSendNotification. Unused File/FTP adapters remain in the project.

If kept: Boomi process "TeleOpti User CREATE": Process Call hcm-otbi-sql-query, Decision on eligibility, fn-teleopti-auth-header builds the SOAP header from connection/extension values, map-ebm-to-teleopti-addperson, op-teleopti-add-person on conn-teleopti-sdk-soap; op-teleopti-get-persons available for duplicate checks; Try/Catch with util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback | Process start + Return Documents |  |
| TeleoptiAuthenticationHeader from DVMs | fn-teleopti-auth-header + connection/extension fields | Credentials out of data tables. |
| xform_EBO_to_AddTeleOptiUser | map-ebm-to-teleopti-addperson |  |
| AddPerson | op-teleopti-add-person on conn-teleopti-sdk-soap |  |
| TeleOpti_BU.dvm | Cross reference TeleOpti_BU | Fix key. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive employee | Async SOAP process with EmployeeEBM EmployeeRequest. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/composite.xml |
| 2 | Read HCM details | OTBI via ProcessHCMExtractSyncProv by Person Number = Employee/EmployeeID. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/BPEL/ProcessCreateTeleOptiUser.bpel |
| 3 | Eligible? | Job ASSOCIATE / SENIOR ASSOCIATE and Assignment Category "Eligible for OTND"; otherwise callback with no status. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/BPEL/ProcessCreateTeleOptiUser.bpel |
| 4 | Build auth header | TeleoptiAuthenticationHeader from TeleoptiAuthHeader.dvm and TeleOpti_BU.dvm. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/Schemas/TeleOptiHeaders.xsd |
| 5 | Enrich and map | Copy OTBI values into the EBM, xform_EBO_to_AddTeleOptiUser. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/Transformations/xform_EBO_to_AddTeleOptiUser.xsl |
| 6 | AddPerson | TeleoptiOrganizationService.AddPerson. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/TeleOpti/TeleoptiCccSdkService.wsdl |
| 7 | Notify and callback | EmailDL notification "User Created in TeleOpti"; status Success. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/BPEL/ProcessCreateTeleOptiUser.bpel |

## 4. Payload Profiles

> **Not available.** No profile components are owned by this interface; the personDto shape comes from the TeleOpti SDK WSDL imported into the connector operations.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | ApplicationLogOnName | Person/FirstName | passthrough |  |  | low | Not unique. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/Transformations/xform_EBO_to_AddTeleOptiUser.xsl | EBM + HCM row to AddPerson |
| MR-002 | Email | Person/EmailAddress | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/Transformations/xform_EBO_to_AddTeleOptiUser.xsl | EBM + HCM row to AddPerson |
| MR-003 | EmploymentNumber | EmployeeID | passthrough |  |  | low | Used by update lookup. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/Transformations/xform_EBO_to_AddTeleOptiUser.xsl | EBM + HCM row to AddPerson |
| MR-004 | TimeZoneId | (none) | hardcoded |  | "Singapore Standard Time" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/Transformations/xform_EBO_to_AddTeleOptiUser.xsl | EBM + HCM row to AddPerson |
| MR-005 | FirstName / LastName / MiddleName | Person/* | transformed |  | substring(...,1,25) | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/Transformations/xform_EBO_to_AddTeleOptiUser.xsl | EBM + HCM row to AddPerson |
| MR-006 | HireDate / DateOfBirth | EmploymentAssignments/OriginalHireDate, DateOfBirth | transformed |  | format MM/DD/YYYY | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/Transformations/xform_EBO_to_AddTeleOptiUser.xsl | EBM + HCM row to AddPerson |
| MR-007 | JobTitle / ManagementLevel | OTBI Job Name (s_3) | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/BPEL/ProcessCreateTeleOptiUser.bpel | EBM + HCM row to AddPerson |
| MR-008 | Department / Location / Campaign | OTBI s_2 / s_4 / s_5 (project ID) | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/BPEL/ProcessCreateTeleOptiUser.bpel | EBM + HCM row to AddPerson |
| MR-009 | ISEmployeeID / ManagerEmailAddress / OMFullname | OTBI s_7 / s_6 / s_1 | passthrough |  |  | low | Manager person number, email, name. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/BPEL/ProcessCreateTeleOptiUser.bpel | EBM + HCM row to AddPerson |
| MR-010 | ISFullname / SOMFullname | Person/LastName, FirstName | calculated |  | "Last,First" (max 100) / "First Last" | low | Employee's own name; intended meaning of IS/SOM fields not determinable. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/Transformations/xform_EBO_to_AddTeleOptiUser.xsl | EBM + HCM row to AddPerson |
| MR-011 | Header BusinessUnit | literal "$ProjectID_Desc" | transformed |  | TeleOpti_BU lookup (broken) | medium | Should use project description s_9. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/BPEL/ProcessCreateTeleOptiUser.bpel | EBM + HCM row to AddPerson |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | TeleOpti eligibility | Job Name ASSOCIATE or SENIOR ASSOCIATE and Assignment Category "Eligible for OTND". | High | Decision after hcm-otbi-sql-query. |
| BR-002 | Business unit from project | TeleOpti BU = TeleOpti_BU(project description) (intended). | High | Cross reference TeleOpti_BU keyed on project description. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | processcreateteleoptiuser_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/composite.xml |
| outbound SOAP 1.1 over HTTPS | https://taskus.teleopticloud.com/SDK/TeleoptiCccSdkService.svc | Custom SOAP header TeleoptiAuthenticationHeader (user name/password/data source) over HTTPS transport | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/TeleOpti/TeleoptiCccSdkService.wsdl |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/composite.xml |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/Transformations/xform_EBO_to_AddTeleOptiUser.xsl | EBM to AddPerson personDto. | map-ebm-to-teleopti-addperson; header via fn-teleopti-auth-header. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll: notify EmailDL notification, callback Failure@. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| TeleoptiAuthHeader.dvm | dvm | yes |
| TeleOpti_BU.dvm | dvm | yes |
| oramds:/apps/Provider/TeleOpti/TeleoptiCccSdkService.wsdl | mds | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| TeleOpti WFM cloud SDK | SOAP / HTTPS | https://taskus.teleopticloud.com/SDK/TeleoptiCccSdkService.svc | TeleOpti (Calabrio) / TaskUs tenant |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | TeleOpti SDK | https://taskus.teleopticloud.com/SDK/TeleoptiCccSdkService.svc | [redacted].wsdl |

## 14. Boomi Migration Readiness Summary

This service was built to add new call-centre agents to TeleOpti, the workforce scheduling system, when they join. It checks in HCM that the person is an Associate or Senior Associate eligible for overtime/night differential, then creates them in TeleOpti with their name, email, employee number, job, department, location, manager and campaign.

Today nothing calls it: the calls from the new-hire and assignment jobs are switched off. It also has a defect that would stop it working correctly if switched back on: the TeleOpti "business unit" is looked up using the literal text "$ProjectID_Desc" instead of the project value, so it is always sent empty. In addition the TeleOpti login name is set to the person's first name, which is not unique.

Before building this in Boomi TaskUs needs to confirm whether TeleOpti provisioning is still required and how business units should be assigned.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | TeleOpti SDK SOAP Conn | conn-teleopti-sdk-soap | connector-settings | — |
| 2 | TeleOpti Person By EmploymentNumber GET Op | op-teleopti-get-persons | connector-action | conn-teleopti-sdk-soap |
| 3 | TeleOpti Person CREATE Op | op-teleopti-add-person | connector-action | conn-teleopti-sdk-soap |
| 4 | TeleOpti SOAP Auth Header Script | fn-teleopti-auth-header | transform.function | — |
| 5 | Map \| Employee EBM to TeleOpti AddPerson | map-ebm-to-teleopti-addperson | transform.map | prof-employee-ebm-xml |
| 6 | TeleOpti User CREATE | proc-teleopti-create-user | process | op-teleopti-add-person, fn-teleopti-auth-header, map-ebm-to-teleopti-addperson, proc-sub-otbi-query, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/BPEL/ProcessCreateTeleOptiUser.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/Transformations/xform_EBO_to_AddTeleOptiUser.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/TeleOpti/TeleoptiCccSdkService.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/TeleoptiAuthHeader.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/TeleOpti_BU.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/composite.xml

## Open questions

### Still outstanding

REVIEW-NEEDED: Should new agents still be created automatically in TeleOpti? (No live caller invokes this service: both calls (from newhire-emp-sch and emp-assignment-sch) are disabled. Confirm whether TeleOpti provisioning is still required.)
  - If this is not answered: Out of scope; not built.

REVIEW-NEEDED: How should an agent's TeleOpti business unit be chosen? (The business-unit lookup passes a fixed text instead of the project, so the business unit is always empty; the correct BU mapping must be confirmed.)
  - If this is not answered: Look up the TeleOpti business unit by the HCM project description.

REVIEW-NEEDED: What should the TeleOpti login name be, given that first names are not unique? (Today the login name is the first name only.)
  - If this is not answered: Use the AD account name (initials + employee number).

REVIEW-NEEDED: Which TeleOpti service account should the new process use? (The current account details are stored in a data table in the integration platform and must move to the Boomi connection.)
  - If this is not answered: TaskUs creates a dedicated integration account in TeleOpti.

