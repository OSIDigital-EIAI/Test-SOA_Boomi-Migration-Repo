# ProcessUpdateEPMSUser\_sch

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

Scheduled job that sends changed employees to EPMS. It reads the HCM Atom employee feeds (assignment changes and name changes) for the scheduler window, adds people from the day's completed campaign-change flow, builds a de-duplicated list of person numbers, re-queries each person from OTBI (assignment, work relationship, person type), maps them to the EPMS profile layout and writes UpdateEmployeeEPMS\_\<timestamp\>.csv, which it then uploads to the EPMS FTP server with a Java shell call (active in this composite).

| Field | Value |
| --- | --- |
| Interface | ProcessUpdateEPMSUser\_sch |
| Type | Scheduled |
| Complexity | complex |
| Components to build | 3 |
| Mapping rules | 9 |
| Business rules | 5 |

## Interface Summary

This job keeps EPMS up to date with employee changes made in HCM. On each run it asks HCM which employees changed since the last run (status, department, location, manager, job, name, rehire or global transfer) and, once the day's campaign-change process has completed in HCM, also picks up everyone whose campaign changed. Each person appears once in the list. For each one it re-reads their current details from HCM and fills the same EPMS profile record as the new-hire job, including the US ADP company code, file number and position ID.  The resulting file is uploaded to the EPMS server by FTP using a shell command with a hardcoded account; if the upload fails two named individuals are emailed. Errors for an individual person are silently skipped, so a person can be missed without anyone being told.  Two things need confirming before the Boomi build: how Boomi should reach EPMS (the current server is addressed by IP with plain FTP; SFTP is preferred) and how often the job really runs (every 30 minutes per the scheduling table, or twice a day per the spreadsheet). In Boomi this becomes a scheduled process that reads the HCM feeds, builds the change list in a script or map step, reuses the EPMS profile and map from the Create interface and sends the file over SFTP.

**Complexity:** complex (score 72)

## Functional Analysis

1. **Scheduled start and window** — Direct binding; read start/end from XXTS\_SCHEDULING; format as ISO with .000Z.
2. **Read assignment feed** — GET empassignment?updated-min&updated-max (Atom).
3. **Read employee update feed** — GET empupdate with the same window via HTTP properties.
4. **Campaign changes (first runs of the day)** — If window start date = today: flow task status for SOA\_\<date\>/Camapign/PH; if COMPLETED fetch extract output.
5. **Build change list** — xform\_EMPList conditions then remove duplicates (keep last).
6. **Per person: query HCM** — Assignment (63 columns), work relationship, person type by person number.
7. **Map record** — Global Transfer -\> GT map, else standard map; Active rows only.
8. **Write and upload file** — UpdateEmployeeEPMS\_\<ts\>.csv to EPMS folder, then lftp upload to /HCM/PROD; email on failure.
9. **Advance window** — Update start date and last run; callback.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding / SOAP async | processupdateepmsuser\_sch\_DirectBinding / \_sch\_client\_ep | none declared |
| outbound | REST GET (Atom) | https://eetm.hcm.us6.oraclecloud.com/hcmCoreApi/atomservlet/employee/empassignment?updated-min={}&updated-max={} | Authorization header from Credentials.dvm \[redacted\] |
| outbound | REST GET (Atom) | https://eetm.hcm.us6.oraclecloud.com/hcmCoreApi/atomservlet/employee/empupdate?updated-min={}&updated-max={} | Authorization header from Credentials.dvm \[redacted\] |
| outbound | SOAP | oramds:/apps/Provider/HCM/FlowActionsService\_wsdl.wsdl | Authorization header from Credentials.dvm \[redacted\] |
| outbound | SOAP | oramds:/apps/Provider/HCM/PayrollProcessingActionService.wsdl | Authorization header from Credentials.dvm \[redacted\] |
| outbound | SOAP async (composite) | oramds:/apps/Provider/HCM/processhcmextractprovabcs\_client\_ep.wsdl | none declared |
| outbound | File (SOA server) | /u01/data/domains/SOACS-Pr\_domain/EPMSFolder/UpdateEmployeeEPMS\_\<ts\>.csv | n/a |
| outbound | FTP via shell lftp | IP-addressed host \[redacted\], directory /HCM/PROD | user/password hardcoded in Java \[redacted\] |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Atom feeds and campaign result to change list | List/PersonNumber, content="Employee Assignment Feed" | empassignment entry content PersonNumber | conditional | "Person Assignment Updated" with AssignmentStatusType new INACTIVE or ACTIVE and ActionCode new != TERMINATION; Updated/Corrected with OrganizationId change; Updated/Corrected with ActionCode LOCATION\_CHANGE, MANAGER\_CHANGE or JOB\_CHANGE; "Person Assignment Created" with ActionCode REHIRE; "Person Manager Updated" with ManagerId change | substring parsing of the JSON text in Atom content | high | — |
| MR-002 | Atom feeds and campaign result to change list | List/PersonNumber, content="Employee Update Feed" | empupdate entry content PersonNumber | conditional | "Person Name Corrected" or "Person Name Updated" | — | medium | — |
| MR-003 | Atom feeds and campaign result to change list | List/PersonNumber, content="Campaign Change" | New\_Campaign/Assignment\_Info/Worker\_Information/Person\_Number | passthrough | Campaign flow COMPLETED | — | low | — |
| MR-004 | Atom feeds and campaign result to change list | List/PersonNumber, content="Global Transfer" | empassignment entry | conditional | "Person Assignment Created" with ActionCode GLB\_TRANSFER | — | low | — |
| MR-005 | Atom feeds and campaign result to change list | List (unique) | List | calculated | — | Drop an entry if a later entry has the same PersonNumber | low | Global Transfer entries are emitted last, so they win. |
| MR-006 | OTBI rows to EPMS profile (update) | All EPMS profile fields except the four DEV-only fields | Assignment, work relationship, person type rowsets | passthrough | Column18 Assignment Status Type = Active | Same column mapping as epms-create-user-sch MAP-001 | medium | Referenced, not re-derived. |
| MR-007 | OTBI rows to EPMS profile (update) | JobCode | Assignment Column59 | passthrough | — | — | low | Update only. |
| MR-008 | OTBI rows to EPMS profile (update) | ADPCompanyCode / FileNumber / ADPPositionID | Column62, Column11, Column63, Column38 | conditional | Column63 = US | MX4 / 6PZ / AK0 rule; substring(PersonNumber,2); code + FileNumber | medium | Same rule as epms-create-user-sch. |
| MR-009 | OTBI rows to EPMS profile (update) | (Global Transfer variant) | same | passthrough | List content = Global Transfer | xform\_HCM\_EPMS\_GT; identical field mapping, Employees root always emitted | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Qualifying changes | Only the listed assignment, manager, name, rehire, global-transfer and campaign events send a person to EPMS. | High |
| BR-002 | One record per person | A person appears once per file, current data re-read from HCM. | High |
| BR-003 | Campaign changes after flow completion | Campaign changes are included only when the SOA\_\<date\> campaign flow is COMPLETED and the window starts today. | Medium |
| BR-004 | Active only | Persons whose assignment is not Active produce no record. | Medium |
| BR-005 | ADP codes for US | As epms-create-user-sch BR-002. | High |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Scheduled start, read window**
2. **Read assignment and update feeds** — HCM Atom
3. **Window starts today?**
4. **Campaign flow completed?** — FlowActionsService
5. **Fetch campaign output** — PayrollProcessingActionService
6. **Build unique change list**
7. **Any persons?**
8. **Query person details** — Three OTBI queries
9. **Global transfer?**
10. **Map (GT map)**
11. **Map (standard)**
12. **More persons?**
13. **Write file and FTP upload**
14. **Upload failed?**
15. **Email failure**
16. **Advance window**

## Data Model

_No profile components are owned by this interface; it uses prof-epms-profile from epms-create-user-sch._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| XXTS window | Sub: Schedule Window Get and Advance (newhire-emp-sch) |
| REST Atom references | HTTP Client connector to HCM atomservlet |
| FlowActions / PayrollProcessing services | As in emp-campaign-change-sch |
| xform\_EMPList + remove\_duplicates | fn-epms-build-change-list |
| xform\_HCM\_EPMS(\_GT) | map-hcm-to-epms-gt / map-hcm-to-epms |
| File adapter + Java lftp | op-epms-put on conn-epms-ftp |

**Components**

| Component | Role |
| --- | --- |
| fn-epms-build-change-list | Change detection and dedup |
| map-hcm-to-epms-gt | Global Transfer map |
| proc-epms-update-user-sch | Scheduled process |

## Client questions

- **Q-001** Which EPMS server and transfer protocol should Boomi use to deliver the employee update file?
  **If this is not answered:** SFTP to the existing EPMS server, directory /HCM/PROD.
- **Q-002** How often should the EPMS update job run?
  **If this is not answered:** Every 30 minutes, 14:00–23:50, as in the scheduling table.
- **Q-003** Should employees from the daily campaign-change process be sent to EPMS only once per day?
  **If this is not answered:** Send campaign changes once per day, on the first run after the campaign flow completes.
- **Q-004** Who should be told when an individual employee cannot be sent to EPMS or the upload fails?
  **If this is not answered:** Report both to the EmailDL EPMSJOB list.
