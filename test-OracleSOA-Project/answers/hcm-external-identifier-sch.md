# Client questions — ProcessUpdateExternalIdentifier

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Is the US third-party payroll ID job running in production today?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The client WSDL exists only in Utils/DEV_MDS_JobVite, the composite is not in the scheduler spreadsheet, and its XXTS_SCHEDULING row is not in the export. Confirm whether this interface is live in production.

**If this is not answered:** It is live and will be rebuilt.

**The choices as we understand them:**

- Yes, live
- No, not used

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-004 · When should the US third-party payroll ID job run each day?

**Why we are asking:** The run time is not in the scheduler spreadsheet or the export.

**If this is not answered:** Daily at 23:00 UTC, picking up all starters since the last successful run.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-002 · Are the payroll ID prefixes still MX4 (TaskUs USA LLC, Santa Monica), 6PZ (other TaskUs USA LLC) and AK0 (TaskUs Inc.), and what should happen for legal employers that match no prefix rule?

**Why we are asking:** The rules are fixed in code; workers of other legal employers get no ID.

**If this is not answered:** Keep the three rules in a maintainable table owned by US payroll.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Who should receive the success and error notices of this job?

**Why we are asking:** They go to one named person today, in a format that is probably not delivered.

**If this is not answered:** The US HRIS distribution list.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->
