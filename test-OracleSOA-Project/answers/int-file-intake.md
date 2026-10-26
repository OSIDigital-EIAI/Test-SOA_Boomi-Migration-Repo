# Client questions — ProcessINTReadFileABCSImpl (file poll)

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Which new-hire file layout is live: the older 113-column layout keyed on the SSS number, or the Jobvite layout keyed on the candidate ID?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Two input layouts exist. Confirm which is live. The HDL SourceSystemId scheme must not change for existing persons.

**If this is not answered:** The Employee_JobVite layout, keyed on the candidate-ID prefix.

**The choices as we understand them:**

- Employee layout (SSS key)
- Jobvite layout (candidate key)

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Which system produces the new-hire files, and where should they be dropped for Boomi (a folder on the Boomi server or an SFTP location)?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The producer of the CSV files in /u01/data/domains/SOACS-Pr_domain/Interviewed and the target drop location for Boomi are not defined in the export.

**If this is not answered:** An SFTP folder managed by TaskUs IT.

**The choices as we understand them:**

- SFTP
- Folder on the Boomi server

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Is the three-minute pause after each hire needed, or can the follow-up step run once HCM confirms the worker is created?

**Why we are asking:** The pause makes large files slow (20 hires take over an hour).

**If this is not answered:** Replace the pause with a check that the worker exists.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Who should be told when a whole file cannot be read?

**Why we are asking:** Unreadable files are sent to the "Rejection" distribution list today.

**If this is not answered:** Keep the "Rejection" list.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->
