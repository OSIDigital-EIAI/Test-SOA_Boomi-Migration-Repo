# Client questions — ProcessTerminateEmpReq_Sch

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Should leaver de-provisioning cover only employees located in the Philippines, as today, or also other countries such as the US?

**Why we are asking:** The current query only selects country PH; other leavers are not de-provisioned by this job.

**If this is not answered:** Philippines only, as today.

**The choices as we understand them:**

- Philippines only
- All countries

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Should the recruiting platform (Interviewed) and TeleOpti still be told when an employee leaves, as the original design intended?

**Why we are asking:** Both steps are switched off in the current code; rebuilding them needs current endpoints and credentials.

**If this is not answered:** Do not notify Interviewed or TeleOpti.

**The choices as we understand them:**

- Neither (as today)
- TeleOpti only
- Both

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · If de-provisioning fails for one leaver, should the job continue with the others and report the failure, and should the failed time window be retried?

**Why we are asking:** Today one failure stops the remaining leavers and the window is still moved on, so they are never retried automatically.

**If this is not answered:** Continue with the remaining leavers, report each failure, and do not advance the window if the run itself fails.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Is the actual termination date the right trigger, or should access be removed on the last working day or a set time before?

**Why we are asking:** Defines when accounts are suspended relative to the employee's last day.

**If this is not answered:** Keep using the actual termination date in HCM.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · Who should receive the per-leaver "Employee Terminated" emails?

**Why we are asking:** Recipients come from the "suspended" distribution list in the lookup table.

**If this is not answered:** Use the current "suspended" distribution list from production.

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->
