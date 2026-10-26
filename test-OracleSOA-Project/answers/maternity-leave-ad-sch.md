# Client questions — ProcessEmpMaternityLeave_Sch

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Should AD accounts be disabled on the first day of maternity leave and re-enabled the day after it ends?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Two defects: the end-of-leave branch iterates over the start-of-leave result, so people starting leave are immediately reactivated and people returning are never reactivated; and CN is passed as "First Last", while ad-update-ou expects the initials+EID form. Confirm the intended behaviour and the CN format.

**If this is not answered:** Disable on the start date, re-enable the day after the end date, using the CN format ad-update-ou expects.

**The choices as we understand them:**

- Disable at start, enable at end
- Disable only; enable manually
- Do not touch AD for maternity leave

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Are Maternity Caesarean, Maternity Miscarriage and Maternity Normal still the complete list of absence types that should trigger this?

**Why we are asking:** The list is hardcoded.

**If this is not answered:** Keep these three, configurable.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Who should be told when an account cannot be disabled or re-enabled?

**Why we are asking:** Failures are silently ignored today.

**If this is not answered:** Email the IT service desk list.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->
