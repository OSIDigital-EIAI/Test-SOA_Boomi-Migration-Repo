# Client questions — ProcessUpdateGSuitUser

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · How should new and returning employees receive their Google sign-in details, given that today passwords are sent by plain email to managers, named HR staff and the trainers list?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Passwords in email must not be carried over; the Boomi notices cannot be designed until the delivery method is agreed (same decision as for AD and Google account creation).

**If this is not answered:** No password in any email; users set their password through the agreed first-sign-in process, and notices only confirm the account.

**The choices as we understand them:**

- Confirmation only, no password
- Password via separate secure channel
- SSO, no Google password

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Which distribution lists should receive the "Google account created" notice for US and for non-US hires, instead of the named individuals in the code?

**Why we are asking:** Recipients are hardcoded people today.

**If this is not answered:** Hiring manager plus the EmailDL "GSuit" list.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · If suspending a leaver's Google account fails, who must be alerted?

**Why we are asking:** Failures are silently ignored today, so a leaver may keep access.

**If this is not answered:** Alert the IT service desk list immediately.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · For rehires, should the old Google account always be restored, or should a new account be created when the old one was deleted?

**Why we are asking:** Today a rehire with no existing account is created as new; with an account it is restored with a new password.

**If this is not answered:** Keep the current behaviour.

**The choices as we understand them:**

- Keep current behaviour
- Always create new

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->
