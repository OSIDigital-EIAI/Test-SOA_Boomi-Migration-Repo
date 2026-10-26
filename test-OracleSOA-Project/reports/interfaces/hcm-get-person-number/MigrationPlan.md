# Migration Plan — ProcessGetPersonNumber

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.043Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessGetPersonNumber |
| Identifier | hcm-get-person-number |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | simple (14) |
| Components to build | 1 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/BPEL/ProcessGetPersonNumber.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessGetPersonNumber/SOA/BPEL/ProcessGetPersonNumber.bpel |
| Depends on | hcm-otbi-sql-query, util-send-notification |

## 2. End-to-End Process Architecture

Async BPEL composite calling ProcessHCMExtractSyncProv and XXSendNotification via MDS WSDLs. Two source copies exist (Employee and Employee_JobVite) for the same deployed composite name; the MDS client WSDL imports its schema from the running server, so which copy is deployed cannot be confirmed from the export, but the caller sends CandidateID, which only the Employee_JobVite copy understands.

Boomi sub-process "Sub: HCM Get Person Number": Decision on National_Id, build escaped SQL, Process Call hcm-otbi-sql-query, return Person_Number and a found/not-found flag; Try/Catch with util-send-notification.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback | Sub-process start + Return Documents |  |
| If1 + bpelx:append SQL | Decision + Set Properties (escaped values) |  |
| HCMExtractSyncProv | Process Call hcm-otbi-sql-query |  |
| catchAll | Try/Catch + util-send-notification |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive request | Async SOAP process (createInstance). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/Schemas/ProcessGetPersonNumber.xsd |
| 2 | Choose lookup key | If National_Id != "": national identifier query; else candidate ID query. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/BPEL/ProcessGetPersonNumber.bpel |
| 3 | Run OTBI query | ProcessHCMExtractSyncProv with the logical SQL; creation date LIKE current server date. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/BPEL/ProcessGetPersonNumber.bpel |
| 4 | Return person number | parseXML(rowset); Person_Number = Row[1]/Column1; callback. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/BPEL/ProcessGetPersonNumber.bpel |

## 4. Payload Profiles

> **Not available.** This interface owns only its process component; it uses the OTBI rowset profile of hcm-otbi-sql-query and a small inline request schema.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | sql | National_Id | conditional | National_Id != "" | Person Real Time query by National Identifier Number, Creation Date LIKE today% | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/BPEL/ProcessGetPersonNumber.bpel | Request to OTBI SQL and result |
| MR-002 | sql | CandidateID | conditional | National_Id empty | Worker Assignment Real Time query by PER_ASG_DF_CANDIDATE_ID_, Creation Date LIKE today% | medium | Employee_JobVite version only. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/BPEL/ProcessGetPersonNumber.bpel | Request to OTBI SQL and result |
| MR-003 | Person_Number | rowset Row[1]/Column1 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/BPEL/ProcessGetPersonNumber.bpel | Request to OTBI SQL and result |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Lookup key priority | National identifier first; candidate ID when no national identifier is given. | Medium | Decision on National_Id. |
| BR-002 | Created today only | Only persons whose Creation Date starts with the current server date. | Medium | Date filter parameter (time zone to confirm). |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessGetPersonNumber/processgetpersonnumber_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/ProcessGetPersonNumber.wsdl |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/composite.xml |
| outbound SOAP (composite) | oramds:/apps/Utilities/Notification/xxsendnotification_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/composite.xml |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| other | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/BPEL/ProcessGetPersonNumber.bpel | bpelx:append to build SQL, xp20:current-date(), oraext:parseXML. | Message/Set Properties for the SQL with escaped values and an explicit date; OTBI sub-process. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll: notify EmailDL notification, callback with empty result. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/HCM/ProcessGetPersonNumber.wsdl (schema imported from the server) | mds | yes |

## 12. External Services

> **Not available.** No external services were recorded for this interface.

## 13. Environment Configuration Matrix

> **Not available.** No environment-specific configuration was recorded.

## 14. Boomi Migration Readiness Summary

When a new hire from Jobvite has been loaded into HCM, the follow-up steps need the employee number HCM gave them. This service looks it up: if a national ID (such as the Philippine SSS number) is supplied it searches people by that ID; otherwise it searches by the Jobvite candidate ID stored on the assignment. In both cases it only considers people created in HCM today.

The "created today" rule uses the integration server's date, so a hire loaded late in the day in one time zone may not be found, and a hire whose load ran on a previous day will never be found. Inputs are put directly into the query text, and when nothing is found the service returns an empty number without saying why. The deployed contract and the caller send first name, last name and start date, but the service ignores them.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Sub: HCM Get Person Number | proc-sub-hcm-get-person-number | process | proc-sub-otbi-query, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/BPEL/ProcessGetPersonNumber.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessGetPersonNumber/SOA/Schemas/ProcessGetPersonNumber.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessGetPersonNumber/SOA/BPEL/ProcessGetPersonNumber.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessGetPersonNumber/SOA/Schemas/ProcessGetPersonNumber.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessGetPersonNumber/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/ProcessGetPersonNumber.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/BPEL/ProcessDatProfileGeneration.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee_JobVite/ProcessDatProfileGeneration/SOA/composite.xml

## Open questions

### Still outstanding

REVIEW-NEEDED: Should the person-number lookup only consider people created in HCM today, and if so in which time zone is "today"? (A hire loaded on a previous day, or across a date boundary, is never found.)
  - If this is not answered: Look back three days in UTC.

REVIEW-NEEDED: If several HCM people share the same national ID or candidate ID, which one should be used? (Today the first row returned is used with no ordering.)
  - If this is not answered: Use the most recently created person and report the duplicate.

