# Client questions — ProcessCreateGSuitUser

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · How should a new employee's first Google password be set, and must they change it at first sign-in?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Today every Google account is created with the same fixed password, the same one used for AD, and no change is forced. This needs the same security decision as the AD account password.

**If this is not answered:** Unique random password per account with change forced at first sign-in, delivered through the process agreed for AD; no password in emails.

**The choices as we understand them:**

- Random, change at first sign-in
- SSO only, no usable password
- Same process as the AD password

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Should the alias be all lower case, and what should happen if the alias address is already taken?

**Why we are asking:** Aliases are built from the preferred name and surname without checks; a clash fails silently apart from an email.

**If this is not answered:** Lower-case the alias and report clashes to the notification list.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Who should be alerted when Google refuses a new account because licences have run out?

**Why we are asking:** Licence errors are meant to go to the IT service desk list, but in production they arrive as normal replies and nobody is alerted.

**If this is not answered:** The "itservicedesk" distribution list.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Should new Google accounts be placed in a specific organisational unit rather than the top level?

**Why we are asking:** All accounts are created in the root unit "/", so Google policies per site or department cannot apply automatically.

**If this is not answered:** Keep the root unit.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->
