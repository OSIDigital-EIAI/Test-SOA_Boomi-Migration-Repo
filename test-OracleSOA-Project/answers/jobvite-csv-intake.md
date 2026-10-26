# Client questions — ProcessCallJobviteRestService

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Which system or person puts the Jobvite candidate files in the folder, and where should Boomi pick them up (a folder on the Boomi server or an SFTP location)?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The file producer and the Boomi drop location are not defined in the export.

**If this is not answered:** An SFTP folder managed by TaskUs IT.

**The choices as we understand them:**

- SFTP
- Folder on the Boomi server

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · What does the "Location" value in the Jobvite file look like, and which part identifies the salary basis?

**Why we are asking:** The location is cut to its first word here but the salary-basis lookup later uses the text after the first space, so Jobvite hires may get no salary basis.

**If this is not answered:** Send the full location and let the salary map choose the part it needs.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · If one candidate in a file fails, should the remaining candidates still be processed?

**Why we are asking:** Today a failure stops the rest of the file silently.

**If this is not answered:** Yes; process the rest and report the failed rows.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->
