# Client questions — ProcessUpdateEPMSUser_sch

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Which EPMS server and transfer protocol should Boomi use to deliver the employee update file?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The EPMS FTP host is an IP address and its credentials are hardcoded in FTPFileTransfer_Class.java (redacted). Confirm the EPMS endpoint and protocol (SFTP preferred) for Boomi; the account itself will be configured directly in Boomi, not through this questionnaire.

**If this is not answered:** SFTP to the existing EPMS server, directory /HCM/PROD.

**The choices as we understand them:**

- SFTP to the same server and folder
- Keep plain FTP to the same server
- Different endpoint or method

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · How often should the EPMS update job run?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Confirm the run frequency: the spreadsheet says 23:00/11:00 UTC, while the scheduling table row says every 30 minutes between 14:00 and 23:50.

**If this is not answered:** Every 30 minutes, 14:00–23:50, as in the scheduling table.

**The choices as we understand them:**

- Every 30 minutes, 14:00–23:50
- Twice daily, 23:00 and 11:00 UTC
- Another schedule

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Should employees from the daily campaign-change process be sent to EPMS only once per day?

**Why we are asking:** Today every run whose window starts on the current date re-reads the completed campaign output, so the same people can be sent several times a day.

**If this is not answered:** Send campaign changes once per day, on the first run after the campaign flow completes.

**The choices as we understand them:**

- Once per day
- Every run, as today

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Who should be told when an individual employee cannot be sent to EPMS or the upload fails?

**Why we are asking:** Per-person errors are silently skipped today and upload failures go to two named individuals.

**If this is not answered:** Report both to the EmailDL EPMSJOB list.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->
