# ProcessNewHireEmpReq\_Sch

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

Twice a day this job finds the people newly created in Oracle HCM Cloud since its last run and sets up their accounts: it builds their TaskUs work email address, creates the Google Workspace account, writes the work email back to HCM, creates the Active Directory account, and then emails three consolidated reports (Google results, AD results, and AD results with passwords) to the IT distribution lists.

| Field | Value |
| --- | --- |
| Interface | ProcessNewHireEmpReq\_Sch |
| Type | Scheduled |
| Complexity | complex |
| Components to build | 17 |
| Mapping rules | 40 |
| Business rules | 7 |

## Interface Summary

This is TaskUs's main new-hire provisioning job. Twice a day it asks HCM which people were created since the last run, looks up each person's details and work location, and then sets up their accounts one by one: it builds a work email address from the name, creates the Google account (which also makes the address unique), records the address back in HCM, and creates the Active Directory account. At the end of the run three spreadsheets are emailed to IT: one for Google, one for AD, and one for AD that includes passwords.  Several things the code was originally built for are switched off: it does not read HCM's "new hire" event feed (it queries HCM reporting instead), it does not create TeleOpti workforce accounts, and it does not call the HCM REST API directly. These parts are not rebuilt unless TaskUs asks for them.  The main concerns are security and data handling. Two of the emailed reports contain a fixed default password and one error path emails the generated AD password; this should be redesigned rather than copied. Names are cleaned (Jr, II, III removed) only for the email address and not for the Google or AD names. The window is moved forward even when the run fails, so new hires in a failed window would not be picked up again.  In Boomi this becomes a scheduled process that calls the already-planned shared sub-processes (HCM query, Google lifecycle, HCM email update, AD create, file and email notifications) and builds the reports in memory instead of on a server folder.

**Complexity:** complex (score 88)

## Functional Analysis

1. **Start and read time window** — Started by the scheduler through the direct binding. Selects start\_date\_time and start\_date\_time+INTERVAL from XXTS\_SCHEDULING for INTERFACE\_NAME ProcessNewHireEmpReq\_Sch, formats both as yyyy-MM-ddTHH:mm:ss and appends .000Z / .999Z. (A manual SOAP entry uses caller-supplied updatedMin/updatedMax instead.)
2. **Find new hires in HCM** — OTBI via ProcessHCMExtractSyncProv: SET VARIABLE PARAM\_EFFECTIVE\_DATE='4000-12-31'; SELECT Effective Start Date s\_1, Person Number s\_2, Creation Date s\_3, Last Update Date s\_4, Person Start Date s\_5 FROM "Workforce Management - Person Real Time" WHERE "Person"."Creation Date" BETWEEN \<min\> AND \<max\>. Rowset parsed with ora:parseEscapedXML.
3. **Stop if nobody found** — If there are no rows (the ATOM-feed alternative is always empty because that call is skipped) the run goes straight to advancing the window.
4. **Start the three reports** — TimeStamp = now; header rows written (File adapter, append) to GSuiteFileAttachment\_\<ts\>.csv, ADFileAttachment\_\<ts\>.csv and ADFileAttachment\_PWD\_\<ts\>.csv in AttachmentPath.
5. **For each new hire (sequential)** — forEach parallel="no" over the rows; each iteration has its own catchAll that emails EmailDL "notification" with the person number and name and continues.
6. **Get person details and location** — Two OTBI calls: employee details (DOB, email, first/last/middle name, hire date, postal code, region, person number, phone, known-as, manager name, country) and location (Location Code, Worker Location Name). Location name is replaced by the ADOfficeLocations code, or blank if not found.
7. **Map to employee record** — xform\_Extract\_GetEmp maps the rowset columns to the HCM "User/items" structure; first and last name are copied to $Details.
8. **Reject names with "?"** — If first or last name contains "?", IT service desk is emailed ("User Name contains Special Charcter") and the person is skipped. Persons with an empty first name are also skipped.
9. **Clean names and build email** — Java embedding strips a trailing " Jr" from the first name, trailing " II", " III", " Jr" from the last name and any "Jr."; then the work email is built (ma./first-word/First.Last rule, @taskus.com, spaces removed).
10. **Create Google account** — xform\_hcm\_to\_gmailcreation builds a ProcessUpdateGSuitUser request with Operation=Create; the call is asynchronous and the process waits for the callback (CreateGSuitUser\_CallBack).
11. **Write work email to HCM** — If the returned primaryEmail is not empty: WorkEmail = primaryEmail and ProcessUpdateEmployeeEmail is called synchronously with PersonNumber, DateFrom = window row Effective Start Date, EmailAddress. A GSuite "Success" report row is written; otherwise a "Failure" row with the response kind.
12. **Create AD account** — xform\_hcm\_to\_AD builds the EmployeeEBM request (EmployeeID, EmailAddress, MobilePhone), first/last name from $Details; ProcessCreateADUser is called asynchronously and the process waits for its callback. Runs regardless of the Google outcome.
13. **Record AD outcome** — status Success -\> rows in the AD report and the AD password report; status Failed -\> AD report row "AD Creation failed for this User" with MessageText; otherwise -\> "This user is already existing".
14. **Email the three reports** — FileNotification called three times: GSuite report to EmailDL GSuit ("Date@\<now\>\#GSuite Consolidated Notification"), AD report to EmailDL AD ("...\#AD Consolidated Notification"), AD password report to EmailDL AD ("...\#AD Consolidated Notification with Password"); file paths from Domain.dvm Email path.
15. **Advance window and acknowledge** — Direct binding: UPDATE xxts\_scheduling start\_date\_time = start + (INTERVAL+1)/86400, set last\_scheduled\_time = now, then callback. SOAP entry: callback only.

## Technical Analysis

| Direction | Protocol | Address | Authentication |
| --- | --- | --- | --- |
| inbound | Direct binding (SCA) | processnewhireempreq\_DirectBinding | WebLogic t3 principal used by the scheduler servlet |
| inbound | SOAP 1.1 async | processnewhireempreq\_sch\_client\_ep | none declared |
| outbound | Database (Oracle SOA infra) | JNDI eis/DB/SOADemoLocalTx, table XXTS\_SCHEDULING | WebLogic data source |
| outbound | SOAP (composite call) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared |
| outbound | SOAP async (composite call) | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateGSuitUser/processupdategsuituser\_client\_ep | none declared |
| outbound | SOAP sync (composite call) | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateEmployeeEmail/processupdateemployeeemail\_client\_ep | none declared |
| outbound | SOAP async (composite call) | http://129.144.156.28:80/soa-infra/services/default/ProcessCreateADUser/processcreateaduser\_client\_ep | none declared |
| outbound | File (local SOA server) | AttachmentPath = /u01/data/domains/SOACS-Pr\_domain/osi\_file/EmailAttachment | n/a |
| outbound | SOAP (composite calls) | oramds:/apps/Utilities/Notification/filenotification\_client\_ep.wsdl and xxsendnotification\_client\_ep.wsdl | none declared |
| outbound | REST (disabled) | https://ebly-dev1.hcm.us2.oraclecloud.com:443/hcmCoreApi/atomservlet/employee/newhire?updated-min&updated-max (prod plan: eetm.hcm.us6) | HTTP Authorization header from Credentials.dvm |

## Mappings

| Rule ID | Mapping | Target field | Source | Type | Condition | Function | Complexity | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MR-001 | OTBI employee rowset to employee record | items/FirstName | Row/Column3 (Employee First Name) | passthrough | — | — | low | — |
| MR-002 | OTBI employee rowset to employee record | items/MiddleName | Row/Column8 | passthrough | — | — | low | — |
| MR-003 | OTBI employee rowset to employee record | items/LastName | Row/Column5 | passthrough | — | — | low | — |
| MR-004 | OTBI employee rowset to employee record | items/ManagerName | Row/Column12 | passthrough | — | — | low | — |
| MR-005 | OTBI employee rowset to employee record | items/PreferredName | Row/Column11 (Employee Known As) | passthrough | — | — | low | Becomes the Google alias. |
| MR-006 | OTBI employee rowset to employee record | items/PersonNumber | Row/Column9 | passthrough | — | — | low | — |
| MR-007 | OTBI employee rowset to employee record | items/WorkPhoneNumber and WorkMobilePhoneNumber | Row/Column10 (Primary Phones Number) | passthrough | — | — | low | Same value in both. |
| MR-008 | OTBI employee rowset to employee record | items/WorkEmail | Row/Column2 (Employee Email Address) | passthrough | — | — | low | Overwritten later by the generated address. |
| MR-009 | OTBI employee rowset to employee record | items/Region, PostalCode | Row/Column7, Row/Column6 | passthrough | — | — | low | — |
| MR-010 | OTBI employee rowset to employee record | items/DateOfBirth, HireDate | Row/Column1, Row/Column4 | passthrough | — | — | low | Column13 (Country) is not mapped; used directly in reports. |
| MR-011 | Name clean-up and work email | items/FirstName (cleaned) | items/FirstName | transformed | trailing " Jr" | remove trailing " Jr"; remove "Jr."; trim | low | Applied after $Details was copied, so only the email uses cleaned names. |
| MR-012 | Name clean-up and work email | items/LastName (cleaned) | items/LastName | transformed | trailing " II", " III", " Jr" | remove suffix; remove "Jr."; trim | low | — |
| MR-013 | Name clean-up and work email | items/WorkEmail | cleaned FirstName, LastName | conditional | lower(FirstName) contains "ma." \| FirstName contains " " \| otherwise | "ma."+Last \| first word+"."+Last \| First+"."+Last, + "@taskus.com", all spaces removed | medium | Uniqueness is handled by gsuite-user-lifecycle (email generation). |
| MR-014 | Employee to Google create request | Email, User/primaryEmail, User/emails/address | items/WorkEmail | passthrough | — | — | low | — |
| MR-015 | Employee to Google create request | Operation | (none) | hardcoded | — | "Create" | low | — |
| MR-016 | Employee to Google create request | EID | items/PersonNumber | passthrough | — | — | low | — |
| MR-017 | Employee to Google create request | Alias | items/PreferredName | passthrough | — | — | low | — |
| MR-018 | Employee to Google create request | User/name/givenName | $Details/firstName | passthrough | — | — | low | Uncleaned name. |
| MR-019 | Employee to Google create request | User/name/familyName | $Details/lastName | transformed | — | translate(lastName," ","") | low | Uncleaned, spaces removed. |
| MR-020 | Employee to Google create request | User/password | (none) | hardcoded | — | literal "" (two quote characters) | low | The callee generates the password. |
| MR-021 | Employee to Google create request | User/hashFunction | (none) | hardcoded | — | "false" | low | — |
| MR-022 | Employee to Google create request | User/addresses/streetAddress, region, postalCode | items/AddressLine1, Region, PostalCode | passthrough | — | — | low | AddressLine1 is never populated by MAP-001. |
| MR-023 | Employee to Google create request | User/organizations/name, title | (none) | hardcoded | — | "taskus.com", "SWE" | low | — |
| MR-024 | Employee to Google create request | User/phones/value | items/WorkMobilePhoneNumber | passthrough | — | — | low | — |
| MR-025 | Employee to HCM work email update | PersonNumber | items/PersonNumber | passthrough | primaryEmail returned | — | low | — |
| MR-026 | Employee to HCM work email update | DateFrom | new-hire Row/Column1 (Effective Start Date) | passthrough | — | — | low | — |
| MR-027 | Employee to HCM work email update | EmailAddress | Google response primaryEmail | passthrough | — | — | low | — |
| MR-028 | Employee to AD create request | Employee/EmployeeID | items/PersonNumber | passthrough | — | — | low | — |
| MR-029 | Employee to AD create request | Employee/Person/EmailAddress | items/WorkEmail | passthrough | — | — | low | Google primaryEmail when Google succeeded, else the generated base address. |
| MR-030 | Employee to AD create request | Employee/EmployeePSFTType/MobilePhone | items/WorkPhoneNumber | passthrough | — | — | low | — |
| MR-031 | Employee to AD create request | Employee/Person/FirstName, LastName | $Details/firstName, lastName | passthrough | — | — | low | Uncleaned names. |
| MR-032 | Outcome to consolidated report rows | Column1 Sl\_No / Column2 Instance\_ID / Column3 Date | loop counter, flow id, now | calculated | — | Date as MM-DD-YYYY hh:mm:ss | low | All three reports. |
| MR-033 | Outcome to consolidated report rows | GSuite Column4..Column9 | Google id; Success/Failure; effectiveStartDate; "First Last"; primaryEmail (or emails/address on failure); aliases | conditional | primaryEmail returned | — | low | — |
| MR-034 | Outcome to consolidated report rows | GSuite Column10 "Password" | (none) | hardcoded | Success rows | hardcoded default password literal \[redacted\] | low | Must not be carried over without a security decision. |
| MR-035 | Outcome to consolidated report rows | GSuite Column11 Department / 13 Project\_ID / 14 Campaign\_Name / 15 Job\_Title | Google response isMailboxSetup / customerId / agreedToTerms / organizations/title | passthrough | — | — | medium | gsuite-user-lifecycle returns HCM values in these Google fields; replace with explicit fields. |
| MR-036 | Outcome to consolidated report rows | Column12 Manager\_Name / Column16 Country / Column17 Location | items/ManagerName / employee Row/Column13 / location code | passthrough | — | — | low | — |
| MR-037 | Outcome to consolidated report rows | Column18 Message / Column19 Error | status text / response kind or MessageText | conditional | Success \| Failure \| Existing | "Google Account created Successfully" / "Google Account creation failed" / "AD Created Successfully for this User" / "AD Creation failed for this User" / "This user is already existing" | low | — |
| MR-038 | Outcome to consolidated report rows | AD Column7 AD\_NT\_Login / Column9 Email\_ID / Column11 Department / Column13 Project\_ID / Column15 Job\_Title | TransmissionAck attribute1 / candidateID / TransmissionMessageID / TransmissionID / SourceObjectName | passthrough | — | — | medium | Fields reused by ad-create-user to return HCM values. |
| MR-039 | Outcome to consolidated report rows | AD Column20 UCMID | TransmissionAck ReferenceValues/UserDefinedID | passthrough | Success rows | — | low | HCM username load content ID. |
| MR-040 | Outcome to consolidated report rows | AD password report Column8 "AD\_NT\_Password" | (none) | hardcoded | AD Success | hardcoded default password literal \[redacted\] | low | Security decision required. |

_The same rows are in `Mappings.csv`, which opens in Excel._

## Business Rules

| ID | Rule | Logic | Priority |
| --- | --- | --- | --- |
| BR-001 | New hire selection | A new hire is any HCM person whose Person Creation Date is between the window start and end. | High |
| BR-002 | Work email naming | Clean suffixes from names; if first name contains "ma." use ma.\<Last\>; if it contains a space use its first word; else \<First\>.\<Last\>; @taskus.com; remove all spaces. | High |
| BR-003 | Unprocessable names | Names containing "?" are not provisioned; IT service desk is notified with name and employee ID. | Medium |
| BR-004 | AD regardless of Google outcome | The AD account is created even if Google creation failed; HCM email is only updated when Google returned an address. | Medium |
| BR-005 | Office location code | Worker Location Name is converted to an AD office code through the ADOfficeLocations table (lower-case match); blank if not found. | Low |
| BR-006 | Window advance | After each run (successful or not) start\_date\_time = start + INTERVAL + 1 second and last\_scheduled\_time = now. | High |
| BR-007 | Consolidated reporting | One row per person per report; reports emailed at the end of the run to EmailDL GSuit and AD. | Medium |

## Flow Diagram

![Flow diagram](flow.svg)

1. **Scheduled start** — Direct binding from SchedulerServlet; 09:00 / 21:00 UTC
2. **Read XXTS window** — Select\_XXTS\_Scheduling; format + .000Z/.999Z
3. **Query new hires (OTBI)** — Person Real Time, Creation Date BETWEEN window
4. **Any new hires?** — count(Row) \> 0
5. **Notify error, advance window** — Process catchAll: notification, then window still advanced
6. **Start three reports** — Header rows GSuite / AD / AD\_PWD CSV
7. **Next new hire** — forEach parallel=no; per-person catchAll notifies and continues
8. **Get details and location** — Two OTBI queries; ADOfficeLocations code
9. **Name contains "?"** — Yes -\> notify itservicedesk, skip
10. **Notify special characters** — XXSendNotification itservicedesk
11. **Clean names, build email** — Java clean-up + ma./first word/First.Last rule
12. **Create Google account** — ProcessUpdateGSuitUser Operation=Create, wait for callback
13. **Google address returned?** — primaryEmail != ""
14. **Update HCM work email** — ProcessUpdateEmployeeEmail
15. **Create AD account** — ProcessCreateADUser, wait for callback
16. **Write report rows** — GSuite row; AD Success/Failed/Existing rows; AD password row on success
17. **More new hires?** — Loop back while counter \<= count(Row)
18. **Email consolidated reports** — FileNotification x3 to EmailDL GSuit / AD
19. **Advance window, callback** — Update start\_date\_time and last\_scheduled\_time

## Data Model

| Profile | Format | Component |
| --- | --- | --- |
| HCM Emps GET Resp JSON Prof | json | prof-hcm-emps-json |
| NewHire Consolidated Report CSV Prof | flatfile | prof-newhire-report-csv |
| HCM Atom Employee Feed Resp XML Prof | xml | prof-hcm-atom-feed-xml |

## Architecture

**How the legacy pieces map to Boomi**

| Legacy | Boomi |
| --- | --- |
| SchedulerServlet + direct binding | Boomi process schedule (09:00, 21:00 UTC) |
| Select/Update XXTS\_SCHEDULING DB adapters | conn-soainfra-db + op-xxts-\* in proc-sub-schedule-window (or persisted property) |
| ProcessHCMExtractSyncProv calls (x1 + x2 per person) | Process Call to hcm-otbi-sql-query sub-process |
| xform\_Extract\_GetEmp.xsl | map-newhire-rowset-to-employee |
| Java\_Remove\_Unwanted + email assigns | fn-newhire-name-email (Groovy) |
| xform\_hcm\_to\_gmailcreation.xsl + async ProcessUpdateGSuitUser | map-newhire-to-gsuite-create + Process Call gsuite-user-lifecycle |
| ProcessUpdateEmployeeEmail | Process Call hcm-update-employee-email |
| xform\_hcm\_to\_AD.xsl + async ProcessCreateADUser | map-newhire-to-ad-create + Process Call ad-create-user |
| File adapter appends to /u01/.../EmailAttachment + FileNotification | prof-newhire-report-csv documents + Combine + util-file-notification |
| catchAll blocks + XXSendNotification | Try/Catch + util-send-notification |
| Disabled ATOM / HCM REST / TeleOpti references | Not built for this process (ATOM and emps operations owned here for other schedulers) |

**Components**

| Component | Role |
| --- | --- |
| conn-soainfra-db | DB connection to SOA infra (XXTS\_SCHEDULING) |
| op-xxts-select-window | Read window |
| op-xxts-advance-start | Advance start\_date\_time |
| op-xxts-set-lastrun | Set last\_scheduled\_time |
| proc-sub-schedule-window | Shared window sub-process |
| conn-hcm-rest | HCM REST/ATOM connection |
| op-hcm-atom-feed-get | ATOM feed GET |
| prof-hcm-atom-feed-xml | ATOM feed profile |
| fn-atom-content-parse | ATOM content JSON parse |
| op-hcm-emps-get | HCM emps GET |
| prof-hcm-emps-json | Employee record (User/items) |
| fn-newhire-name-email | Name clean-up and email rule |
| map-newhire-rowset-to-employee | OTBI rowset to employee |
| map-newhire-to-gsuite-create | Google create request |
| map-newhire-to-ad-create | AD create request |
| prof-newhire-report-csv | Report CSV profile |
| proc-newhire-emp-sch | Main scheduled process |

## Client questions

- **Q-001** How should new employees receive their initial Google and Active Directory passwords, given that today a fixed default password is listed in the emailed reports?
  **If this is not answered:** Remove all password columns and the password report; accounts are created with a random password that must be changed at first login, delivered through a separate process owned by IT.
- **Q-002** If a scheduled run fails part-way, should the new hires from that time window be picked up again on the next run?
  **If this is not answered:** Do not advance the window when the run fails; the next run retries the same window and support is notified.
- **Q-003** Should the migrated job keep using the existing XXTS\_SCHEDULING table to remember where it left off, or can Boomi keep that information itself?
  **If this is not answered:** Keep the XXTS\_SCHEDULING table during migration and parallel run.
- **Q-004** Should new hires also get a TeleOpti workforce account, as the original design intended?
  **If this is not answered:** Do not create TeleOpti accounts from this job.
- **Q-005** When a name ends in "Jr", "II" or "III", should that suffix also be removed from the Google display name and the Active Directory name, or only from the email address as today?
  **If this is not answered:** Keep today's behaviour: suffix removed from the email address only.
- **Q-006** Is selecting new hires by the date the person was created in HCM still correct, or should the job use HCM's new-hire event feed or the hire date?
  **If this is not answered:** Keep selecting by HCM creation date.
- **Q-007** Who should receive the Google and AD onboarding reports and the error notices for this job?
  **If this is not answered:** Use the current GSuit, AD, notification and itservicedesk lists as held in the production lookup table.
- **Q-008** How many new hires are typically created in HCM per day, and on the busiest day?
  **If this is not answered:** Assume up to 200 new hires per run.
