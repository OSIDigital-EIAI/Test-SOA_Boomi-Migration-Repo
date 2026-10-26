# Client questions — XXSendNotification

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Which mail service and sending account should the migrated notifications be sent from?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The current Gmail account is configured inside the code; the Boomi mail connection cannot be set up without a target relay and account.

**If this is not answered:** Keep sending through Gmail SMTP (smtp.gmail.com:587) from svc.soa@taskus.com with a newly issued credential stored only on the Boomi connection.

**The choices as we understand them:**

- Keep Gmail SMTP — Same account, new credential.
- Use the corporate SMTP relay — TaskUs provides host, port and account.
- Use the Boomi platform mail service — Only suitable if external relays are not required.

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · If a notification email cannot be sent, should the calling integration be told it failed, or should the failure only be logged as today?

**Why we are asking:** Today failures are silently logged, so alerts can be lost; changing this affects how every caller handles errors.

**If this is not answered:** Log the failure against the execution and return a failure flag to the caller without stopping the caller's processing.

**The choices as we understand them:**

- Log only (as today)
- Log and return a failure flag
- Raise an error in the caller

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Should notifications keep being sent as high-priority emails?

**Why we are asking:** Every message is flagged high priority today; recipients may prefer normal priority for informational notices.

**If this is not answered:** Keep high priority on all notifications.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Is any system outside the integrations being migrated (for example a monitoring tool or script) calling the notification service directly?

**Why we are asking:** Decides whether a web service listener must be published in Boomi or an internal sub-process is enough.

**If this is not answered:** No external callers; built as an internal sub-process only.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · Roughly how many integration notification emails are sent on a normal day and on the busiest day?

**Why we are asking:** Needed to check the mail relay's sending limits.

**If this is not answered:** Assume fewer than 500 per day, within standard relay limits.

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->
