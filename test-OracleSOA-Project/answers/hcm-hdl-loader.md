# Client questions — ProcessHDLProvABCSImpl

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Which HCM integration user should Boomi use to upload files and start HCM Data Loader, and does your HCM tenant accept basic [redacted] for it?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Legacy sends one stored credential as a basic [redacted] header while Oracle's service description asks for message-level security; the Boomi connections cannot be configured without this.

**If this is not answered:** A dedicated HCM integration user with basic [redacted] over HTTPS, as in legacy.

**The choices as we understand them:**

- Basic [redacted] (as today)
- WS-Security username token

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Should the new process wait for HCM Data Loader to finish and report rows that failed to load?

**Why we are asking:** Today the load is only requested; failed loads are never reported.

**If this is not answered:** Poll the load status and email failures to the IT service desk list.

**The choices as we understand them:**

- Check and report
- Submit only (as today)

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Who should be told when an HCM data load fails?

**Why we are asking:** Errors go to the "itservicedesk" list today; HRIS may be the right owner.

**If this is not answered:** Keep the "itservicedesk" list.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->
