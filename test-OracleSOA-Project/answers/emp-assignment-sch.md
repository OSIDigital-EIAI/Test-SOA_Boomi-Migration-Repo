# Client questions — ProcessEmpAssignmentReq_Sch

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Are Google suspensions and AD moves currently happening when an employee's assignment is made inactive, reactivated or rehired in HCM?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** A defect in the current country check means these actions may not have been running; the answer sets the baseline the new build is tested against.

**If this is not answered:** Build the intended behaviour (PH employees: rehire, inactive, active and organisation changes actioned) with the country check fixed.

**The choices as we understand them:**

- Yes, they happen today
- No, they do not happen
- Not sure - verify during parallel run

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Should location, job and manager changes in HCM trigger any updates (for example in TeleOpti or AD)?

**Why we are asking:** These events are recognised but their only actions (TeleOpti updates) are switched off.

**If this is not answered:** No action for location, job or manager changes.

**The choices as we understand them:**

- No action
- Update TeleOpti
- Update AD manager/title

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Should these access changes apply only to Philippine employees, or to all countries?

**Why we are asking:** The job filters on country PH.

**If this is not answered:** Philippines only.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · When one HCM change includes both a status change and an organisation change, should both be applied?

**Why we are asking:** Today only the first matching action runs.

**If this is not answered:** Apply only the status change, as today.

**The choices as we understand them:**

- First match only
- Apply all applicable actions

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · Should a rehired employee also get a TeleOpti account again?

**Why we are asking:** The TeleOpti creation on rehire is switched off.

**If this is not answered:** No TeleOpti account on rehire.

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->
