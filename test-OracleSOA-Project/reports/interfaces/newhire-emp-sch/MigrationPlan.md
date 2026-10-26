# Migration Plan — ProcessNewHireEmpReq_Sch

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:06.749Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessNewHireEmpReq_Sch |
| Identifier | newhire-emp-sch |
| Trigger | Scheduled |
| Starts with | No inbound data |
| Complexity | complex (88) |
| Components to build | 17 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/ProcessNewHireEmpReq_Sch_cfgplan.xml |
| Depends on | hcm-otbi-sql-query, ad-create-user, gsuite-user-lifecycle, hcm-update-employee-email, util-send-notification, util-file-notification, hcm-dat-generation |

## 2. End-to-End Process Architecture

A BPEL composite started by a custom WebLogic TimerManager servlet through its direct binding. It reads and advances its window in table XXTS_SCHEDULING with three DB adapter references, queries HCM through the ProcessHCMExtractSyncProv composite, orchestrates the ProcessUpdateGSuitUser, ProcessUpdateEmployeeEmail and ProcessCreateADUser composites (two of them asynchronously with callback receives), appends three CSV reports to a server folder with the File adapter and asks FileNotification to email them. Several references (ATOM feed, HCM REST, TeleOpti) are wired but disabled.

A scheduled Boomi process "NewHire Employee Provisioning Sch" (no-data start, simultaneous executions disallowed) that calls "Sub: Schedule Window Get and Advance", then "Sub: HCM OTBI Execute SQL" for new hires; splits rows and, per person inside a Try/Catch, runs the two OTBI lookups, the Groovy name/email script and the maps, and calls the gsuite-user-lifecycle, hcm-update-employee-email and ad-create-user sub-processes synchronously. Report rows are accumulated as documents (Combine) and sent with "Sub: Send File Notification"; errors go to "Sub: Send Notification". This interface also owns the shared scheduling-window and HCM REST/ATOM components used by the other schedulers.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| SchedulerServlet + direct binding | Boomi process schedule (09:00, 21:00 UTC) |  |
| Select/Update XXTS_SCHEDULING DB adapters | conn-soainfra-db + op-xxts-* in proc-sub-schedule-window (or persisted property) | Decision pending. |
| ProcessHCMExtractSyncProv calls (x1 + x2 per person) | Process Call to hcm-otbi-sql-query sub-process |  |
| xform_Extract_GetEmp.xsl | map-newhire-rowset-to-employee |  |
| Java_Remove_Unwanted + email assigns | fn-newhire-name-email (Groovy) |  |
| xform_hcm_to_gmailcreation.xsl + async ProcessUpdateGSuitUser | map-newhire-to-gsuite-create + Process Call gsuite-user-lifecycle | Synchronous in Boomi. |
| ProcessUpdateEmployeeEmail | Process Call hcm-update-employee-email |  |
| xform_hcm_to_AD.xsl + async ProcessCreateADUser | map-newhire-to-ad-create + Process Call ad-create-user |  |
| File adapter appends to /u01/.../EmailAttachment + FileNotification | prof-newhire-report-csv documents + Combine + util-file-notification | No server folder. |
| catchAll blocks + XXSendNotification | Try/Catch + util-send-notification |  |
| Disabled ATOM / HCM REST / TeleOpti references | Not built for this process (ATOM and emps operations owned here for other schedulers) |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Start and read time window | Started by the scheduler through the direct binding. Selects start_date_time and start_date_time+INTERVAL from XXTS_SCHEDULING for INTERFACE_NAME ProcessNewHireEmpReq_Sch, formats both as yyyy-MM-ddTHH:mm:ss and appends .000Z / .999Z. (A manual SOAP entry uses caller-supplied updatedMin/updatedMax instead.) | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel |
| 2 | Find new hires in HCM | OTBI via ProcessHCMExtractSyncProv: SET VARIABLE PARAM_EFFECTIVE_DATE='4000-12-31'; SELECT Effective Start Date s_1, Person Number s_2, Creation Date s_3, Last Update Date s_4, Person Start Date s_5 FROM "Workforce Management - Person Real Time" WHERE "Person"."Creation Date" BETWEEN <min> AND <max>. Rowset parsed with ora:parseEscapedXML. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel |
| 3 | Stop if nobody found | If there are no rows (the ATOM-feed alternative is always empty because that call is skipped) the run goes straight to advancing the window. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel |
| 4 | Start the three reports | TimeStamp = now; header rows written (File adapter, append) to GSuiteFileAttachment_<ts>.csv, ADFileAttachment_<ts>.csv and ADFileAttachment_PWD_<ts>.csv in AttachmentPath. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel |
| 5 | For each new hire (sequential) | forEach parallel="no" over the rows; each iteration has its own catchAll that emails EmailDL "notification" with the person number and name and continues. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel |
| 6 | Get person details and location | Two OTBI calls: employee details (DOB, email, first/last/middle name, hire date, postal code, region, person number, phone, known-as, manager name, country) and location (Location Code, Worker Location Name). Location name is replaced by the ADOfficeLocations code, or blank if not found. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel |
| 7 | Map to employee record | xform_Extract_GetEmp maps the rowset columns to the HCM "User/items" structure; first and last name are copied to $Details. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_Extract_GetEmp.xsl |
| 8 | Reject names with "?" | If first or last name contains "?", IT service desk is emailed ("User Name contains Special Charcter") and the person is skipped. Persons with an empty first name are also skipped. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel |
| 9 | Clean names and build email | Java embedding strips a trailing " Jr" from the first name, trailing " II", " III", " Jr" from the last name and any "Jr."; then the work email is built (ma./first-word/First.Last rule, @taskus.com, spaces removed). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel |
| 10 | Create Google account | xform_hcm_to_gmailcreation builds a ProcessUpdateGSuitUser request with Operation=Create; the call is asynchronous and the process waits for the callback (CreateGSuitUser_CallBack). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl |
| 11 | Write work email to HCM | If the returned primaryEmail is not empty: WorkEmail = primaryEmail and ProcessUpdateEmployeeEmail is called synchronously with PersonNumber, DateFrom = window row Effective Start Date, EmailAddress. A GSuite "Success" report row is written; otherwise a "Failure" row with the response kind. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel |
| 12 | Create AD account | xform_hcm_to_AD builds the EmployeeEBM request (EmployeeID, EmailAddress, MobilePhone), first/last name from $Details; ProcessCreateADUser is called asynchronously and the process waits for its callback. Runs regardless of the Google outcome. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_AD.xsl |
| 13 | Record AD outcome | status Success -> rows in the AD report and the AD password report; status Failed -> AD report row "AD Creation failed for this User" with MessageText; otherwise -> "This user is already existing". | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel |
| 14 | Email the three reports | FileNotification called three times: GSuite report to EmailDL GSuit ("Date@<now>#GSuite Consolidated Notification"), AD report to EmailDL AD ("...#AD Consolidated Notification"), AD password report to EmailDL AD ("...#AD Consolidated Notification with Password"); file paths from Domain.dvm Email path. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel |
| 15 | Advance window and acknowledge | Direct binding: UPDATE xxts_scheduling start_date_time = start + (INTERVAL+1)/86400, set last_scheduled_time = now, then callback. SOAP entry: callback only. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel |

## 4. Payload Profiles

| Profile | Format | Field | Type | Required |
| --- | --- | --- | --- | --- |
| HCM Emps GET Resp JSON Prof | json | User/items/FirstName | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/MiddleName | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/LastName | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/PreferredName | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/ManagerName | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/PersonNumber | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/WorkPhoneNumber | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/WorkMobilePhoneNumber | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/WorkEmail | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/AddressLine1 | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/Region | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/PostalCode | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/DateOfBirth | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/HireDate | string | no |
| HCM Emps GET Resp JSON Prof | json | User/items/links[]/href | string | no |
| NewHire Consolidated Report CSV Prof | flatfile | Column1..Column20 | string | no |
| NewHire Consolidated Report CSV Prof | flatfile | GSuite report header | string | no |
| NewHire Consolidated Report CSV Prof | flatfile | AD report header | string | no |
| NewHire Consolidated Report CSV Prof | flatfile | AD password report header | string | no |
| HCM Atom Employee Feed Resp XML Prof | xml | feed/entry | repeating | no |
| HCM Atom Employee Feed Resp XML Prof | xml | feed/entry/summary | string | no |
| HCM Atom Employee Feed Resp XML Prof | xml | feed/entry/updated | string | no |
| HCM Atom Employee Feed Resp XML Prof | xml | feed/entry/content | string | no |

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | items/FirstName | Row/Column3 (Employee First Name) | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_Extract_GetEmp.xsl | OTBI employee rowset to employee record |
| MR-002 | items/MiddleName | Row/Column8 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_Extract_GetEmp.xsl | OTBI employee rowset to employee record |
| MR-003 | items/LastName | Row/Column5 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_Extract_GetEmp.xsl | OTBI employee rowset to employee record |
| MR-004 | items/ManagerName | Row/Column12 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_Extract_GetEmp.xsl | OTBI employee rowset to employee record |
| MR-005 | items/PreferredName | Row/Column11 (Employee Known As) | passthrough |  |  | low | Becomes the Google alias. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_Extract_GetEmp.xsl | OTBI employee rowset to employee record |
| MR-006 | items/PersonNumber | Row/Column9 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_Extract_GetEmp.xsl | OTBI employee rowset to employee record |
| MR-007 | items/WorkPhoneNumber and WorkMobilePhoneNumber | Row/Column10 (Primary Phones Number) | passthrough |  |  | low | Same value in both. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_Extract_GetEmp.xsl | OTBI employee rowset to employee record |
| MR-008 | items/WorkEmail | Row/Column2 (Employee Email Address) | passthrough |  |  | low | Overwritten later by the generated address. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_Extract_GetEmp.xsl | OTBI employee rowset to employee record |
| MR-009 | items/Region, PostalCode | Row/Column7, Row/Column6 | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_Extract_GetEmp.xsl | OTBI employee rowset to employee record |
| MR-010 | items/DateOfBirth, HireDate | Row/Column1, Row/Column4 | passthrough |  |  | low | Column13 (Country) is not mapped; used directly in reports. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_Extract_GetEmp.xsl | OTBI employee rowset to employee record |
| MR-011 | items/FirstName (cleaned) | items/FirstName | transformed | trailing " Jr" | remove trailing " Jr"; remove "Jr."; trim | low | Applied after $Details was copied, so only the email uses cleaned names. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Name clean-up and work email |
| MR-012 | items/LastName (cleaned) | items/LastName | transformed | trailing " II", " III", " Jr" | remove suffix; remove "Jr."; trim | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Name clean-up and work email |
| MR-013 | items/WorkEmail | cleaned FirstName, LastName | conditional | lower(FirstName) contains "ma." \| FirstName contains " " \| otherwise | "ma."+Last \| first word+"."+Last \| First+"."+Last, + "@taskus.com", all spaces removed | medium | Uniqueness is handled by gsuite-user-lifecycle (email generation). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Name clean-up and work email |
| MR-014 | Email, User/primaryEmail, User/emails/address | items/WorkEmail | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl | Employee to Google create request |
| MR-015 | Operation | (none) | hardcoded |  | "Create" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl | Employee to Google create request |
| MR-016 | EID | items/PersonNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl | Employee to Google create request |
| MR-017 | Alias | items/PreferredName | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl | Employee to Google create request |
| MR-018 | User/name/givenName | $Details/firstName | passthrough |  |  | low | Uncleaned name. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl | Employee to Google create request |
| MR-019 | User/name/familyName | $Details/lastName | transformed |  | translate(lastName," ","") | low | Uncleaned, spaces removed. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl | Employee to Google create request |
| MR-020 | User/password | (none) | hardcoded |  | literal "" (two quote characters) | low | The callee generates the password. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl | Employee to Google create request |
| MR-021 | User/hashFunction | (none) | hardcoded |  | "false" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl | Employee to Google create request |
| MR-022 | User/addresses/streetAddress, region, postalCode | items/AddressLine1, Region, PostalCode | passthrough |  |  | low | AddressLine1 is never populated by MAP-001. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl | Employee to Google create request |
| MR-023 | User/organizations/name, title | (none) | hardcoded |  | "taskus.com", "SWE" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl | Employee to Google create request |
| MR-024 | User/phones/value | items/WorkMobilePhoneNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl | Employee to Google create request |
| MR-025 | PersonNumber | items/PersonNumber | passthrough | primaryEmail returned |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Employee to HCM work email update |
| MR-026 | DateFrom | new-hire Row/Column1 (Effective Start Date) | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Employee to HCM work email update |
| MR-027 | EmailAddress | Google response primaryEmail | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Employee to HCM work email update |
| MR-028 | Employee/EmployeeID | items/PersonNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_AD.xsl | Employee to AD create request |
| MR-029 | Employee/Person/EmailAddress | items/WorkEmail | passthrough |  |  | low | Google primaryEmail when Google succeeded, else the generated base address. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_AD.xsl | Employee to AD create request |
| MR-030 | Employee/EmployeePSFTType/MobilePhone | items/WorkPhoneNumber | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_AD.xsl | Employee to AD create request |
| MR-031 | Employee/Person/FirstName, LastName | $Details/firstName, lastName | passthrough |  |  | low | Uncleaned names. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Employee to AD create request |
| MR-032 | Column1 Sl_No / Column2 Instance_ID / Column3 Date | loop counter, flow id, now | calculated |  | Date as MM-DD-YYYY hh:mm:ss | low | All three reports. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Outcome to consolidated report rows |
| MR-033 | GSuite Column4..Column9 | Google id; Success/Failure; effectiveStartDate; "First Last"; primaryEmail (or emails/address on failure); aliases | conditional | primaryEmail returned |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Outcome to consolidated report rows |
| MR-034 | GSuite Column10 "Password" | (none) | hardcoded | Success rows | hardcoded default password literal [redacted] | low | Must not be carried over without a security decision. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Outcome to consolidated report rows |
| MR-035 | GSuite Column11 Department / 13 Project_ID / 14 Campaign_Name / 15 Job_Title | Google response isMailboxSetup / customerId / agreedToTerms / organizations/title | passthrough |  |  | medium | gsuite-user-lifecycle returns HCM values in these Google fields; replace with explicit fields. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Outcome to consolidated report rows |
| MR-036 | Column12 Manager_Name / Column16 Country / Column17 Location | items/ManagerName / employee Row/Column13 / location code | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Outcome to consolidated report rows |
| MR-037 | Column18 Message / Column19 Error | status text / response kind or MessageText | conditional | Success \| Failure \| Existing | "Google Account created Successfully" / "Google Account creation failed" / "AD Created Successfully for this User" / "AD Creation failed for this User" / "This user is already existing" | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Outcome to consolidated report rows |
| MR-038 | AD Column7 AD_NT_Login / Column9 Email_ID / Column11 Department / Column13 Project_ID / Column15 Job_Title | TransmissionAck attribute1 / candidateID / TransmissionMessageID / TransmissionID / SourceObjectName | passthrough |  |  | medium | Fields reused by ad-create-user to return HCM values. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Outcome to consolidated report rows |
| MR-039 | AD Column20 UCMID | TransmissionAck ReferenceValues/UserDefinedID | passthrough | Success rows |  | low | HCM username load content ID. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Outcome to consolidated report rows |
| MR-040 | AD password report Column8 "AD_NT_Password" | (none) | hardcoded | AD Success | hardcoded default password literal [redacted] | low | Security decision required. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Outcome to consolidated report rows |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | New hire selection | A new hire is any HCM person whose Person Creation Date is between the window start and end. | High | proc-sub-schedule-window gives the window; proc-sub-otbi-query runs the selection SQL. |
| BR-002 | Work email naming | Clean suffixes from names; if first name contains "ma." use ma.<Last>; if it contains a space use its first word; else <First>.<Last>; @taskus.com; remove all spaces. | High | Groovy fn-newhire-name-email. |
| BR-003 | Unprocessable names | Names containing "?" are not provisioned; IT service desk is notified with name and employee ID. | Medium | Decision shape + proc-sub-send-notification (EmailDL itservicedesk). |
| BR-004 | AD regardless of Google outcome | The AD account is created even if Google creation failed; HCM email is only updated when Google returned an address. | Medium | Sequential sub-process calls with Decision on primaryEmail. |
| BR-005 | Office location code | Worker Location Name is converted to an AD office code through the ADOfficeLocations table (lower-case match); blank if not found. | Low | Cross Reference Table ADOfficeLocations lookup in map. |
| BR-006 | Window advance | After each run (successful or not) start_date_time = start + INTERVAL + 1 second and last_scheduled_time = now. | High | proc-sub-schedule-window; failure policy pending decision. |
| BR-007 | Consolidated reporting | One row per person per report; reports emailed at the end of the run to EmailDL GSuit and AD. | Medium | Combine documents into prof-newhire-report-csv; proc-sub-file-notification x2 (password report subject to decision). |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound Direct binding (SCA) | processnewhireempreq_DirectBinding | WebLogic t3 principal used by the scheduler servlet | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/composite.xml |
| inbound SOAP 1.1 async | processnewhireempreq_sch_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/composite.xml |
| outbound Database (Oracle SOA infra) | JNDI eis/DB/SOADemoLocalTx, table XXTS_SCHEDULING | WebLogic data source | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |
| outbound SOAP (composite call) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/composite.xml |
| outbound SOAP async (composite call) | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateGSuitUser/processupdategsuituser_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/GSuit/processupdategsuituser_client_ep.wsdl |
| outbound SOAP sync (composite call) | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateEmployeeEmail/processupdateemployeeemail_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/ProcessUpdateEmployeeEmail.wsdl |
| outbound SOAP async (composite call) | http://129.144.156.28:80/soa-infra/services/default/ProcessCreateADUser/processcreateaduser_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processcreateaduser_client_ep.wsdl |
| outbound File (local SOA server) | AttachmentPath = /u01/data/domains/SOACS-Pr_domain/osi_file/EmailAttachment | n/a | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Adapters/Attachment_File_Write_file.jca |
| outbound SOAP (composite calls) | oramds:/apps/Utilities/Notification/filenotification_client_ep.wsdl and xxsendnotification_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/composite.xml |
| outbound REST (disabled) | https://ebly-dev1.hcm.us2.oraclecloud.com:443/hcmCoreApi/atomservlet/employee/newhire?updated-min&updated-max (prod plan: eetm.hcm.us6) | HTTP Authorization header from Credentials.dvm | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/WADLs/NewHireFeed.wadl |
| DB adapter (pure SQL) Select_XXTS_Scheduling | SELECT TO_CHAR(start_date_time,'yyyy-mm-dd HH24:mi:ss') start_date_time, TO_CHAR(start_date_time + ((interval)/86400),'yyyy-mm-dd HH24:mi:ss') end_date_time FROM XXTS_SCHEDULING where INTERFACE_NAME=#Interface_Name; JNDI eis/DB/SOADemoLocalTx; retry 4/1/2/120. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca |
| DB adapter (pure SQL) Update_Scheduling_StartDateTime | UPDATE xxts_scheduling SET start_date_time= to_date(TO_CHAR(#start_date_time + ((interval+1)/86400),'yyyy-mm-dd HH24:mi:ss'),'yyyy-mm-dd HH24:mi:ss') WHERE INTERFACE_NAME=#Interface_Name | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Adapters/Update_Scheduling_StartDateTime_db.jca |
| DB adapter (pure SQL) Update_XXTS_Schedule | update XXTS_SCHEDULING set last_scheduled_time =#last_scheduled_time where interface_name=#interface_name | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca |
| File adapter Attachment_File_Write | JNDI eis/FileAdapter; LogicalDirectory AttachmentPath (composite: [redacted]; Dev plan: SOACS-Te_domain); FileNamingConvention "dummy" overridden by jca.file.FileName; Append=true; NXSD FileAttachment.xsd (Column1..20, comma, double-quoted, US-ASCII). | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Adapters/Attachment_File_Write_file.jca |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| gsuite-user-lifecycle | GSuite User Lifecycle (Operation=Create) | synchronous |
| hcm-update-employee-email | Sub: HCM Update Work Email | synchronous |
| ad-create-user | AD User CREATE | synchronous |
| util-file-notification | Sub: Send File Notification | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Java_Remove_Unwanted: strips " Jr" / " II" / " III" suffixes and "Jr." from HCM names before the email address is built. | Groovy fn-newhire-name-email (clean-up + email rule). | Medium |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel | Skip_DeleteFile_Embedding: delete a file path (disabled by skipCondition). | Not needed. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_Extract_GetEmp.xsl | OTBI employee rowset to User/items record. | map-newhire-rowset-to-employee | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl | Employee to ProcessUpdateGSuitUser Create request. | map-newhire-to-gsuite-create | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_AD.xsl | Employee to ProcessCreateADUser request. | map-newhire-to-ad-create | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Process catchAll: notify EmailDL "notification", advance the schedule window (scheduled path) and call back. Per-person catchAll: notify and continue. Scoped catchAll on the AD success rows emails a hardcoded address. |
| Retries | DB adapter retries (4, backoff 2, max 120 s); no retry of business calls; failed windows are not retried. |
| Compensation | None. Partially provisioned people (e.g. Google created, AD failed) are only visible in the reports. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| EmailDL.dvm (notification, itservicedesk, GSuit, AD) | dvm | yes |
| Domain.dvm (Email path) | dvm | yes |
| ADOfficeLocations.dvm (Location -> Code, 30 rows) | dvm | yes |
| Credentials.dvm (Authorization, HCMURL - disabled calls only) | dvm | yes |
| MDS client WSDLs for ProcessUpdateGSuitUser, ProcessCreateADUser, ProcessUpdateEmployeeEmail | mds | yes |
| XXTS_SCHEDULING table rows and data | other | no |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| SOA infra database (XXTS_SCHEDULING) | JDBC | eis/DB/SOADemoLocalTx | TaskUs / OSI (SOA infrastructure) |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | AttachmentPath | [redacted] | [redacted].xml |
| DEV | AttachmentPath | [redacted] | [redacted].xml |
| PROD | NewHireFeed (disabled) | https://eetm.hcm.us6.oraclecloud.com:[redacted] | [redacted].xml |
| DEV | NewHireFeed (disabled) | https://ebly-dev1.hcm.us2.oraclecloud.com:[redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

This is TaskUs's main new-hire provisioning job. Twice a day it asks HCM which people were created since the last run, looks up each person's details and work location, and then sets up their accounts one by one: it builds a work email address from the name, creates the Google account (which also makes the address unique), records the address back in HCM, and creates the Active Directory account. At the end of the run three spreadsheets are emailed to IT: one for Google, one for AD, and one for AD that includes passwords.

Several things the code was originally built for are switched off: it does not read HCM's "new hire" event feed (it queries HCM reporting instead), it does not create TeleOpti workforce accounts, and it does not call the HCM REST API directly. These parts are not rebuilt unless TaskUs asks for them.

The main concerns are security and data handling. Two of the emailed reports contain a fixed default password and one error path emails the generated AD password; this should be redesigned rather than copied. Names are cleaned (Jr, II, III removed) only for the email address and not for the Google or AD names. The window is moved forward even when the run fails, so new hires in a failed window would not be picked up again.

In Boomi this becomes a scheduled process that calls the already-planned shared sub-processes (HCM query, Google lifecycle, HCM email update, AD create, file and email notifications) and builds the reports in memory instead of on a server folder.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | HCM Emps GET Resp JSON Prof | prof-hcm-emps-json | profile.json | — |
| 2 | HCM Atom Employee Feed Resp XML Prof | prof-hcm-atom-feed-xml | profile.xml | — |
| 3 | NewHire Consolidated Report CSV Prof | prof-newhire-report-csv | profile.flatfile | — |
| 4 | HCM Cloud REST Conn | conn-hcm-rest | connector-settings | — |
| 5 | SOAINFRA XXTS Scheduling DB Conn | conn-soainfra-db | connector-settings | — |
| 6 | HCM Atom Employee Feed GET Op | op-hcm-atom-feed-get | connector-action | conn-hcm-rest |
| 7 | HCM Emps By Query GET Op | op-hcm-emps-get | connector-action | conn-hcm-rest |
| 8 | XXTS Scheduling LastRun UPDATE Op | op-xxts-set-lastrun | connector-action | conn-soainfra-db |
| 9 | XXTS Scheduling StartDateTime UPDATE Op | op-xxts-advance-start | connector-action | conn-soainfra-db |
| 10 | XXTS Scheduling Window SELECT Op | op-xxts-select-window | connector-action | conn-soainfra-db |
| 11 | Atom Entry Content JSON Parse Script | fn-atom-content-parse | transform.function | — |
| 12 | NewHire Name Cleanup and Email Build Script | fn-newhire-name-email | transform.function | — |
| 13 | Map \| Employee to AD Create Request | map-newhire-to-ad-create | transform.map | prof-hcm-emps-json, prof-employee-ebm-xml |
| 14 | Map \| Employee to GSuite Create Request | map-newhire-to-gsuite-create | transform.map | prof-hcm-emps-json |
| 15 | Map \| OTBI NewHire Rowset to Employee Canonical | map-newhire-rowset-to-employee | transform.map | prof-otbi-rowset, prof-hcm-emps-json |
| 16 | Sub: Schedule Window Get and Advance | proc-sub-schedule-window | process | op-xxts-select-window, op-xxts-advance-start, op-xxts-set-lastrun |
| 17 | NewHire Employee Provisioning Sch | proc-newhire-emp-sch | process | proc-sub-schedule-window, proc-sub-otbi-query, fn-newhire-name-email, map-newhire-rowset-to-employee, map-newhire-to-gsuite-create, map-newhire-to-ad-create, prof-newhire-report-csv, proc-sub-send-notification, proc-sub-file-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/BPEL/ProcessNewHireEmpReq_Sch.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/ProcessNewHireEmpReq_Sch_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/ProcessNewHireEmpReq_Sch_Dev_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Adapters/Select_XXTS_Scheduling_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Adapters/Update_Scheduling_StartDateTime_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Adapters/Update_XXTS_Schedule_db.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Adapters/Attachment_File_Write_file.jca
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_Extract_GetEmp.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_gmailcreation.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_hcm_to_AD.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_GSuite_Attachment.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_AD_Attachment.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Transformations/xform_AD_PWD_Attachment.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Schemas/FileAttachment.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Schemas/GetUser_Response.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Schemas/atom_newhire_response.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/Schemas/ProcessNewHireEmpReq_Sch.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/WADLs/NewHireFeed.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/WADLs/GetEmployee.wadl
- legacy/oracle-soa/Taskus_Project/ProjectCode/Java/SchedulerServlet/ViewController/src/com/taskus/soa/scheduler/SchedulerServlet.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/Documents/TaskUs_SOA_Scheduler.xlsx
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Domain.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/ADOfficeLocations.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Credentials.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/GSuit/processupdategsuituser_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/AD/processcreateaduser_client_ep.wsdl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Provider/HCM/ProcessUpdateEmployeeEmail.wsdl

## Open questions

### Still outstanding

REVIEW-NEEDED: How should new employees receive their initial Google and Active Directory passwords, given that today a fixed default password is listed in the emailed reports? (The legacy GSuite and AD-password reports contain a hardcoded default password and are emailed to the GSuit/AD distribution lists; the report layout and the password step cannot be finalised until this is decided.)
  - If this is not answered: Remove all password columns and the password report; accounts are created with a random password that must be changed at first login, delivered through a separate process owned by IT.

REVIEW-NEEDED: If a scheduled run fails part-way, should the new hires from that time window be picked up again on the next run? (Today the window is moved forward even when the run fails, so those new hires are never provisioned automatically.)
  - If this is not answered: Do not advance the window when the run fails; the next run retries the same window and support is notified.

REVIEW-NEEDED: Should the migrated job keep using the existing XXTS_SCHEDULING table to remember where it left off, or can Boomi keep that information itself? (Keeping the table eases parallel running with the old platform; moving to Boomi removes the dependency on the SOA database.)
  - If this is not answered: Keep the XXTS_SCHEDULING table during migration and parallel run.

REVIEW-NEEDED: Should new hires also get a TeleOpti workforce account, as the original design intended? (The TeleOpti creation step is switched off in the current code, and no other live process creates TeleOpti accounts.)
  - If this is not answered: Do not create TeleOpti accounts from this job.

REVIEW-NEEDED: When a name ends in "Jr", "II" or "III", should that suffix also be removed from the Google display name and the Active Directory name, or only from the email address as today? (Today the suffix is removed only when building the email address, so names differ between the email and the accounts.)
  - If this is not answered: Keep today's behaviour: suffix removed from the email address only.

REVIEW-NEEDED: Is selecting new hires by the date the person was created in HCM still correct, or should the job use HCM's new-hire event feed or the hire date? (The original new-hire event feed call is switched off; the job uses creation date, which may include records created for other reasons.)
  - If this is not answered: Keep selecting by HCM creation date.

REVIEW-NEEDED: Who should receive the Google and AD onboarding reports and the error notices for this job? (Recipients come from the GSuit, AD, notification and itservicedesk distribution lists, and one error path uses hardcoded individual addresses that will be removed.)
  - If this is not answered: Use the current GSuit, AD, notification and itservicedesk lists as held in the production lookup table.

REVIEW-NEEDED: How many new hires are typically created in HCM per day, and on the busiest day? (Each person waits for Google and AD creation in turn; volume determines run time and whether the twice-daily schedule is sufficient.)
  - If this is not answered: Assume up to 200 new hires per run.

