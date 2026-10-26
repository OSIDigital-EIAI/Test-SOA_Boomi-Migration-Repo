# Client questions — ProcessEmpCampaignExtract_Sch

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Is "Camapign" (spelt this way) still the exact name of the HCM payroll flow that produces the campaign-change extract, and is "PH" the right legislative data group?

**Why we are asking:** The flow name and data group are hardcoded and must match HCM exactly or the submission fails.

**If this is not answered:** Keep flow name "Camapign" and legislative data group "PH" as in the current code.

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · If HCM rejects the daily campaign flow submission, who should be notified?

**Why we are asking:** Today a failure is silent and the follow-on AD campaign job simply finds no completed flow.

**If this is not answered:** Email the integration "notification" distribution list.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Which HCM integration account should the migrated job use to submit HCM flows?

**Why we are asking:** The current account details are held in a shared lookup table and must be configured on the Boomi connection.

**If this is not answered:** Use the existing HCM integration account with a newly rotated password.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Should the campaign extract keep running only once per day, and only for Philippine workers?

**Why we are asking:** The flow instance name is date-based, so more frequent runs would collide; the data group is PH only.

**If this is not answered:** Once per day, Philippines only.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->
