# ProcessEmpCampaignChange\_Sch

Test-OracleSOA-Project · Scheduled · medium complexity

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

Every day, once the HCM "Camapign" payroll extract submitted earlier that day has completed, this job reads the list of Philippine workers whose campaign (project) changed and moves each one to the Active Directory location and groups of their new campaign. It then emails a CSV report of successes and failures to the IT service desk.

| Field | Value |
| --- | --- |
| Interface | ProcessEmpCampaignChange\_Sch |
| Type | Scheduled |
| Complexity | medium |
| Components to build | 7 |
| Mapping rules | 16 |
| Business rules | 4 |

## Interface Summary

This job applies campaign moves to Active Directory. Each day it asks HCM whether today's campaign extract flow (started an hour earlier by the companion job) has completed. If it has, it fetches the list of workers whose campaign changed and, one by one, asks the AD service to move each person to the new campaign's organisational unit and groups. It waits for each answer and records it in a CSV report, which is then emailed to the IT service desk.  The job only looks at "today's" flow, identified by the name SOA\_ plus the date. If the flow has not finished by the time the job runs, that day's changes are not picked up later. If the job ran twice, the same workers would be processed again. Errors for an individual worker are emailed to the service desk and the job carries on; any other error is emailed and the time window still moves on.  In Boomi this becomes a scheduled process with two HCM SOAP calls, a loop that calls the AD sub-process synchronously for each worker, and a report built in memory and emailed through the shared file-notification sub-process.

**Complexity:** medium (score 50)

## Functional Analysis

1. **Start and read window** — Direct binding from the scheduler; Select\_XXTS\_Scheduling for ProcessEmpCampaignChange\_Sch; start\_date\_time formatted and .000Z appended (used only for the window update). The SOAP client path skips this.
2. **Check today's flow status** — FlowActionsService.getFlowTaskInstanceStatus with flowInstanceName "SOA\_"+current-date(), flowTaskInstanceName "Camapign", legislativeDataGroupName "PH"; HTTP Authorization header from Credentials.dvm.
3. **Completed?** — Continue only if result = "COMPLETED"; otherwise go to the window update.
4. **Fetch campaign extract** — PayrollProcessingActionService.fetchExtractOutput (instanceName SOA\_\<date\>, taskName "Camapign", mode "FLOW"); result string parsed with oraext:parseXML into New\_Campaign.
5. **Any workers?** — If count(Assignment\_Info) \> 0: FileName = CampaignChange\_\<yyyy-MM-ddTHH:mm:ss\>.csv and header row written to AttachmentPath.
6. **For each worker (sequential)** — forEach parallel="no"; per-worker catchAll emails EmailDL itservicedesk with the fault and continues.
7. **Look up job name** — OTBI: SELECT Employee Mailing County Code s\_1, Job Name s\_2 FROM Worker Assignment Real Time WHERE Person Number = '\<Person\_Number\>' (appended correctly with bpelx:append).
8. **Move AD user to new campaign** — ProcessUpdateOUADUser: Action "Change", CN = initials + Person\_Number, Employee\_ID = Person\_Number, Status = Project\_ID; then waits for the processResponse callback.
9. **Write report row** — Status "Success" if result contains "Success", else "Failure"; Message = text after "@" in result; Project Description from the AD response.
10. **Email report** — FileNotification to EmailDL itservicedesk, subject "Consolidated Notification for Campaign Change", file from Domain.dvm Email path.
11. **Advance window** — Direct path: Update\_Scheduling\_StartDateTime and Update\_XXTS\_Schedule, then callback. Client path: callback with result = first OBJECT\_ACTION\_ID.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding / SOAP async | processempcampaignchange\_sch\_DirectBinding / processempcampaignchange\_sch\_client\_ep | none declared |
| outbound | SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmProcFlowCoreController/FlowActionsService | HTTP Authorization header (Credentials.dvm) |
| outbound | SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmBatchProcessCore/PayrollProcessingActionService | HTTP Authorization header (Credentials.dvm) |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared |
| outbound | SOAP async (composite) | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateOUADUser/processupdateouaduser\_client\_ep | none declared |
| outbound | File (SOA server) | AttachmentPath /u01/data/domains/SOACS-Pr\_domain/osi\_file/EmailAttachment (also set from Domain.dvm Email path) | n/a |
| outbound | SOAP (composite) | oramds:/apps/Utilities/Notification/filenotification\_client\_ep.wsdl / xxsendnotification\_client\_ep.wsdl | none declared |
| outbound | Database | eis/DB/SOADemoLocalTx, XXTS\_SCHEDULING | WebLogic data source |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Campaign worker to AD OU change request | Action | (none) | hardcoded | — | "Change" | low | — |
| MR-002 | Campaign worker to AD OU change request | CN | Worker\_Information/First\_Name, Last\_Name, Person\_Number | calculated | — | lower(first char First\_Name)+lower(first char Last\_Name)+Person\_Number | medium | substring-before(x,substring(x,2)) idiom; use first character directly in Boomi. |
| MR-003 | Campaign worker to AD OU change request | Employee\_ID | Worker\_Information/Person\_Number | passthrough | — | — | low | — |
| MR-004 | Campaign worker to AD OU change request | Status | Worker\_Information/Project\_ID | passthrough | — | — | low | For Action=Change the AD service treats Status as the new project ID. |
| MR-005 | AD result to report row | Sl\_No | loop counter | calculated | — | — | low | — |
| MR-006 | AD result to report row | Instance\_ID | flow id | calculated | — | execution id in Boomi | low | — |
| MR-007 | AD result to report row | Employee\_ID | Person\_Number | passthrough | — | — | low | — |
| MR-008 | AD result to report row | Employee\_Name | First\_Name, Last\_Name | calculated | — | First\_Name + " " + Last\_Name | low | — |
| MR-009 | AD result to report row | JobTitle | OTBI Row\[1\]/Column2 (Job Name) | passthrough | — | — | low | — |
| MR-010 | AD result to report row | New\_Campaign | Project\_ID | passthrough | — | — | low | — |
| MR-011 | AD result to report row | Project\_Description | AD response Description | passthrough | — | — | low | — |
| MR-012 | AD result to report row | Status | AD response result | conditional | contains(result,"Success") | "Success" / "Failure" | low | — |
| MR-013 | AD result to report row | Error (header "Message") | AD response result | transformed | — | substring-after(result,"@") | low | — |
| MR-014 | Flow status and extract requests | flowInstanceName / instanceName | (current date) | calculated | — | concat("SOA\_", xp20:current-date()) | low | Must equal the name built by emp-campaign-extract-sch. |
| MR-015 | Flow status and extract requests | flowTaskInstanceName / taskName | (none) | hardcoded | — | "Camapign" | low | — |
| MR-016 | Flow status and extract requests | legislativeDataGroupName / mode | (none) | hardcoded | — | "PH" / "FLOW" | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Process only completed extracts | Read the extract only when today's flow instance status is COMPLETED. | High |
| BR-002 | Campaign move in AD | Each worker in the extract is moved with Action "Change" and Status = new Project ID. | High |
| BR-003 | Success detection | An AD result containing "Success" is reported as Success; otherwise Failure, with the text after "@" as message. | Medium |
| BR-004 | Report distribution | Email the CSV report to EmailDL itservicedesk when at least one worker was in the extract. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Scheduled start 15:00 UTC** — Direct binding
2. **Check today's flow status** — getFlowTaskInstanceStatus SOA\_\<date\>
3. **Flow completed?** — result = COMPLETED
4. **Notify error, advance window** — process catchAll
5. **Fetch campaign extract** — fetchExtractOutput mode FLOW
6. **Any workers?** — count(Assignment\_Info) \> 0
7. **Look up job name** — OTBI per worker
8. **Move AD user (Change)** — ProcessUpdateOUADUser, wait for callback
9. **Notify worker error** — per-worker catchAll, continue
10. **Write report row** — Success / Failure
11. **More workers?** — forEach parallel=no
12. **Email report to service desk** — FileNotification itservicedesk
13. **Advance window, callback**

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| HCM Campaign Extract Result XML Prof | xml | prof-campaign-extract-result |
| Campaign Change Report CSV Prof | flatfile | prof-campaign-report-csv |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| FlowActionsService.getFlowTaskInstanceStatus | op-hcm-flow-task-status (emp-campaign-extract-sch) |
| PayrollProcessingActionService.fetchExtractOutput + oraext:parseXML | op-hcm-fetch-extract + unescape step + prof-campaign-extract-result |
| ProcessUpdateOUADUser invoke + receive | map-campaign-worker-to-ouad + synchronous Process Call ad-update-ou |
| File adapter append + FileNotification | map-campaign-result-to-report + Combine + util-file-notification |
| Per-worker catchAll | Try/Catch per document |

**Components**

| Component | Role |
| --- | --- |
| conn-hcm-payroll-soap | SOAP connection to PayrollProcessingActionService |
| op-hcm-fetch-extract | fetchExtractOutput operation |
| prof-campaign-extract-result | Extract result profile |
| prof-campaign-report-csv | Report CSV profile |
| map-campaign-worker-to-ouad | AD change request map |
| map-campaign-result-to-report | Report row map |
| proc-emp-campaign-change-sch | Main scheduled process |

## Client questions

- **Q-001** If the daily campaign extract in HCM has not finished when this job runs, should the job try again later the same day or pick it up the next day?
  **If this is not answered:** Retry hourly until the end of the day, then report the missed extract to the service desk.
- **Q-002** Should a worker who appears in the extract twice in one day (for example if the job is rerun) be moved again?
  **If this is not answered:** Process each extract only once per day.
- **Q-003** Who should receive the daily campaign change report and the error notices?
  **If this is not answered:** Keep sending both to the current itservicedesk list.
- **Q-004** Roughly how many campaign changes happen on a typical day and on the busiest day?
  **If this is not answered:** Assume up to 500 per day.
