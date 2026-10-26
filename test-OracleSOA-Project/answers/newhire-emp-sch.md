# Client questions — ProcessNewHireEmpReq_Sch

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · How should new employees receive their initial Google and Active Directory passwords, given that today a fixed default password is listed in the emailed reports?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The legacy GSuite and AD-password reports contain a hardcoded default password and are emailed to the GSuit/AD distribution lists; the report layout and the password step cannot be finalised until this is decided.

**If this is not answered:** Remove all password columns and the password report; accounts are created with a random password that must be changed at first login, delivered through a separate process owned by IT.

**The choices as we understand them:**

- Remove passwords from reports; separate secure delivery
- Keep a password report but send it only to a restricted list
- Send each password individually to the hiring manager

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · If a scheduled run fails part-way, should the new hires from that time window be picked up again on the next run?

**Why we are asking:** Today the window is moved forward even when the run fails, so those new hires are never provisioned automatically.

**If this is not answered:** Do not advance the window when the run fails; the next run retries the same window and support is notified.

**The choices as we understand them:**

- Retry the window on the next run
- Skip it and rely on the error email (as today)

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Should the migrated job keep using the existing XXTS_SCHEDULING table to remember where it left off, or can Boomi keep that information itself?

**Why we are asking:** Keeping the table eases parallel running with the old platform; moving to Boomi removes the dependency on the SOA database.

**If this is not answered:** Keep the XXTS_SCHEDULING table during migration and parallel run.

**The choices as we understand them:**

- Keep XXTS_SCHEDULING
- Store the last run time in Boomi

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Should new hires also get a TeleOpti workforce account, as the original design intended?

**Why we are asking:** The TeleOpti creation step is switched off in the current code, and no other live process creates TeleOpti accounts.

**If this is not answered:** Do not create TeleOpti accounts from this job.

**The choices as we understand them:**

- No (as today)
- Yes, create TeleOpti accounts for eligible new hires

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · When a name ends in "Jr", "II" or "III", should that suffix also be removed from the Google display name and the Active Directory name, or only from the email address as today?

**Why we are asking:** Today the suffix is removed only when building the email address, so names differ between the email and the accounts.

**If this is not answered:** Keep today's behaviour: suffix removed from the email address only.

**The choices as we understand them:**

- Email address only (as today)
- Remove from all account names

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->

### Q-006 · Is selecting new hires by the date the person was created in HCM still correct, or should the job use HCM's new-hire event feed or the hire date?

**Why we are asking:** The original new-hire event feed call is switched off; the job uses creation date, which may include records created for other reasons.

**If this is not answered:** Keep selecting by HCM creation date.

**Answer:**

<!-- answer:Q-006 -->
<!-- /answer:Q-006 -->

### Q-007 · Who should receive the Google and AD onboarding reports and the error notices for this job?

**Why we are asking:** Recipients come from the GSuit, AD, notification and itservicedesk distribution lists, and one error path uses hardcoded individual addresses that will be removed.

**If this is not answered:** Use the current GSuit, AD, notification and itservicedesk lists as held in the production lookup table.

**Answer:**

<!-- answer:Q-007 -->
<!-- /answer:Q-007 -->

### Q-008 · How many new hires are typically created in HCM per day, and on the busiest day?

**Why we are asking:** Each person waits for Google and AD creation in turn; volume determines run time and whether the twice-daily schedule is sufficient.

**If this is not answered:** Assume up to 200 new hires per run.

**Answer:**

<!-- answer:Q-008 -->
<!-- /answer:Q-008 -->
