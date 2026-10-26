# Client questions — FileNotification

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Should report emails keep an empty body, or include a short standard message describing the attached report?

**Why we are asking:** Decides the content of every report email sent by the migrated integrations.

**If this is not answered:** Keep an empty body, as today; the subject and attachment carry the information.

**The choices as we understand them:**

- Empty body (as today)
- Short standard text naming the report and run time

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · If a report email cannot be sent, who should be told, and should the report be kept for re-sending?

**Why we are asking:** Today a failed send is only logged on the server and the report file is left behind; recipients never learn a report was missed.

**If this is not answered:** Log the failure, send an error notification to the "notification" distribution list, and do not retain the report.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Are there any limits on attachment size or on sending files containing personal data by email that the migrated reports must respect?

**Why we are asking:** Reports include employee identifiers and, in the current new-hire flow, passwords; mail policy may require changes to what is attached.

**If this is not answered:** Attachments are sent as today, without passwords (per the separate password-handling decision), with no size limit applied.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->
