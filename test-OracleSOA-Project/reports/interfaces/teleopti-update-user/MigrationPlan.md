# Migration Plan — ProcessUpdateTeleOptiUser

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.282Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessUpdateTeleOptiUser |
| Identifier | teleopti-update-user |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | medium (32) |
| Components to build | 3 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/BPEL/ProcessUpdateTeleOptUser.bpel |
| Depends on | teleopti-create-user, hcm-otbi-sql-query, util-send-notification, hcm-dat-generation |

## 2. End-to-End Process Architecture

Async BPEL composite: ProcessHCMExtractSyncProv, TeleOpti SDK via a wrapper WSDL (MDS import), XXSendNotification; update and cascade scopes disabled.

If confirmed: Boomi process "TeleOpti User UPDATE": hcm-otbi-sql-query, eligibility Decision, op-teleopti-get-persons, map-ebm-to-teleopti-updateperson, op-teleopti-update-person, optional cascade loop; Try/Catch with util-send-notification; single result returned.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback | Process start + Return Documents | One reply on every path. |
| GetPersonsByQuery | op-teleopti-get-persons (teleopti-create-user) |  |
| UpdatePerson (disabled) | op-teleopti-update-person |  |
| xform_EBO_to_UpdateTeleOptiUser | map-ebm-to-teleopti-updateperson | Align time zone. |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive employee | Async SOAP process with EmployeeEBM. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/composite.xml |
| 2 | Read HCM details | OTBI by Person Number. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/BPEL/ProcessUpdateTeleOptUser.bpel |
| 3 | Eligible? | Departments.dvm allow-list, ASSOCIATE / SENIOR ASSOCIATE, "Eligible for OT ND"; otherwise callback (twice). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Departments.dvm |
| 4 | Find TeleOpti person | GetPersonsByQuery by employment number with auth header. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/Transformations/xform_getPersonByQuery.xsl |
| 5 | Update person (disabled) | Copy OTBI values, xform_EBO_to_UpdateTeleOptiUser, set Id from the lookup, UpdatePerson. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/Transformations/xform_EBO_to_UpdateTeleOptiUser.xsl |
| 6 | Cascade to reports (disabled) | If ReqNumber = PersonUpdate: OTBI for workers whose Manager Person Number = this person, then look up and update each in TeleOpti. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/BPEL/ProcessUpdateTeleOptUser.bpel |
| 7 | Callback | processResponse (no status set). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/BPEL/ProcessUpdateTeleOptUser.bpel |

## 4. Payload Profiles

> **Not available.** No profile components are owned by this interface; personDto comes from the TeleOpti SDK WSDL.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | Id | GetPersonsByQuery PersonDto/Id | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/BPEL/ProcessUpdateTeleOptUser.bpel | EBM + HCM row to UpdatePerson |
| MR-002 | FirstName / LastName | OTBI s_9 / s_10 | transformed |  | substring(...,1,25) | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/BPEL/ProcessUpdateTeleOptUser.bpel | EBM + HCM row to UpdatePerson |
| MR-003 | JobTitle / Department / Location | OTBI s_3 / s_2 / s_4 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/BPEL/ProcessUpdateTeleOptUser.bpel | EBM + HCM row to UpdatePerson |
| MR-004 | ISEmployeeID / ManagerEmailAddress | OTBI s_7 / s_6 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/BPEL/ProcessUpdateTeleOptUser.bpel | EBM + HCM row to UpdatePerson |
| MR-005 | TimeZoneId | (none) | hardcoded |  | "W. Europe Standard Time" | low | Differs from create (Singapore). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/Transformations/xform_EBO_to_UpdateTeleOptiUser.xsl | EBM + HCM row to UpdatePerson |
| MR-006 | ApplicationLogOnName / EmploymentNumber / Email | Person/FirstName / EmployeeID / Person/EmailAddress | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/Transformations/xform_EBO_to_UpdateTeleOptiUser.xsl | EBM + HCM row to UpdatePerson |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Update eligibility | Department in Departments.dvm, Job ASSOCIATE / SENIOR ASSOCIATE, Assignment Category "Eligible for OT ND". | Medium | Decision; align wording with teleopti-create-user. |
| BR-002 | Cascade to reports | For PersonUpdate, also update every worker whose manager is this person (disabled). | Low | Only if confirmed: OTBI + loop. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | processupdateteleoptuser_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/composite.xml |
| outbound SOAP 1.1 over HTTPS | https://taskus.teleopticloud.com/SDK/TeleoptiCccSdkService.svc | Custom SOAP header TeleoptiAuthenticationHeader | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/WSDLs/TeleoptiCccSdkServiceWrapper.wsdl |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/composite.xml |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| teleopti-create-user | TeleOpti connection, get-persons operation and auth header | synchronous |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/Transformations/xform_EBO_to_UpdateTeleOptiUser.xsl | EBM to UpdatePerson personDto. | map-ebm-to-teleopti-updateperson. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/Transformations/xform_getPersonByQuery.xsl | GetPersonsByQuery by employment number. | op-teleopti-get-persons request. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll notify only. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| Departments.dvm | dvm | yes |
| TeleoptiAuthHeader.dvm / TeleOpti_BU.dvm | dvm | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| TeleOpti WFM cloud SDK | SOAP / HTTPS | https://taskus.teleopticloud.com/SDK/TeleoptiCccSdkService.svc | TeleOpti (Calabrio) / TaskUs tenant |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | TeleOpti SDK | https://taskus.teleopticloud.com/SDK/TeleoptiCccSdkService.svc | [redacted].wsdl |

## 14. Boomi Migration Readiness Summary

This service was meant to keep agents' details in TeleOpti in line with HCM: when an agent's name or job information changes, it finds them in TeleOpti by employee number and updates their record, and for a manager it was also meant to update everyone reporting to them.

In the exported code every update step is switched off. What still runs is the HCM lookup, an eligibility check (department in an allowed list, Associate or Senior Associate, assignment category "Eligible for OT ND") and the TeleOpti lookup. So TeleOpti is never updated. The code also has inconsistencies with the creation service: it checks for "Eligible for OT ND" (with a space) where creation checks "Eligible for OTND", and it would set the time zone to Western Europe where creation uses Singapore. On the not-eligible path it answers the caller twice, and on error it never answers.

TaskUs needs to decide whether TeleOpti updates, and the cascade to direct reports, are wanted at all before this is built.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | TeleOpti Person UPDATE Op | op-teleopti-update-person | connector-action | conn-teleopti-sdk-soap |
| 2 | Map \| Employee EBM to TeleOpti UpdatePerson | map-ebm-to-teleopti-updateperson | transform.map | prof-employee-ebm-xml |
| 3 | TeleOpti User UPDATE | proc-teleopti-update-user | process | op-teleopti-get-persons, op-teleopti-update-person, fn-teleopti-auth-header, map-ebm-to-teleopti-updateperson, proc-sub-otbi-query, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/BPEL/ProcessUpdateTeleOptUser.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/Transformations/xform_EBO_to_UpdateTeleOptiUser.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/Transformations/xform_getPersonByQuery.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessUpdateTeleOptiUser/SOA/WSDLs/TeleoptiCccSdkServiceWrapper.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/TeleOpti/TeleoptiCccSdkService.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/TeleoptiAuthHeader.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/TeleOpti_BU.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Departments.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/TeleOptiWorkforce/ProcessCreateTeleOptiUser/SOA/Transformations/xform_EBO_to_AddTeleOptiUser.xsl

## Open questions

### Still outstanding

REVIEW-NEEDED: Should employee changes in HCM update TeleOpti automatically, and should a manager's change also update everyone reporting to them? (All TeleOpti update calls are disabled in the source; confirm whether TeleOpti updates (and the manager cascade) should exist in the target.)
  - If this is not answered: Not built.

REVIEW-NEEDED: Which time zone should TeleOpti records use: Singapore (as on creation) or Western Europe (as in the update code)? (The two services set different time zones.)
  - If this is not answered: Singapore Standard Time, as on creation.

REVIEW-NEEDED: Is the eligibility category spelled "Eligible for OTND" or "Eligible for OT ND" in HCM? (Creation and update check different spellings, so one of them never matches.)
  - If this is not answered: Use the value found in HCM for both.

