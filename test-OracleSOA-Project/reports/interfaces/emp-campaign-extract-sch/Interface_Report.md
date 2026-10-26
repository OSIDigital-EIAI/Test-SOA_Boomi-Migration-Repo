# ProcessEmpCampaignExtract\_Sch

Test-OracleSOA-Project · Scheduled · simple complexity

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

Every day this job asks Oracle HCM Cloud to start the payroll flow named "Camapign" (sic) for the Philippines legislative data group, passing the effective date of the current schedule window and "changes only". The flow produces an extract of workers whose campaign (project) has changed, which a later job (emp-campaign-change-sch) reads to move those people in Active Directory.

| Field | Value |
| --- | --- |
| Interface | ProcessEmpCampaignExtract\_Sch |
| Type | Scheduled |
| Complexity | simple |
| Components to build | 6 |
| Mapping rules | 6 |
| Business rules | 2 |

## Interface Summary

This small job starts a payroll flow in Oracle HCM once a day. The flow, named "Camapign" in HCM (the spelling is in the source and must match HCM exactly), runs for the Philippines legislative data group with two parameters: the effective date (the start date of the job's time window) and "Changes only = Y". The flow instance is named "SOA\_" followed by the current date, which is how the follow-on campaign-change job finds it an hour later.  The job does not check whether HCM accepted the request, and if anything fails it sends no notification: it simply moves its time window on. In Boomi it becomes a scheduled process with one SOAP call to HCM, a simple map, and added error handling.  One point needs checking during the build: the HCM web service advertises WS-Security (SAML or username token) policies, while the legacy job authenticates with an HTTP Authorization header. The Boomi connection must use whichever method HCM accepts today.

**Complexity:** simple (score 18)

## Functional Analysis

1. **Start and read window** — Direct binding from the scheduler; Select\_XXTS\_Scheduling for INTERFACE\_NAME ProcessEmpCampaignExtract\_Sch; start\_date\_time formatted yyyy-MM-ddTHH:mm:ss; DateTime = date part; .000Z appended to start\_date\_time. (SOAP client entry uses the caller's DateTime.)
2. **Build submitFlow request** — xform\_flowActionService.xsl: flowName "Camapign", parameter "Effective Date" = DateTime, parameter "Changes only" = "Y", flowInstanceName = "SOA\_" + current-date(), legislativeDataGroupName "PH", recurringFlag false.
3. **Submit HCM flow** — FlowActionsService.submitFlow (SOAP 1.1, synchronous) with HTTP header Authorization from Credentials.dvm. The response is stored but not inspected.
4. **Advance window** — Direct binding: Update\_XXTS\_Schedule (last\_scheduled\_time = now) then Update\_Scheduling\_StartDateTime (start + INTERVAL + 1 s), then callback. SOAP client: callback only.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding / SOAP async | processempcampaignextract\_sch\_DirectBinding / processempcampaignextract\_sch\_client\_ep | none declared |
| outbound | SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmProcFlowCoreController/FlowActionsService | HTTP Authorization header (value from Credentials.dvm); service WSDL advertises wss11 SAML-or-username-token policies |
| outbound | Database | eis/DB/SOADemoLocalTx, table XXTS\_SCHEDULING | WebLogic data source |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | Schedule window to HCM submitFlow | submitFlow/flowName | (none) | hardcoded | — | "Camapign" | low | Spelling must match the HCM flow. |
| MR-002 | Schedule window to HCM submitFlow | parameterValues\[1\] ParameterName / ParameterValue | process/DateTime | transformed | — | name "Effective Date"; value = window start date (yyyy-MM-dd, from substring-before(start\_date\_time,"T")) | low | — |
| MR-003 | Schedule window to HCM submitFlow | parameterValues\[2\] ParameterName / ParameterValue | (none) | hardcoded | — | "Changes only" = "Y" | low | — |
| MR-004 | Schedule window to HCM submitFlow | submitFlow/flowInstanceName | (current date) | calculated | — | concat("SOA\_", xp20:current-date()) | low | xp20:current-date may carry a timezone suffix depending on server settings; the exact string must equal what emp-campaign-change-sch builds. Input element flowInstanceName is ignored. |
| MR-005 | Schedule window to HCM submitFlow | submitFlow/legislativeDataGroupName | (none) | hardcoded | — | "PH" | low | — |
| MR-006 | Schedule window to HCM submitFlow | submitFlow/recurringFlag | (none) | hardcoded | — | "false" | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Daily campaign extract submission | Submit flow "Camapign" for LDG PH once per run with Effective Date = window start date and Changes only = Y. | High |
| BR-002 | Instance naming contract | Flow instance is named "SOA\_" + current date so that emp-campaign-change-sch can find it later the same day. | High |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Scheduled start 14:00 UTC** — Direct binding from SchedulerServlet
2. **Read XXTS window** — Effective date = window start date
3. **Build submitFlow request** — Camapign, PH, Changes only = Y, SOA\_\<date\>
4. **Submit HCM flow** — FlowActionsService.submitFlow
5. **Fault: advance window** — catchAll; no notification
6. **Advance window, callback** — Last run and start date updated

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| HCM Flow SUBMIT Req XML Prof | xml | prof-hcm-flow-submit-req |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| SchedulerServlet + XXTS DB adapters | Process schedule + proc-sub-schedule-window |
| xform\_flowActionService.xsl | map-campaign-submitflow |
| FlowActionsService reference (MDS WSDL) + Authorization header | conn-hcm-flowactions-soap + op-hcm-flow-submit |
| catchAll without notification | Try/Catch + util-send-notification |

**Components**

| Component | Role |
| --- | --- |
| conn-hcm-flowactions-soap | SOAP connection to HCM FlowActionsService |
| op-hcm-flow-submit | submitFlow operation |
| op-hcm-flow-task-status | getFlowTaskInstanceStatus operation |
| prof-hcm-flow-submit-req | submitFlow request profile |
| map-campaign-submitflow | Window to submitFlow map |
| proc-emp-campaign-extract-sch | Main scheduled process |

## Client questions

- **Q-001** Is "Camapign" (spelt this way) still the exact name of the HCM payroll flow that produces the campaign-change extract, and is "PH" the right legislative data group?
  **If this is not answered:** Keep flow name "Camapign" and legislative data group "PH" as in the current code.
- **Q-002** If HCM rejects the daily campaign flow submission, who should be notified?
  **If this is not answered:** Email the integration "notification" distribution list.
- **Q-003** Which HCM integration account should the migrated job use to submit HCM flows?
  **If this is not answered:** Use the existing HCM integration account with a newly rotated password.
- **Q-004** Should the campaign extract keep running only once per day, and only for Philippine workers?
  **If this is not answered:** Once per day, Philippines only.
