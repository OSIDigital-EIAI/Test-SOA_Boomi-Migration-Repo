# Client questions — ProcessUpdateEmployeeEmail

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Is the Oracle HCM SOAP worker service (version 2) still enabled in your HCM release, or should the new process use the HCM REST API to update work emails?

**Why we are asking:** The legacy service may be deprecated; the connection cannot be built until this is confirmed.

**If this is not answered:** Keep the SOAP WorkerService mergePerson as today.

**The choices as we understand them:**

- Keep SOAP worker service
- Move to HCM REST

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · If the work email cannot be saved in HCM, should the new-hire process stop for that person or continue and report it?

**Why we are asking:** Today the failure is invisible to the new-hire job.

**If this is not answered:** Continue the new hire and report the failure to the notification list.

**The choices as we understand them:**

- Continue and report
- Stop for that person

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->
