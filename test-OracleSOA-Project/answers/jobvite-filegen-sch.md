# Client questions — ProcessJobviteFileGen_Sch

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · How often, and at what time, should the Jobvite employee file be sent?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The schedule is held in a production table that is not in the export, and the job is not in the scheduler spreadsheet.

**If this is not answered:** Once a day at 23:00 UTC, before the Jobvite consolidated job at 23:30.

**The choices as we understand them:**

- Once a day
- Twice a day
- Other (please specify)

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Does Jobvite receive this file over FTP or SFTP, and which TaskUs team can configure the Jobvite server connection in Boomi?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The server details are configured on the legacy server, not in the export; the file contains personal data.

**If this is not answered:** SFTP, configured by TaskUs IT directly in Boomi.

**The choices as we understand them:**

- SFTP
- FTP
- FTPS

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Should the file now fill in DepartmentName, LocationName and State, which are currently sent empty?

**Why we are asking:** The department and location are already queried from HCM but not written to the file.

**If this is not answered:** Keep today's behaviour (empty).

**The choices as we understand them:**

- Keep empty
- Populate department and location

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Which countries do TaskUs employees live in today, and should any country beyond Philippines, Taiwan and the United States be sent?

**Why we are asking:** The country conversion table has only three entries; employees in other countries get an empty country.

**If this is not answered:** Keep the three existing entries and report rows with no country match.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · Should Jobvite keep receiving the full employee list on each run, or only changes since the last file?

**Why we are asking:** Today every run sends all active employees; a large file risks being cut short by the HCM query row limit.

**If this is not answered:** Full list, with paging so no rows are lost.

**The choices as we understand them:**

- Full list
- Changes only

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->

### Q-006 · Who should be told when the file cannot be produced or uploaded?

**Why we are asking:** The legacy job has no error handling, so failures go unnoticed.

**If this is not answered:** Send to the "notification" distribution list used by the other HR jobs.

**Answer:**

<!-- answer:Q-006 -->
<!-- /answer:Q-006 -->
