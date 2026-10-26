# Client questions — UpdateADUserName_Sch

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · If this job does not run on a given day, should that day's starters be picked up on the next run?

**Why we are asking:** The job selects only workers starting "today", so a missed day is never caught up.

**If this is not answered:** Select starters from the last successful run date up to today.

**The choices as we understand them:**

- Catch up missed days
- Today only (as today)

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Should starters whose AD account does not exist yet be reported, and to whom?

**Why we are asking:** Today they are silently skipped and never retried.

**If this is not answered:** List them in an email to the AD distribution list and retry them on the next run.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Should the job confirm with HCM that the user names were actually loaded and report any load errors?

**Why we are asking:** The current job never checks the HCM Data Loader result.

**If this is not answered:** Check the load status and report errors to the notification list.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Who should receive error notices for this job?

**Why we are asking:** Errors currently go to one consultant's personal address.

**If this is not answered:** Send to the "notification" distribution list.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->
