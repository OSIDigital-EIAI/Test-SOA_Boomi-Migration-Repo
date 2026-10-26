# ProcessCreateTeleOptiUser

Test-OracleSOA-Project · Listener · medium complexity

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

Creates a person in the TeleOpti workforce-management cloud (SDK SOAP OrganizationService.AddPerson) for a new agent. It reads the worker's manager, department, job, location, project and assignment category from HCM and creates the person only when the job is ASSOCIATE or SENIOR ASSOCIATE and the assignment category is "Eligible for OTND".

| Field | Value |
| --- | --- |
| Interface | ProcessCreateTeleOptiUser |
| Type | Listener |
| Complexity | medium |
| Components to build | 6 |
| Mapping rules | 11 |
| Business rules | 2 |

## Interface Summary

This service was built to add new call-centre agents to TeleOpti, the workforce scheduling system, when they join. It checks in HCM that the person is an Associate or Senior Associate eligible for overtime/night differential, then creates them in TeleOpti with their name, email, employee number, job, department, location, manager and campaign.  Today nothing calls it: the calls from the new-hire and assignment jobs are switched off. It also has a defect that would stop it working correctly if switched back on: the TeleOpti "business unit" is looked up using the literal text "$ProjectID\_Desc" instead of the project value, so it is always sent empty. In addition the TeleOpti login name is set to the person's first name, which is not unique.  Before building this in Boomi TaskUs needs to confirm whether TeleOpti provisioning is still required and how business units should be assigned.

**Complexity:** medium (score 34)

## Functional Analysis

1. **Receive employee** — Async SOAP process with EmployeeEBM EmployeeRequest.
2. **Read HCM details** — OTBI via ProcessHCMExtractSyncProv by Person Number = Employee/EmployeeID.
3. **Eligible?** — Job ASSOCIATE / SENIOR ASSOCIATE and Assignment Category "Eligible for OTND"; otherwise callback with no status.
4. **Build auth header** — TeleoptiAuthenticationHeader from TeleoptiAuthHeader.dvm and TeleOpti\_BU.dvm.
5. **Enrich and map** — Copy OTBI values into the EBM, xform\_EBO\_to\_AddTeleOptiUser.
6. **AddPerson** — TeleoptiOrganizationService.AddPerson.
7. **Notify and callback** — EmailDL notification "User Created in TeleOpti"; status Success.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | processcreateteleoptiuser\_client\_ep | none declared |
| outbound | SOAP 1.1 over HTTPS | https://taskus.teleopticloud.com/SDK/TeleoptiCccSdkService.svc | Custom SOAP header TeleoptiAuthenticationHeader (user name/password/data source) over HTTPS transport |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | EBM + HCM row to AddPerson | ApplicationLogOnName | Person/FirstName | passthrough | — | — | low | Not unique. |
| MR-002 | EBM + HCM row to AddPerson | Email | Person/EmailAddress | passthrough | — | — | low | — |
| MR-003 | EBM + HCM row to AddPerson | EmploymentNumber | EmployeeID | passthrough | — | — | low | Used by update lookup. |
| MR-004 | EBM + HCM row to AddPerson | TimeZoneId | (none) | hardcoded | — | "Singapore Standard Time" | low | — |
| MR-005 | EBM + HCM row to AddPerson | FirstName / LastName / MiddleName | Person/\* | transformed | — | substring(...,1,25) | low | — |
| MR-006 | EBM + HCM row to AddPerson | HireDate / DateOfBirth | EmploymentAssignments/OriginalHireDate, DateOfBirth | transformed | — | format MM/DD/YYYY | low | — |
| MR-007 | EBM + HCM row to AddPerson | JobTitle / ManagementLevel | OTBI Job Name (s\_3) | passthrough | — | — | low | — |
| MR-008 | EBM + HCM row to AddPerson | Department / Location / Campaign | OTBI s\_2 / s\_4 / s\_5 (project ID) | passthrough | — | — | low | — |
| MR-009 | EBM + HCM row to AddPerson | ISEmployeeID / ManagerEmailAddress / OMFullname | OTBI s\_7 / s\_6 / s\_1 | passthrough | — | — | low | Manager person number, email, name. |
| MR-010 | EBM + HCM row to AddPerson | ISFullname / SOMFullname | Person/LastName, FirstName | calculated | — | "Last,First" (max 100) / "First Last" | low | Employee's own name; intended meaning of IS/SOM fields not determinable. |
| MR-011 | EBM + HCM row to AddPerson | Header BusinessUnit | literal "$ProjectID\_Desc" | transformed | — | TeleOpti\_BU lookup (broken) | medium | Should use project description s\_9. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | TeleOpti eligibility | Job Name ASSOCIATE or SENIOR ASSOCIATE and Assignment Category "Eligible for OTND". | High |
| BR-002 | Business unit from project | TeleOpti BU = TeleOpti\_BU(project description) (intended). | High |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive new agent** — No live caller today
2. **Read HCM details** — OTBI
3. **Eligible agent?** — ASSOCIATE / SENIOR ASSOCIATE, OTND
4. **Create TeleOpti person** — AddPerson with auth header
5. **Callback, no action**
6. **Notify, callback Success**
7. **Notify, callback Failure**

## Data Model

_No profile components are owned by this interface; the personDto shape comes from the TeleOpti SDK WSDL imported into the connector operations._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback | Process start + Return Documents |
| TeleoptiAuthenticationHeader from DVMs | fn-teleopti-auth-header + connection/extension fields |
| xform\_EBO\_to\_AddTeleOptiUser | map-ebm-to-teleopti-addperson |
| AddPerson | op-teleopti-add-person on conn-teleopti-sdk-soap |
| TeleOpti\_BU.dvm | Cross reference TeleOpti\_BU |

**Components**

| Component | Role |
| --- | --- |
| conn-teleopti-sdk-soap | TeleOpti SDK SOAP connection |
| op-teleopti-add-person | AddPerson |
| op-teleopti-get-persons | GetPersonsByQuery (by employment number) |
| fn-teleopti-auth-header | SOAP auth header builder |
| map-ebm-to-teleopti-addperson | AddPerson map |
| proc-teleopti-create-user | Main process |

## Client questions

- **Q-001** Should new agents still be created automatically in TeleOpti?
  **If this is not answered:** Out of scope; not built.
- **Q-002** How should an agent's TeleOpti business unit be chosen?
  **If this is not answered:** Look up the TeleOpti business unit by the HCM project description.
- **Q-003** What should the TeleOpti login name be, given that first names are not unique?
  **If this is not answered:** Use the AD account name (initials + employee number).
- **Q-004** Which TeleOpti service account should the new process use?
  **If this is not answered:** TaskUs creates a dedicated integration account in TeleOpti.
