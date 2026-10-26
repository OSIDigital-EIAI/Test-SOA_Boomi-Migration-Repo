# Client questions — ProcessEmpCampaignChange_Sch

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · If the daily campaign extract in HCM has not finished when this job runs, should the job try again later the same day or pick it up the next day?

**Why we are asking:** Today the job only looks for "today's" flow; if it is not completed at 15:00 UTC those campaign moves are never applied automatically.

**If this is not answered:** Retry hourly until the end of the day, then report the missed extract to the service desk.

**The choices as we understand them:**

- Retry later the same day
- Skip, as today
- Pick up on the next run

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Should a worker who appears in the extract twice in one day (for example if the job is rerun) be moved again?

**Why we are asking:** Rerunning today would repeat the same AD moves and report rows.

**If this is not answered:** Process each extract only once per day.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Who should receive the daily campaign change report and the error notices?

**Why we are asking:** Both go to the "itservicedesk" distribution list today.

**If this is not answered:** Keep sending both to the current itservicedesk list.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Roughly how many campaign changes happen on a typical day and on the busiest day?

**Why we are asking:** Each worker is processed one at a time with a wait for the AD result, so volume determines run time.

**If this is not answered:** Assume up to 500 per day.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->
