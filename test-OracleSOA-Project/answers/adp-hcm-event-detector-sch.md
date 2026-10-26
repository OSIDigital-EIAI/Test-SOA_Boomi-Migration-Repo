# Client questions — ProcessNewHireEmpBIReq_Sch

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Which employee events must be sent to ADP — new hires, rehires, terminations, name changes, marital-status changes, transfers — and how should each be detected?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Only terminations reach ADP from this detector; new hires, rehires, name and marital-status changes are disabled. The window is never advanced, so every run re-reads the same window. Confirm which events must be sent to ADP and by which trigger. Without that, adp-employee-masterdata cannot be scoped.

**If this is not answered:** Send new hires, rehires, terminations, name and marital-status changes, detected from HCM each run, with the window advanced after each successful run.

**The choices as we understand them:**

- Terminations only, as today
- All events the job was designed for
- A different set

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · How often should employee events be sent to ADP?

**Why we are asking:** No schedule for this job is in the export.

**If this is not answered:** Once a day.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->
