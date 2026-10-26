# Client questions — ProcessUpdateADUser

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · When an employee changes organisation in HCM, which details on their AD account should change: department only, department and display name, or also the OU and groups?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The live caller sends no department or display name, so it is unclear what the update is meant to achieve; OU moves are handled separately by ad-update-ou.

**If this is not answered:** Update the department from HCM only.

**The choices as we understand them:**

- Department only
- Department and display name
- Department, OU and groups

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · US users get no OU because the OU table has no US projects, so their account address falls back to the Floating area and the update will likely fail. How should US staff be handled?

**Why we are asking:** This interface uses only the Philippine OU table.

**If this is not answered:** Find the account by employee ID instead of building its address, so US and PH staff are handled the same way.

**The choices as we understand them:**

- Find account by employee ID
- Use the US OU table
- Exclude US staff

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Who should be told when an AD update succeeds or fails?

**Why we are asking:** Success notices go to one named person and failure notices are formatted so they are probably not delivered.

**If this is not answered:** No success notices; failures to the "itservicedesk" distribution list.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->
