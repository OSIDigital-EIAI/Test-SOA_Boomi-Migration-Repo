# UpdateADUserName\_Sch

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

Every day this job takes the workers whose assignment starts today and is active and payroll-eligible, checks that each has an Active Directory account named by the TaskUs convention (initial of first name + initial of last name + person number), and loads those account names into Oracle HCM Cloud as the workers' HCM user names through HCM Data Loader (a User.dat file).

| Field | Value |
| --- | --- |
| Interface | UpdateADUserName\_Sch |
| Type | Scheduled |
| Complexity | medium |
| Components to build | 1 |
| Mapping rules | 5 |
| Business rules | 3 |

## Interface Summary

This job keeps Oracle HCM user names in line with Active Directory. Each day it asks HCM for workers whose assignment starts that day, works out each person's AD account name using the TaskUs rule (first letter of first name, first letter of last name, person number, all in lower case), and checks that the account exists in AD. For every match it adds a line to an HCM Data Loader file that sets the HCM user name to the AD name, then sends the file to HCM.  Today the file is built on the integration server, zipped and encoded by a small Java program that then empties the folder. Another integration (the AD account creation's user-name load) uses the same folder, so the two can interfere. In Boomi the job will reuse the shared HCM user-name load sub-process and work entirely in memory.  The job filters on "today" rather than on its scheduled time window, so a missed day is never caught up. It also does not check whether HCM actually loaded the file, and its error email goes to a single hardcoded person in a format the notification utility cannot read, so failures are probably unnoticed.

**Complexity:** medium (score 45)

## Functional Analysis

1. **Start and read window** — Direct binding; Select\_XXTS\_Scheduling for UpdateADUserName\_Sch; updatedMin/Max formatted; EffectiveStartDate = current date (yyyy-MM-dd). SOAP client path uses caller-supplied EffectiveStartDate.
2. **Query today's starters** — OTBI via ProcessHCMExtractSyncProv: SELECT Person Number s\_1, Employee First Name s\_2, Employee Last Name s\_3, Effective Start Date s\_4 FROM "Workforce Management - Worker Assignment Real Time" WHERE Effective Start Date = '\<today\>' AND Assignment Status = 'Active - Payroll Eligible'.
3. **Any starters?** — If count(Row) = 0 nothing else is done except the window update.
4. **Check AD account per worker** — forEach parallel="no": LDAP search (eis/ldap/TestConnection) for sAMAccountName = lower(initial Column2)+lower(initial Column3)+Column1; match if entriesReturned contains "1".
5. **Collect User.dat rows** — For each match add Details {METADATA "MERGE", User "User", PersonNumber Column1, Username derived}.
6. **Write User.dat** — File adapter writes User.dat (pipe-delimited, header METADATA\|User\|PersonNumber\|Username) to /u01/data/domains/SOACS-Pr\_domain/osi\_adUser/put. Runs even if no worker matched.
7. **Zip and encode** — Java ZIPtoBASE64: zip all \*.dat files in the folder into User.zip, Base64 (chunked), then delete everything in the folder; errors only audited.
8. **Check in to UCM** — GenericSoapOperation CHECKIN\_UNIVERSAL: dDocTitle User, dSecurityGroup FAFusionImportExport, dDocType Document, dDocAccount hcm$/dataloader$/import$, primaryFile User.zip; Authorization header from Credentials.dvm.
9. **Load into HCM** — HCMDataLoader.importAndLoadData with ContentId = UCM dDocName; Authorization header from Credentials.dvm; result not checked.
10. **Advance window** — Update\_Scheduling\_StartDateTime and Update\_XXTS\_Schedule; callback on the direct binding.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding / SOAP async | updateadusername\_sch\_direct\_binding / updateadusername\_sch\_client\_ep | none declared |
| outbound | LDAP | JNDI eis/ldap/TestConnection (host not in export) | bind account configured in WebLogic |
| outbound | SOAP 1.1 over HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | HTTP Authorization header (Credentials.dvm) |
| outbound | SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCommonDataLoader/HCMDataLoader | HTTP Authorization header (Credentials.dvm) |
| outbound | SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared |
| outbound | File (SOA server) | /u01/data/domains/SOACS-Pr\_domain/osi\_adUser/put/User.dat | n/a |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | OTBI starter row to User.dat record | METADATA | (none) | hardcoded | AD account found | "MERGE" | low | — |
| MR-002 | OTBI starter row to User.dat record | User | (none) | hardcoded | — | "User" | low | — |
| MR-003 | OTBI starter row to User.dat record | PersonNumber | Row/Column1 | passthrough | — | — | low | — |
| MR-004 | OTBI starter row to User.dat record | Username | Row/Column2, Column3, Column1 | calculated | — | lower(first char First Name)+lower(first char Last Name)+Person Number | medium | Same rule used for the LDAP filter; legacy idiom substring-before(x,substring(x,2)). |
| MR-005 | OTBI starter row to User.dat record | File header | (none) | hardcoded | — | "METADATA\|User\|PersonNumber\|Username" | low | — |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | Starter selection | Workers with assignment Effective Start Date = today and Assignment Status = Active - Payroll Eligible. | High |
| BR-002 | AD user name convention | Username = lower(first letter of first name) + lower(first letter of last name) + person number; only loaded if the AD account exists. | High |
| BR-003 | HCM user name load | Matched users are loaded with HDL User object MERGE rows. | High |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Scheduled start 15:30 UTC** — Direct binding
2. **Query today's starters** — OTBI Effective Start Date = today
3. **Any starters?** — count(Row) != 0
4. **Notify error, advance window** — Hardcoded recipient
5. **Search AD for user name** — LDAP sAMAccountName = initials + person number
6. **Account found?** — entriesReturned contains 1
7. **Add User.dat row** — MERGE\|User\|PersonNumber\|Username
8. **More starters?** — forEach parallel=no
9. **Write, zip, encode file** — File adapter + Java ZIPtoBASE64
10. **Check in to UCM** — CHECKIN\_UNIVERSAL
11. **HDL import and load** — importAndLoadData; not verified
12. **Advance window, callback**

## Data Model

_This interface owns only its process component; it uses the OTBI rowset profile (hcm-otbi-sql-query) and the User DAT profile (hcm-create-user-dat)._

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| ProcessHCMExtractSyncProv | Process Call hcm-otbi-sql-query |
| ValidateADUser LDAP adapter | op-ldap-search (ad-create-user) |
| File adapter + Java ZIPtoBASE64 + UCM + HDL calls | Process Call hcm-create-user-dat |
| catchAll to hardcoded address | Try/Catch + util-send-notification (EmailDL) |

**Components**

| Component | Role |
| --- | --- |
| proc-update-ad-username-sch | Main scheduled process |

## Client questions

- **Q-001** If this job does not run on a given day, should that day's starters be picked up on the next run?
  **If this is not answered:** Select starters from the last successful run date up to today.
- **Q-002** Should starters whose AD account does not exist yet be reported, and to whom?
  **If this is not answered:** List them in an email to the AD distribution list and retry them on the next run.
- **Q-003** Should the job confirm with HCM that the user names were actually loaded and report any load errors?
  **If this is not answered:** Check the load status and report errors to the notification list.
- **Q-004** Who should receive error notices for this job?
  **If this is not answered:** Send to the "notification" distribution list.
