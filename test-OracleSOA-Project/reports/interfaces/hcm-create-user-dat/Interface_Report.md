# ProcessCreateUserDAT

Test-OracleSOA-Project · Listener · simple complexity

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

Sets the Oracle HCM Cloud user name of one or more workers to their Active Directory account name. It writes an HCM Data Loader User.dat file (METADATA\|User\|PersonNumber\|Username header and one MERGE row per worker), zips and Base64-encodes it, checks it into UCM and calls HCM Data Loader importAndLoadData, returning the UCM content ID.

| Field | Value |
| --- | --- |
| Interface | ProcessCreateUserDAT |
| Type | Listener |
| Complexity | simple |
| Components to build | 3 |
| Mapping rules | 5 |
| Business rules | 1 |

## Interface Summary

This small service makes a worker's HCM sign-in name match their network (AD) account name. When a new AD account is created, the AD creation service calls it with the employee number and account name. It writes a one-line HCM Data Loader "User" file, packs it, uploads it to Oracle's content server and asks HCM to load it.  It repeats the logic of the shared HCM loading service instead of reusing it, and uses its own folder on the integration server. That folder is also used by the daily AD user-name job (update-ad-username-sch), and each run empties it, so two runs close together can lose or mix up files. Errors are emailed to one consultant in a format the notification service cannot read, and the HCM load result is never checked.  In Boomi this becomes a short sub-process: build the User.dat in memory and hand it to the shared HCM loading sub-process (hcm-hdl-loader).

**Complexity:** simple (score 22)

## Functional Analysis

1. **Receive request** — Async SOAP process (createInstance) with Input\[\] PersonNumber, Username.
2. **Write User.dat** — xform\_employee\_to\_dat builds header + MERGE rows; File adapter Write to /u01/data/domains/SOACS-Pr\_domain/osi\_adUser/put/User.dat.
3. **Zip and encode** — Java ZIPtoBASE64(folder, folder/User.zip): zip \*.dat, Base64 chunked, clean folder.
4. **Check in to UCM** — xform\_dat\_to\_ucm GenericRequest CHECKIN\_UNIVERSAL with Authorization header from Credentials.dvm.
5. **Import and load** — importAndLoadData with ContentId = dDocName and Authorization header.
6. **Return content ID** — Callback processResponse/result = ContentId.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessCreateUserDAT/processcreateuserdat\_client\_ep | none declared |
| outbound | File (SOA server) | /u01/data/domains/SOACS-Pr\_domain/osi\_adUser/put/User.dat | n/a |
| outbound | SOAP 1.1 over HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | HTTP Authorization header (Credentials.dvm) |
| outbound | SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCommonDataLoader/HCMDataLoader | HTTP Authorization header (Credentials.dvm) |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | User request to HDL User.dat | Header line | (none) | hardcoded | — | "METADATA\|User\|PersonNumber\|Username" | low | First Details record. |
| MR-002 | User request to HDL User.dat | METADATA | (none) | hardcoded | — | "MERGE" | low | — |
| MR-003 | User request to HDL User.dat | User | (none) | hardcoded | — | "User" | low | — |
| MR-004 | User request to HDL User.dat | PersonNumber | process/Input/PersonNumber | looped | — | — | low | One row per Input. |
| MR-005 | User request to HDL User.dat | Username | process/Input/Username | looped | — | — | low | AD sAMAccountName. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | HCM user name = AD account name | MERGE User rows set Username for PersonNumber. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Receive user rows** — From ad-create-user
2. **Build User.dat** — Header + MERGE rows
3. **Write, zip, encode** — Shared folder + Java
4. **Check in to UCM** — CHECKIN\_UNIVERSAL
5. **Import and load** — Not verified
6. **Email error** — Hardcoded individual
7. **Return content ID** — Callback

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| HDL User DAT Prof | flatfile | prof-hdl-user-dat |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| receive / callback | Sub-process start + Return Documents |
| xform\_employee\_to\_dat + File adapter | map-user-to-user-dat + prof-hdl-user-dat (in memory) |
| Java ZIPtoBASE64 + UCM + HDL calls | Process Call hcm-hdl-loader |
| catchAll to hardcoded individual | Try/Catch + util-send-notification (EmailDL) |

**Components**

| Component | Role |
| --- | --- |
| prof-hdl-user-dat | User.dat flat-file profile |
| map-user-to-user-dat | Request to User.dat |
| proc-sub-hcm-create-user-dat | Main sub-process |

## Client questions

- **Q-001** Who should be told when setting a new employee's HCM user name fails?
  **If this is not answered:** Send to the "itservicedesk" distribution list.
- **Q-002** Should the process confirm that HCM accepted the user-name change and report any rejected rows?
  **If this is not answered:** Yes, via the load-status check of the shared HCM loading sub-process.
