# Migration Plan — ProcessCreateUserDAT

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.986Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessCreateUserDAT |
| Identifier | hcm-create-user-dat |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | simple (22) |
| Components to build | 3 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/BPEL/ProcessCreateUserDAT.bpel |
| Depends on | hcm-hdl-loader, util-send-notification |

## 2. End-to-End Process Architecture

Async BPEL composite: File adapter write, Java zip/Base64 on the server folder, UCM GenericSoapPort and HCMDataLoader through MDS WSDLs, XXSendNotification on error. It duplicates the logic of ProcessHDLProvABCSImpl instead of calling it.

Boomi sub-process "Sub: HCM Create User DAT": map-user-to-user-dat builds User.dat in memory, Process Call hcm-hdl-loader (proc-sub-hdl-load) with title User and file User.zip, return ContentId; Try/Catch with util-send-notification to a distribution list.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback | Sub-process start + Return Documents |  |
| xform_employee_to_dat + File adapter | map-user-to-user-dat + prof-hdl-user-dat (in memory) |  |
| Java ZIPtoBASE64 + UCM + HDL calls | Process Call hcm-hdl-loader | Removes duplicated logic and the shared folder. |
| catchAll to hardcoded individual | Try/Catch + util-send-notification (EmailDL) |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive request | Async SOAP process (createInstance) with Input[] PersonNumber, Username. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Schemas/ProcessCreateUserDAT.xsd |
| 2 | Write User.dat | xform_employee_to_dat builds header + MERGE rows; File adapter Write to /u01/data/domains/SOACS-Pr_domain/osi_adUser/put/User.dat. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Transformations/xform_employee_to_dat.xsl |
| 3 | Zip and encode | Java ZIPtoBASE64(folder, folder/User.zip): zip *.dat, Base64 chunked, clean folder. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java |
| 4 | Check in to UCM | xform_dat_to_ucm GenericRequest CHECKIN_UNIVERSAL with Authorization header from Credentials.dvm. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Transformations/xform_dat_to_ucm.xsl |
| 5 | Import and load | importAndLoadData with ContentId = dDocName and Authorization header. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Transformations/xform_hcm_request.xsl |
| 6 | Return content ID | Callback processResponse/result = ContentId. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/BPEL/ProcessCreateUserDAT.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| HDL User DAT Prof | flatfile | ADUser/Details/METADATA | string | yes |
| HDL User DAT Prof | flatfile | ADUser/Details/User | string | yes |
| HDL User DAT Prof | flatfile | ADUser/Details/PersonNumber | string | yes |
| HDL User DAT Prof | flatfile | ADUser/Details/Username | string | yes |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | Header line | (none) | hardcoded |  | "METADATA\|User\|PersonNumber\|Username" | low | First Details record. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Transformations/xform_employee_to_dat.xsl | User request to HDL User.dat |
| MR-002 | METADATA | (none) | hardcoded |  | "MERGE" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Transformations/xform_employee_to_dat.xsl | User request to HDL User.dat |
| MR-003 | User | (none) | hardcoded |  | "User" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Transformations/xform_employee_to_dat.xsl | User request to HDL User.dat |
| MR-004 | PersonNumber | process/Input/PersonNumber | looped |  |  | low | One row per Input. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Transformations/xform_employee_to_dat.xsl | User request to HDL User.dat |
| MR-005 | Username | process/Input/Username | looped |  |  | low | AD sAMAccountName. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Transformations/xform_employee_to_dat.xsl | User request to HDL User.dat |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | HCM user name = AD account name | MERGE User rows set Username for PersonNumber. | Medium | map-user-to-user-dat then Process Call hcm-hdl-loader (dDocTitle User, file User.zip). |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessCreateUserDAT/processcreateuserdat_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processcreateuserdat_client_ep.wsdl |
| outbound File (SOA server) | /u01/data/domains/SOACS-Pr_domain/osi_adUser/put/User.dat | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Adapters/WriteUserDATFile_file.jca |
| outbound SOAP 1.1 over HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/GenericSoapPort.wsdl |
| outbound SOAP 1.1 over HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCommonDataLoader/HCMDataLoader | HTTP Authorization header (Credentials.dvm) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/HCMDataLoader.wsdl |
| File adapter WriteUserDATFile | JNDI eis/FileAdapter; PhysicalDirectory [redacted]; FileNamingConvention User.dat; Append false; NumberMessages 1; NXSD UserDATFile.xsd (pipe-delimited). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Adapters/WriteUserDATFile_file.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-hdl-loader | Sub: HCM HDL Zip Checkin and Load | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java | Zip *.dat in the folder into User.zip, Base64-encode, clean the folder. | fn-zip-base64 inside proc-sub-hdl-load (hcm-hdl-loader). | Medium |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Transformations/xform_employee_to_dat.xsl | Input rows to User.dat records with header. | map-user-to-user-dat into prof-hdl-user-dat. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Transformations/xform_dat_to_ucm.xsl | UCM check-in request for User.zip. | Parameters passed to proc-sub-hdl-load. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | catchAll: notify hardcoded address, callback with empty result. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| oramds:/apps/Provider/AD/processcreateuserdat_client_ep.wsdl | mds | yes |
| Credentials.dvm (Authorization) | dvm | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Oracle HCM Cloud UCM | SOAP / HTTPS | https://eetm.fs.us6.oraclecloud.com:443/idcws/GenericSoapPort | Oracle (TaskUs tenant) |
| Oracle HCM Cloud HCM Data Loader | SOAP / HTTPS | https://eetm.hcm.us6.oraclecloud.com:443/hcmCommonDataLoader/HCMDataLoader | Oracle (TaskUs tenant) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | DAT folder | [redacted] | [redacted].jca |

## 14. Boomi Migration Readiness Summary

This small service makes a worker's HCM sign-in name match their network (AD) account name. When a new AD account is created, the AD creation service calls it with the employee number and account name. It writes a one-line HCM Data Loader "User" file, packs it, uploads it to Oracle's content server and asks HCM to load it.

It repeats the logic of the shared HCM loading service instead of reusing it, and uses its own folder on the integration server. That folder is also used by the daily AD user-name job (update-ad-username-sch), and each run empties it, so two runs close together can lose or mix up files. Errors are emailed to one consultant in a format the notification service cannot read, and the HCM load result is never checked.

In Boomi this becomes a short sub-process: build the User.dat in memory and hand it to the shared HCM loading sub-process (hcm-hdl-loader).

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | HDL User DAT Prof | prof-hdl-user-dat | profile.flatfile | — |
| 2 | Map \| User Request to HDL User DAT | map-user-to-user-dat | transform.map | prof-hdl-user-dat |
| 3 | Sub: HCM Create User DAT | proc-sub-hcm-create-user-dat | process | map-user-to-user-dat, proc-sub-hdl-load, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/BPEL/ProcessCreateUserDAT.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Adapters/WriteUserDATFile_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Schemas/ProcessCreateUserDAT.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Schemas/UserDATFile.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Transformations/xform_employee_to_dat.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Transformations/xform_dat_to_ucm.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateUserDAT/SOA/Transformations/xform_hcm_request.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processcreateuserdat_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessCreateADUser/SOA/BPEL/ProcessCreateADUser.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/UpdateADUserName_Sch/SOA/Adapters/WriteUserDATFile_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessHDLProvABCSImpl/SOA/SCA-INF/src/com/taskus/osi/ZIPtoBASE64.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/GenericSoapPort.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/HCMDataLoader.wsdl

## Open questions

### Still outstanding

REVIEW-NEEDED: Who should be told when setting a new employee's HCM user name fails? (Errors go to one consultant's personal address today, in a format that is probably not delivered.)
  - If this is not answered: Send to the "itservicedesk" distribution list.

REVIEW-NEEDED: Should the process confirm that HCM accepted the user-name change and report any rejected rows? (The load result is never checked today.)
  - If this is not answered: Yes, via the load-status check of the shared HCM loading sub-process.

