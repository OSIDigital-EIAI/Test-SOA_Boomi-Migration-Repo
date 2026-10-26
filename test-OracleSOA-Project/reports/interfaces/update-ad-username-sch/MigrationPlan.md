# Migration Plan — UpdateADUserName_Sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.846Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | UpdateADUserName_Sch |
| Identifier | update-ad-username-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | medium (45) |
| Components to build | 1 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/BPEL/UpdateADUserName_Sch.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/UpdateADUserName_Sch_Prod_cfgplan.xml |
| Depends on | newhire-emp-sch, hcm-otbi-sql-query, ad-create-user, hcm-create-user-dat, util-send-notification |

## 2. End-to-End Process Architecture

BPEL composite started by the XXTS_SCHEDULING servlet. It queries OTBI through ProcessHCMExtractSyncProv, checks AD with the LDAP adapter, writes User.dat with the File adapter, zips and Base64-encodes it with an embedded Java class, checks it into UCM and calls HCMDataLoader through MDS WSDLs. This duplicates the logic of the ProcessCreateUserDAT composite and shares its folder.

Scheduled Boomi process "AD Username to HCM User Sync Sch": shared window sub-process, hcm-otbi-sql-query for starters, op-ldap-search and fn-ad-sam-build from ad-create-user for the AD check, then a single Process Call to hcm-create-user-dat (which builds User.dat in memory and loads it through hcm-hdl-loader) when at least one account matched. Errors go to util-send-notification with a distribution list.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| ProcessHCMExtractSyncProv | Process Call hcm-otbi-sql-query |  |
| ValidateADUser LDAP adapter | op-ldap-search (ad-create-user) |  |
| File adapter + Java ZIPtoBASE64 + UCM + HDL calls | Process Call hcm-create-user-dat | Removes duplicated logic and the shared folder. |
| catchAll to hardcoded address | Try/Catch + util-send-notification (EmailDL) |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Start and read window | Direct binding; Select_XXTS_Scheduling for UpdateADUserName_Sch; updatedMin/Max formatted; EffectiveStartDate = current date (yyyy-MM-dd). SOAP client path uses caller-supplied EffectiveStartDate. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/BPEL/UpdateADUserName_Sch.bpel |
| 2 | Query today's starters | OTBI via ProcessHCMExtractSyncProv: SELECT Person Number s_1, Employee First Name s_2, Employee Last Name s_3, Effective Start Date s_4 FROM "Workforce Management - Worker Assignment Real Time" WHERE Effective Start Date = '<today>' AND Assignment Status = 'Active - Payroll Eligible'. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/BPEL/UpdateADUserName_Sch.bpel |
| 3 | Any starters? | If count(Row) = 0 nothing else is done except the window update. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/BPEL/UpdateADUserName_Sch.bpel |
| 4 | Check AD account per worker | forEach parallel="no": LDAP search (eis/ldap/TestConnection) for sAMAccountName = lower(initial Column2)+lower(initial Column3)+Column1; match if entriesReturned contains "1". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/BPEL/UpdateADUserName_Sch.bpel |
| 5 | Collect User.dat rows | For each match add Details {METADATA "MERGE", User "User", PersonNumber Column1, Username derived}. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/BPEL/UpdateADUserName_Sch.bpel |
| 6 | Write User.dat | File adapter writes User.dat (pipe-delimited, header METADATA\|User\|PersonNumber\|Username) to /u01/data/domains/SOACS-Pr_domain/osi_adUser/put. Runs even if no worker matched. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Adapters/WriteUserDATFile_file.jca |
| 7 | Zip and encode | Java ZIPtoBASE64: zip all *.dat files in the folder into User.zip, Base64 (chunked), then delete everything in the folder; errors only audited. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java |
| 8 | Check in to UCM | GenericSoapOperation CHECKIN_UNIVERSAL: dDocTitle User, dSecurityGroup FAFusionImportExport, dDocType Document, dDocAccount hcm$/dataloader$/import$, primaryFile User.zip; Authorization header from Credentials.dvm. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Transformations/xform_CheckIN_UCM.xsl |
| 9 | Load into HCM | HCMDataLoader.importAndLoadData with ContentId = UCM dDocName; Authorization header from Credentials.dvm; result not checked. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Transformations/xform_HCMDataLoader.xsl |
| 10 | Advance window | Update_Scheduling_StartDateTime and Update_XXTS_Schedule; callback on the direct binding. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/BPEL/UpdateADUserName_Sch.bpel |

## 4. Payload Profiles

> **Not available.** This interface owns only its process component; it uses the OTBI rowset profile (hcm-otbi-sql-query) and the User DAT profile (hcm-create-user-dat).

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | METADATA | (none) | hardcoded | AD account found | "MERGE" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/BPEL/UpdateADUserName_Sch.bpel | OTBI starter row to User.dat record |
| MR-002 | User | (none) | hardcoded |  | "User" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/BPEL/UpdateADUserName_Sch.bpel | OTBI starter row to User.dat record |
| MR-003 | PersonNumber | Row/Column1 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/BPEL/UpdateADUserName_Sch.bpel | OTBI starter row to User.dat record |
| MR-004 | Username | Row/Column2, Column3, Column1 | calculated |  | lower(first char First Name)+lower(first char Last Name)+Person Number | medium | Same rule used for the LDAP filter; legacy idiom substring-before(x,substring(x,2)). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/BPEL/UpdateADUserName_Sch.bpel | OTBI starter row to User.dat record |
| MR-005 | File header | (none) | hardcoded |  | "METADATA\|User\|PersonNumber\|Username" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Schemas/UserDATFile.xsd | OTBI starter row to User.dat record |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | Starter selection | Workers with assignment Effective Start Date = today and Assignment Status = Active - Payroll Eligible. | High | OTBI SQL via hcm-otbi-sql-query (date source to be decided). |
| BR-002 | AD user name convention | Username = lower(first letter of first name) + lower(first letter of last name) + person number; only loaded if the AD account exists. | High | fn-ad-sam-build (ad-create-user) + op-ldap-search. |
| BR-003 | HCM user name load | Matched users are loaded with HDL User object MERGE rows. | High | Process Call hcm-create-user-dat with the collected rows (skip when none matched). |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding / SOAP async | updateadusername_sch_direct_binding / updateadusername_sch_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/composite.xml |
| outbound LDAP | JNDI eis/ldap/TestConnection (host not in export) | bind account configured in WebLogic | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Adapters/ValidateADUser_ldap.jca |
| outbound SOAP 1.1 over HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/GenericSoapPort.wsdl |
| outbound SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCommonDataLoader/HCMDataLoader | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/HCMDataLoader.wsdl |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/composite.xml |
| outbound File (SOA server) | /u01/data/domains/SOACS-Pr_domain/osi_adUser/put/User.dat | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Adapters/WriteUserDATFile_file.jca |
| LDAP adapter ValidateADUser | JNDI eis/ldap/TestConnection; search; SearchScope wholeSubtree; ReturnAttributes cn; SizeLimit 1000; TimeLimit 0; FollowReferrals ignore; AliasDereferencing never; base DN and filter set in BPEL. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Adapters/ValidateADUser_ldap.jca |
| File adapter WriteUserDATFile | JNDI eis/FileAdapter; PhysicalDirectory [redacted]; FileNamingConvention User.dat; Append false; NXSD UserDATFile.xsd (pipe-delimited, outbound header "METADATA\|User\|PersonNumber\|Username"). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Adapters/WriteUserDATFile_file.jca |
| DB adapters (XXTS_SCHEDULING) | Select window; advance start; set last run. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| newhire-emp-sch | Sub: Schedule Window Get and Advance (shared) | synchronous |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| ad-create-user | AD User SEARCH Op and sAMAccountName builder (shared components) | synchronous |
| hcm-create-user-dat | Sub: HCM Create User DAT | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java | Zip every .dat in the folder, Base64-encode (chunked), clean the folder. | Reuse hcm-hdl-loader zip/Base64 (fn-zip-base64) via hcm-create-user-dat; in-memory, no folder. | Medium |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Transformations/xform_CheckIN_UCM.xsl | UCM CHECKIN_UNIVERSAL request (href points to a Te_domain path; only the file name matters). | Part of hcm-hdl-loader. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Transformations/xform_HCMDataLoader.xsl | importAndLoadData ContentId from UCM dDocName. | Part of hcm-hdl-loader. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Process catchAll: notify hardcoded address, advance window, callback. Java errors audited only. |
| Retries | DB adapter retries only. |
| Compensation | None; no HDL status check. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| Credentials.dvm (Authorization) | dvm | yes |
| oramds:/apps/Provider/HCM/GenericSoapPort.wsdl, HCMDataLoader.wsdl | mds | yes |
| commons-io / commons-codec jars (SCA-INF/lib) | library | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud UCM | SOAP / HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | Oracle (TaskUs tenant) |
| Oracle HCM Cloud HCM Data Loader | SOAP / HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCommonDataLoader/HCMDataLoader | Oracle (TaskUs tenant) |
| TaskUs Active Directory | LDAP | eis/ldap/TestConnection (host not in export) | TaskUs IT |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | DAT folder | [redacted] | [redacted].jca |
| PROD | HCMDataLoader | https://eetm.hcm.us6.oraclecloud.com:[redacted] | [redacted].wsdl |

## 14. Boomi Migration Readiness Summary

This job keeps Oracle HCM user names in line with Active Directory. Each day it asks HCM for workers whose assignment starts that day, works out each person's AD account name using the TaskUs rule (first letter of first name, first letter of last name, person number, all in lower case), and checks that the account exists in AD. For every match it adds a line to an HCM Data Loader file that sets the HCM user name to the AD name, then sends the file to HCM.

Today the file is built on the integration server, zipped and encoded by a small Java program that then empties the folder. Another integration (the AD account creation's user-name load) uses the same folder, so the two can interfere. In Boomi the job will reuse the shared HCM user-name load sub-process and work entirely in memory.

The job filters on "today" rather than on its scheduled time window, so a missed day is never caught up. It also does not check whether HCM actually loaded the file, and its error email goes to a single hardcoded person in a format the notification utility cannot read, so failures are probably unnoticed.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | AD Username to HCM User Sync Sch | proc-update-ad-username-sch | process | proc-sub-schedule-window, proc-sub-otbi-query, op-ldap-search, fn-ad-sam-build, proc-sub-hcm-create-user-dat, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/BPEL/UpdateADUserName_Sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/UpdateADUserName_Sch_Prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Adapters/ValidateADUser_ldap.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Adapters/WriteUserDATFile_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Schemas/UserDATFile.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Transformations/xform_CheckIN_UCM.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Transformations/xform_HCMDataLoader.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/GenericSoapPort.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/HCMDataLoader.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Adapters/WriteUserDATFile_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utilities/XXSendNotification/SOA/SCA-INF/src/com/osi/FormatEmailContent.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Credentials.dvm

## Open questions

### Still outstanding

REVIEW-NEEDED: If this job does not run on a given day, should that day's starters be picked up on the next run? (The job selects only workers starting "today", so a missed day is never caught up.)
  - If this is not answered: Select starters from the last successful run date up to today.

REVIEW-NEEDED: Should starters whose AD account does not exist yet be reported, and to whom? (Today they are silently skipped and never retried.)
  - If this is not answered: List them in an email to the AD distribution list and retry them on the next run.

REVIEW-NEEDED: Should the job confirm with HCM that the user names were actually loaded and report any load errors? (The current job never checks the HCM Data Loader result.)
  - If this is not answered: Check the load status and report errors to the notification list.

REVIEW-NEEDED: Who should receive error notices for this job? (Errors currently go to one consultant's personal address.)
  - If this is not answered: Send to the "notification" distribution list.

