# Client questions — ProcessCreateTeleOptiUser

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Should new agents still be created automatically in TeleOpti?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** No live caller invokes this service: both calls (from newhire-emp-sch and emp-assignment-sch) are disabled. Confirm whether TeleOpti provisioning is still required.

**If this is not answered:** Out of scope; not built.

**The choices as we understand them:**

- Yes, build it
- No, retire it

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · How should an agent's TeleOpti business unit be chosen?

**Why we are asking:** The business-unit lookup passes a fixed text instead of the project, so the business unit is always empty; the correct BU mapping must be confirmed.

**If this is not answered:** Look up the TeleOpti business unit by the HCM project description.

**The choices as we understand them:**

- By project description
- By project ID

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · What should the TeleOpti login name be, given that first names are not unique?

**Why we are asking:** Today the login name is the first name only.

**If this is not answered:** Use the AD account name (initials + employee number).

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Which TeleOpti service account should the new process use?

**Why we are asking:** The current account details are stored in a data table in the integration platform and must move to the Boomi connection.

**If this is not answered:** TaskUs creates a dedicated integration account in TeleOpti.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->
