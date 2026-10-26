# Client questions — ProcessDatProfileGeneration (REST JobVite_Rest)

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Which version of the new-hire follow-up service is running in production: the Jobvite version, or the version that also creates Google accounts and updates the work email?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Four variants exist. The GSuite variant additionally creates Google accounts here and updates the HCM work email; if that variant is live, Google accounts for recruited hires are created in this flow and not only in newhire-emp-sch. Confirm which variant is deployed.

**If this is not answered:** The Employee_JobVite version (no Google creation here).

**The choices as we understand them:**

- Jobvite version
- Version with Google creation

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Where should the day's hire results be kept for the 23:30 summary job once the server files go away?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Legacy appends consolidated rows to JobviteConsolidate_PH.csv / _US.csv on the server for the 23:30 job to pick up. Decide the Boomi staging mechanism (a database table, or a document cache plus the day's executions).

**If this is not answered:** A small database table, written per hire and read and cleared at 23:30.

**The choices as we understand them:**

- Database table
- Boomi document cache

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Can TaskUs issue new Jobvite API credentials for Boomi and revoke the ones in the current integration?

**Why we are asking:** The Jobvite api and sc keys are embedded in the reference URL in composite.xml (redacted) and must be rotated and re-issued for Boomi.

**If this is not answered:** New keys issued and stored in the Boomi Jobvite connection.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Should the HR hire lists go only to the team for the hire's country?

**Why we are asking:** The legacy country condition also sends to both PH and US lists whenever both Country and CountryCode are populated, which is probably a defect; confirm the routing.

**If this is not answered:** Send only to the hire's country team.

**The choices as we understand them:**

- Country team only
- Both teams (as today)

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · Does the detailed HR list need salary, visa and tax ID data, given that it is emailed to a distribution list?

**Why we are asking:** The list contains every field of the hire record.

**If this is not answered:** Reduce the list to identification, job and status fields.

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->

### Q-006 · Should allowances be loaded using the HCM person number rather than the SSS number?

**Why we are asking:** The payroll allowance load identifies employees by SSS number, while this service already knows the real person number.

**If this is not answered:** Use the HCM person number.

**Answer:**

<!-- answer:Q-006 -->
<!-- /answer:Q-006 -->
