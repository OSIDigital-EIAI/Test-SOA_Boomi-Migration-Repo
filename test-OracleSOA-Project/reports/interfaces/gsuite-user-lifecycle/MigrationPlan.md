# Migration Plan — ProcessUpdateGSuitUser

**Project:** Test-OracleSOA-Project  
**Source platform:** Oracle SOA  
**Generated:** 2026-10-26T02:05:07.233Z  
**Status:** DRAFT — assembled from the analysis. Nothing has been built in Boomi from this document.

## 1. Interface Metadata

| Attribute | Value |
| --- | --- |
| Interface | ProcessUpdateGSuitUser |
| Identifier | gsuite-user-lifecycle |
| Trigger | Listener |
| Starts with | Web Services Server |
| Complexity | complex (75) |
| Components to build | 5 |
| Source artefacts | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/composite.xml, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/BPEL/ProcessUpdateGSuitUser.bpel, legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateGSuitUser/SOA/BPEL/ProcessUpdateGSuitUser.bpel |
| Depends on | gsuite-get-access-token, gsuite-create-user, gsuite-email-generation, ad-create-user, hcm-otbi-sql-query, util-send-notification |

## 2. End-to-End Process Architecture

Large async BPEL composite: OTBI enrichment, token service, Java relay for get/update, async calls to ProcessEmailGenerationABCSImpl and ProcessCreateGSuitUser with callback receives, Java password generation, many notifications. Direct Google REST references remain but are skipped. The Employee copy is an older version calling Google directly.

Boomi process "GSuite User Lifecycle" (sub-process for the calling jobs): Process Call hcm-otbi-sql-query, op-google-user-get, Route on Operation to map-gsuite-update-name / map-gsuite-suspend / map-gsuite-rehire + op-google-user-update, or Process Calls to gsuite-email-generation and gsuite-create-user for Create; explicit status returned; Try/Catch with util-send-notification to EmailDL keys; no passwords in emails, logs or responses.

| Legacy construct | Boomi equivalent | Notes |
| --- | --- | --- |
| receive / callback | Process start + Return Documents |  |
| Relay getGSuiteUser / updateGSuiteUser | op-google-user-get (gsuite-email-generation) / op-google-user-update | Direct HTTPS. |
| Fault-driven not-found handling | Decision on HTTP 404 |  |
| Async ProcessEmailGenerationABCSImpl / ProcessCreateGSuitUser | Process Calls gsuite-email-generation / gsuite-create-user | Synchronous. |
| xform_suspend_gsuiteuser / xform_rehire_gsuite_user | map-gsuite-suspend / map-gsuite-rehire |  |
| PwdGenerationClass | fn-password-generator (ad-create-user) if retained |  |
| catchAll | Try/Catch + util-send-notification |  |

## 3. Process Flow Sequence

| # | Step | What happens | Source |
| --- | --- | --- | --- |
| 1 | Receive event | Async SOAP process with Operation and user data. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Schemas/ProcessUpdateGSuitUser.xsd |
| 2 | Read HCM details | If EID not empty: OTBI by Person Number; Department := s_1, ProjectID := s_4, ManagerEmail := s_8. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/BPEL/ProcessUpdateGSuitUser.bpel |
| 3 | Get access token | ProcessGetGSuitAccessToken. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/BPEL/ProcessUpdateGSuitUser.bpel |
| 4 | Look up Google user | Relay getGSuiteUser by Email; empty primaryEmail -> fault with relay error text. | legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/service/GoogleServices.java |
| 5 | Not found (Create/Rehire) | Fault handler: random password, xform_create_gsuit_user_1 -> gsuite-create-user (async); on Success email the manager/HR; set Operation=Skip. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_create_gsuit_user_1.xsl |
| 6 | Update | updateGSuiteUser with givenName/familyName; notify trainers; output from xform_update_output. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_update_output.xsl |
| 7 | Suspend / reactivate | updateGSuiteUser with suspended flag; notify EmailDL suspended. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_suspend_gsuiteuser.xsl |
| 8 | Rehire | Random password; updateGSuiteUser with name, password, suspended, changePasswordAtNextLogin from input; notify trainers and pwdtrainers (password in body). | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_rehire_gsuite_user.xsl |
| 9 | Create (user found) | If externalIds value != EID: gsuite-email-generation; if Status != Existing create as in step 5; else report existing account. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/BPEL/ProcessUpdateGSuitUser.bpel |
| 10 | Callback | Google user structure with kind/id/emails/title/orgUnitPath etc.; password included. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/BPEL/ProcessUpdateGSuitUser.bpel |

## 4. Payload Profiles

> **Not available.** Uses the Google user profile owned by gsuite-create-user; the lifecycle request is an inline schema replicating Google user fields.

## 5. Mapping Documents

| Rule | Mapping | Target field | Source | Type | Complexity |
| --- | --- | --- | --- | --- | --- |
| MR-001 | userKey / primaryEmail | Email | passthrough | Operation = Update |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/BPEL/ProcessUpdateGSuitUser.bpel | Name update |
| MR-002 | name.givenName / name.familyName | User/name/givenName, familyName | passthrough |  |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/BPEL/ProcessUpdateGSuitUser.bpel | Name update |
| MR-003 | primaryEmail | Email | passthrough | Operation = Suspend |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_suspend_gsuiteuser.xsl | Suspend / reactivate |
| MR-004 | suspended | User/suspended | passthrough |  |  | low | true = suspend, false = reactivate | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_suspend_gsuiteuser.xsl | Suspend / reactivate |
| MR-005 | primaryEmail | Google user primaryEmail | passthrough | Operation = Rehire |  | low |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_rehire_gsuite_user.xsl | Rehire restore |
| MR-006 | name, suspended, changePasswordAtNextLogin, ipWhitelisted, ims, emails, ... | User/* | conditional | present in input |  | medium |  | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_rehire_gsuite_user.xsl | Rehire restore |
| MR-007 | password | generated | calculated |  | random 12 chars | medium | Per security decision. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_rehire_gsuite_user.xsl | Rehire restore |
| MR-008 | primaryEmail, employeeID, name, AliasName, password, emails, addresses, phones | User/*, EID, Alias | passthrough | Create |  | low | primaryEmail replaced by the gsuite-email-generation result. | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_create_gsuit_user_1.xsl | Lifecycle request to create request |

## 6. Business Logic

| Rule | Name | Logic | Priority | Boomi implementation |
| --- | --- | --- | --- | --- |
| BR-001 | One account per employee | Do not create when the found account carries the same employee ID or email generation reports Existing. | High | Decision on externalIds + gsuite-email-generation status. |
| BR-002 | Suspension | suspended flag set from the caller; notify the suspended list. | High | map-gsuite-suspend + op-google-user-update; raise errors. |
| BR-003 | Rehire restore | Restore with new password and caller-supplied flags; notify trainers. | Medium | map-gsuite-rehire + op-google-user-update. |
| BR-004 | Notice routing | US new hires: manager + named HR list; others: manager + consultant; name updates and rehires: trainers; suspensions: suspended. | Medium | EmailDL cross reference keys via util-send-notification. |

## 7. Connectors and Adapters

| Kind | Detail | Authentication | Source |
| --- | --- | --- | --- |
| inbound SOAP 1.1 async (WS-Addressing callback) | http://129.144.156.28:80/soa-infra/services/default/ProcessUpdateGSuitUser/processupdategsuituser_client_ep | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/composite.xml |
| outbound REST over plain HTTP | http://129.144.156.139:9073/google-restservice-context-root/resources/googleServices/getGSuiteUser | Google bearer token as parameter | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/ProcessUpdateGSuitUser_Prod_cfgplan.xml |
| outbound REST over plain HTTP | http://129.144.156.139:9073/google-restservice-context-root/resources/googleServices/updateGSuiteUser | Google bearer token in body | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Domain.dvm |
| outbound SOAP async (composites) | oramds:/apps/Provider/GSuit/processemailgenerationabcsimplprocess_client_ep.wsdl, processcreategsuituserprocess_client_ep.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/composite.xml |
| outbound SOAP (composite) | oramds:/apps/Provider/HCM/ProcessHCMExtractSyncProv.wsdl | none declared | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/composite.xml |
| REST references GetGsuiteUser_JavaImp / UpdateGsuiteUser_JavaImp | Relay at http://129.144.156.139:[redacted] (PROD cfgplan / Domain.dvm); direct Google REST references (GetGSuitUser, UpdateGSuitUser, CreateGSuitUser, GetUser_By_ExternalID) kept but skipped. | — | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/ProcessUpdateGSuitUser_Prod_cfgplan.xml |

## 8. Sub-Process Call Hierarchy

| Interface | Name | Invocation |
| --- | --- | --- |
| gsuite-get-access-token | Google Directory connection (OAuth 2.0) | synchronous |
| gsuite-email-generation | Sub: GSuite Generate Unique Email | synchronous |
| gsuite-create-user | Sub: GSuite User CREATE | synchronous |
| ad-create-user | fn-password-generator (shared) | synchronous |
| hcm-otbi-sql-query | Sub: HCM OTBI Execute SQL | synchronous |
| util-send-notification | Sub: Send Notification | synchronous |

## 9. Custom Code

| Language | File | Purpose | Boomi recommendation | Risk |
| --- | --- | --- | --- | --- |
| java | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/SCA-INF/src/com/osi/PwdGenerationClass.java | PwdGenerationClass.randomString(12,1,1,1,1) (java.util.Random) for new-hire and rehire passwords; value written to the audit trail. | fn-password-generator (ad-create-user) only if the security decision keeps generated passwords. | High |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_suspend_gsuiteuser.xsl | Suspend/reactivate request. | map-gsuite-suspend. | Low |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_rehire_gsuite_user.xsl | Rehire restore request incl. password. | map-gsuite-rehire. | Medium |
| xslt | legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_create_gsuit_user_1.xsl | Lifecycle request to gsuite-create-user request. | Map within proc-gsuite-user-lifecycle. | Low |

## 10. Error Handling

| Concern | How it works today |
| --- | --- |
| Strategy | Faults used for control flow (not-found -> create); catchAll routes by message text and calls back; suspend errors swallowed. |
| Retries | None. |
| Compensation | None. |

## 11. MDS / Shared Artifacts

| Artefact | Kind | In the export? |
| --- | --- | --- |
| Domain.dvm (GsuiteURL) | dvm | yes |
| EmailDL.dvm | dvm | yes |

## 12. External Services

| Service | Protocol | Endpoint | Owner |
| --- | --- | --- | --- |
| Google Admin SDK Directory API | HTTPS REST (via relay) | https://www.googleapis.com/admin/directory/v1/users | Google |
| GSuiteServices relay | HTTP REST | http://129.144.156.139:9073/google-restservice-context-root/resources/googleServices | TaskUs / OSI |

## 13. Environment Configuration Matrix

| Environment | Property | Value | Source |
| --- | --- | --- | --- |
| PROD | Relay | http://129.144.156.139:[redacted] | [redacted].xml |

## 14. Boomi Migration Readiness Summary

This service is the single place where Google Workspace accounts are changed when something happens to an employee in HCM. The calling job says what to do: create, update the name, suspend or reactivate, or rehire. The service first looks up the employee's current details in HCM and finds their Google account, then acts.

For a new hire it checks that nobody already has the proposed address (and that the person does not already have an account under their employee ID), creates the account, and emails the account details to the manager and HR. For a leaver or suspension it suspends the account and tells the "suspended" list; for reactivation it lifts the suspension. For a rehire it restores the account with a newly generated password and emails that password to a trainers list.

Passwords are the main issue. New-hire emails send the same fixed password used everywhere (see gsuite-create-user and ad-create-user) to the manager and, for US hires, to six named people; rehire emails send the real new password in plain text to a distribution list; generated passwords are also written to the server log and returned to the caller. Other issues: a failed suspension is silently ignored (so a leaver may keep access without anyone knowing); success and failure are told apart by matching error message text; and all Google calls go through the custom Java relay over plain HTTP.

In Boomi this becomes a process with a Route on Operation, calling Google directly through the shared connection, with a secure password-delivery approach and explicit failure handling.

| # | Component | Identifier | Boomi type | Depends on |
| --- | --- | --- | --- | --- |
| 1 | Google User UPDATE Op | op-google-user-update | connector-action | conn-google-directory |
| 2 | Map \| Lifecycle Request to Google Name Update | map-gsuite-update-name | transform.map | prof-google-user-json |
| 3 | Map \| Lifecycle Request to Google Rehire | map-gsuite-rehire | transform.map | prof-google-user-json |
| 4 | Map \| Lifecycle Request to Google Suspend | map-gsuite-suspend | transform.map | prof-google-user-json |
| 5 | GSuite User Lifecycle | proc-gsuite-user-lifecycle | process | op-google-user-get, op-google-user-update, map-gsuite-update-name, map-gsuite-suspend, map-gsuite-rehire, fn-password-generator, proc-sub-gsuite-create-user, proc-sub-gsuite-unique-email, proc-sub-otbi-query, proc-sub-send-notification |

## 15. Appendix

- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/BPEL/ProcessUpdateGSuitUser.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/ProcessUpdateGSuitUser_Prod_cfgplan.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Schemas/ProcessUpdateGSuitUser.xsd
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_suspend_gsuiteuser.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_rehire_gsuite_user.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_create_gsuit_user_1.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessUpdateGSuitUser/SOA/Transformations/xform_update_output.xsl
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessUpdateGSuitUser/SOA/BPEL/ProcessUpdateGSuitUser.bpel
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/Domain.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Utils/PROD_MDS/apps/Utilities/DVM/EmailDL.dvm
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessNewHireEmpReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessTerminateEmpReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpAssignmentReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/Employee/ProcessEmpUpdateReq_Sch/SOA/composite.xml
- legacy/oracle-soa/Taskus_Project/ProjectCode/Java/TaskusJavaApplication/GSuiteServices/src/com/osi/service/GoogleServices.java
- legacy/oracle-soa/Taskus_Project/ProjectCode/SOA/GSuite_Java_Process/Employee/ProcessCreateGSuitUser/SOA/Transformations/xfrom_HCM_GSuite.xsl

## Open questions

### Still outstanding

REVIEW-NEEDED: How should new and returning employees receive their Google sign-in details, given that today passwords are sent by plain email to managers, named HR staff and the trainers list? (Passwords in email must not be carried over; the Boomi notices cannot be designed until the delivery method is agreed (same decision as for AD and Google account creation).)
  - If this is not answered: No password in any email; users set their password through the agreed first-sign-in process, and notices only confirm the account.

REVIEW-NEEDED: Which distribution lists should receive the "Google account created" notice for US and for non-US hires, instead of the named individuals in the code? (Recipients are hardcoded people today.)
  - If this is not answered: Hiring manager plus the EmailDL "GSuit" list.

REVIEW-NEEDED: If suspending a leaver's Google account fails, who must be alerted? (Failures are silently ignored today, so a leaver may keep access.)
  - If this is not answered: Alert the IT service desk list immediately.

REVIEW-NEEDED: For rehires, should the old Google account always be restored, or should a new account be created when the old one was deleted? (Today a rehire with no existing account is created as new; with an account it is restored with a new password.)
  - If this is not answered: Keep the current behaviour.

