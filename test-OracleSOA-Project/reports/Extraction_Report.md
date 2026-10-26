# Test-OracleSOA-Project

Every interface found in the oracle-soa export, and what the analysis was able to establish about each one.

**45 of 45** interfaces have a full report.

| # | Interface | Type | Complexity | Components |
| --- | --- | --- | --- | --- |
| 1 | [XXSendNotification](interfaces/util-send-notification/Interface_Report.md) | listener | simple | 5 |
| 2 | [FileNotification](interfaces/util-file-notification/Interface_Report.md) | listener | simple | 2 |
| 3 | [ProcessHCMExtractSyncProv + ProcessHCMExtractProvABCS](interfaces/hcm-otbi-sql-query/Interface_Report.md) | listener | medium | 7 |
| 4 | [ProcessNewHireEmpReq\_Sch](interfaces/newhire-emp-sch/Interface_Report.md) | scheduled | complex | 17 |
| 5 | [ProcessTerminateEmpReq\_Sch](interfaces/terminate-emp-sch/Interface_Report.md) | scheduled | medium | 3 |
| 6 | [ProcessEmpUpdateReq\_Sch](interfaces/emp-update-sch/Interface_Report.md) | scheduled | medium | 3 |
| 7 | [ProcessEmpAssignmentReq\_Sch](interfaces/emp-assignment-sch/Interface_Report.md) | scheduled | complex | 6 |
| 8 | [ProcessEmpCampaignExtract\_Sch](interfaces/emp-campaign-extract-sch/Interface_Report.md) | scheduled | simple | 6 |
| 9 | [ProcessEmpCampaignChange\_Sch](interfaces/emp-campaign-change-sch/Interface_Report.md) | scheduled | medium | 7 |
| 10 | [UpdateADUserName\_Sch](interfaces/update-ad-username-sch/Interface_Report.md) | scheduled | medium | 1 |
| 11 | [ProcessJobviteFileGen\_Sch](interfaces/jobvite-filegen-sch/Interface_Report.md) | scheduled | simple | 5 |
| 12 | [ProcessCreateADUser](interfaces/ad-create-user/Interface_Report.md) | listener | complex | 8 |
| 13 | [ProcessUpdateADUser](interfaces/ad-update-user/Interface_Report.md) | listener | simple | 1 |
| 14 | [ProcessUpdateOUADUser](interfaces/ad-update-ou/Interface_Report.md) | listener | complex | 4 |
| 15 | [ProcessRemoveOldADGroups](interfaces/ad-remove-old-groups/Interface_Report.md) | listener | simple | 1 |
| 16 | [ProcessHDLProvABCSImpl](interfaces/hcm-hdl-loader/Interface_Report.md) | listener | medium | 8 |
| 17 | [ProcessCreateUserDAT](interfaces/hcm-create-user-dat/Interface_Report.md) | listener | simple | 3 |
| 18 | [ProcessUpdateEmployeeEmail](interfaces/hcm-update-employee-email/Interface_Report.md) | listener | simple | 4 |
| 19 | [ProcessGetPersonNumber](interfaces/hcm-get-person-number/Interface_Report.md) | listener | simple | 1 |
| 20 | [getHDLErrorsProvABCSImpl](interfaces/hcm-get-hdl-errors/Interface_Report.md) | listener | simple | 5 |
| 21 | [ProcessPBLProvABCSImpl](interfaces/hcm-pbl-allowances/Interface_Report.md) | listener | medium | 4 |
| 22 | [ProcessGetGSuitAccessToken](interfaces/gsuite-get-access-token/Interface_Report.md) | listener | simple | 1 |
| 23 | [ProcessCreateGSuitUser](interfaces/gsuite-create-user/Interface_Report.md) | listener | medium | 6 |
| 24 | [ProcessEmailGenerationABCSImpl](interfaces/gsuite-email-generation/Interface_Report.md) | listener | simple | 2 |
| 25 | [ProcessCreateGSuiteAlias](interfaces/gsuite-create-alias/Interface_Report.md) | listener | simple | 1 |
| 26 | [ProcessUpdateGSuitUser](interfaces/gsuite-user-lifecycle/Interface_Report.md) | listener | complex | 5 |
| 27 | [ProcessCreateTeleOptiUser](interfaces/teleopti-create-user/Interface_Report.md) | listener | medium | 6 |
| 28 | [ProcessUpdateTeleOptiUser](interfaces/teleopti-update-user/Interface_Report.md) | listener | medium | 3 |
| 29 | [ProcessDATGenerationABCSImpl](interfaces/hcm-dat-generation/Interface_Report.md) | listener | complex | 7 |
| 30 | [ProcessEmployeeHCMProvABCSImpl](interfaces/hcm-employee-provisioning/Interface_Report.md) | listener | medium | 3 |
| 31 | [ProcessEmployeeINTReqABCSImpl](interfaces/emp-int-rest-intake/Interface_Report.md) | listener | simple | 4 |
| 32 | [ProcessINTReadFileABCSImpl (file poll)](interfaces/int-file-intake/Interface_Report.md) | scheduled | complex | 6 |
| 33 | [ProcessINTReadFileABCSImpl (REST CreateEmployee)](interfaces/jobvite-create-employee-rest/Interface_Report.md) | listener | medium | 2 |
| 34 | [ProcessDatProfileGeneration (REST JobVite\_Rest)](interfaces/dat-profile-generation-rest/Interface_Report.md) | listener | complex | 15 |
| 35 | [ProcessDatProfileGeneration (archive file poll)](interfaces/dat-profile-archive-poll/Interface_Report.md) | scheduled | simple | 2 |
| 36 | [ProcessCallJobviteRestService](interfaces/jobvite-csv-intake/Interface_Report.md) | scheduled | simple | 4 |
| 37 | [ProcessJobviteConsolidatedNotification\_Sch](interfaces/jobvite-consolidated-notification-sch/Interface_Report.md) | scheduled | simple | 3 |
| 38 | [ProcessUpdateExternalIdentifier](interfaces/hcm-external-identifier-sch/Interface_Report.md) | scheduled | medium | 3 |
| 39 | [ProcessCreateEPMSUser\_sch](interfaces/epms-create-user-sch/Interface_Report.md) | scheduled | medium | 5 |
| 40 | [ProcessUpdateEPMSUser\_sch](interfaces/epms-update-user-sch/Interface_Report.md) | scheduled | complex | 3 |
| 41 | [ProcessEmployeeADPProvABCS](interfaces/adp-employee-masterdata/Interface_Report.md) | listener | complex | 6 |
| 42 | [ProcessADPTransProvABCS](interfaces/adp-transactional-file-sch/Interface_Report.md) | scheduled | complex | 4 |
| 43 | [ProcessNewHireEmpBIReq\_Sch](interfaces/adp-hcm-event-detector-sch/Interface_Report.md) | scheduled | medium | 2 |
| 44 | [ProcessIncompleteHours\_Sch](interfaces/incomplete-hours-sch/Interface_Report.md) | scheduled | medium | 4 |
| 45 | [ProcessEmpMaternityLeave\_Sch](interfaces/maternity-leave-ad-sch/Interface_Report.md) | scheduled | simple | 1 |

---

_Interfaces without a report are listed above with the reason. They are not omitted: an interface missing from this list would read as one that does not exist._
