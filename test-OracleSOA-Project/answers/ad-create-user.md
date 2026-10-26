# Client questions — ProcessCreateADUser

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · How should a new employee's first AD password be set and given to them?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Today every account gets the same fixed password, which is also emailed to managers and HR, while a separate random password is logged and never used. A security decision on initial-password generation and delivery is needed before this process can be built.

**If this is not answered:** A unique random password per account, change forced at first login, delivered by a separate channel agreed with IT; no password in emails or logs.

**The choices as we understand them:**

- Unique random password, delivered separately — Change forced at first login
- Keep a common initial password — Change forced at first login; not emailed
- Self-service activation — No password sent; user activates via IT process

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Which Active Directory server should the new process connect to, over which connection type, and which IT team will set up its service account in Boomi?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The production AD host, port and protocol and the bind account are not in the export.

**If this is not answered:** LDAPS on port 636 to the production domain controller, service account configured by TaskUs IT.

**The choices as we understand them:**

- LDAPS (636)
- LDAP with StartTLS (389)

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Who should receive the "AD user created" notice for Philippine hires and for US hires?

**Why we are asking:** Hardcoded recipients (one OSI consultant for PH; five TaskUs HR/recruitment people plus the consultant for US) must be replaced by distribution lists. TaskUs to confirm the lists.

**If this is not answered:** The hiring manager plus the EmailDL "AD" list for PH, and the manager plus the EmailDL "UShrisJobvite" list for US.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · When a new hire has the same first and last name as someone already in the same AD area, how should the account be named?

**Why we are asking:** The account's directory name is "First Last" with no tie-breaker, so the second person's account cannot be created.

**If this is not answered:** Append the employee number to the display name for the second person.

**The choices as we understand them:**

- Add employee number
- Add middle initial
- Report for manual creation

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · If a returning employee still has a disabled AD account, should that account be re-enabled or should a new account be created?

**Why we are asking:** Today a disabled account is ignored and a new one is attempted, which can clash with the old account name.

**If this is not answered:** Report the case to the IT service desk and do not create a second account.

**The choices as we understand them:**

- Re-enable old account
- Create a new account
- Report to service desk

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->

### Q-006 · Should the IT service desk be emailed when an AD account cannot be created?

**Why we are asking:** The failure notices exist but are switched off, so failures are only visible in the new-hire job's reports.

**If this is not answered:** Yes, send failures to the EmailDL "itservicedesk" list.

**Answer:**

<!-- answer:Q-006 -->
<!-- /answer:Q-006 -->

### Q-007 · When a new hire's project is not in the OU and group tables, is placing the account in the general "Floating" area with no project group still correct, and should anyone be told?

**Why we are asking:** The project tables are maintained by hand; missing projects silently fall back to the Floating area.

**If this is not answered:** Keep the Floating fallback and include such accounts in the success notice.

**Answer:**

<!-- answer:Q-007 -->
<!-- /answer:Q-007 -->
