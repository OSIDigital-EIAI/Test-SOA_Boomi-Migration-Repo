# Client questions — ProcessUpdateOUADUser

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · How should US employees be handled when they are suspended, return or change project, given that today they never get a project area in AD?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The US branch writes the OU to a path that does not exist, so US users get no OU. Confirm the intended behaviour for US users and for missing project mappings.

**If this is not answered:** Use the US OU and group tables for US employees, the same way account creation does.

**The choices as we understand them:**

- Use US tables
- Do not move US accounts
- Use the Floating area

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · When a project has no AD area configured, should the account go to the general "Floating" area or should the move stop and be reported?

**Why we are asking:** Today this case produces a malformed request that fails.

**If this is not answered:** Stop and report to the notification list.

**The choices as we understand them:**

- Move to Floating
- Stop and report

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Are TaskUs AD accounts named by full name ("First Last") or by initials plus employee number?

**Why we are asking:** Suspension, leave and return moves build the account address from initials plus employee number; accounts created by the AD creation process use the full name.

**If this is not answered:** Find every account by employee ID instead of building its address.

**The choices as we understand them:**

- Full name
- Initials + employee number
- Both exist

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · When an agent changes project but their new manager has no AD account yet, should the move still happen?

**Why we are asking:** Today nothing is changed and a failure is returned.

**If this is not answered:** Move the account and update department and groups, leave the manager unchanged and report it.

**The choices as we understand them:**

- Move, report missing manager
- Do not move (as today)

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · Who should receive the "user moved" notices and the error notices?

**Why we are asking:** Moves are emailed to the "suspended" list and errors to the "notification" list; confirm these are still right.

**If this is not answered:** Keep the "suspended" and "notification" lists.

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->
