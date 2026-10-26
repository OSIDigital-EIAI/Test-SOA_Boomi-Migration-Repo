# Client questions — ProcessEmpUpdateReq_Sch

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · When an employee's name is changed or corrected in HCM, which systems should be updated: Google account name, Active Directory display name, TeleOpti?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Today the Google and AD updates are switched off and the TeleOpti update is switched off inside the TeleOpti service, so only aliases and trainer emails happen.

**If this is not answered:** Build as today: create Google aliases for preferred-name changes and email trainers; no Google, AD or TeleOpti renames.

**The choices as we understand them:**

- Aliases and trainer email only
- Also rename in Google and AD
- Google, AD and TeleOpti

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · How often should name changes be processed (for example hourly, daily)?

**Why we are asking:** The job is not in the scheduler spreadsheet and its schedule is not in the export.

**If this is not answered:** Run once a day at 15:00 UTC with the other employee jobs.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Are trainers currently receiving the "Employee Details Updated" emails?

**Why we are asking:** The email body format used by this job appears incompatible with the notification utility, so the emails may never have been delivered.

**If this is not answered:** Send the trainer email in the working format to the current trainers list.

**The choices as we understand them:**

- Yes, keep them
- No, and not needed
- No, but they are wanted

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Should a preferred name be added to Google as a full email address (for example preferred.lastname@taskus.com) or exactly as entered in HCM?

**Why we are asking:** This job passes the preferred name as entered; the alias service sends it to Google unchanged.

**If this is not answered:** Pass the preferred name exactly as entered, as today.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->
