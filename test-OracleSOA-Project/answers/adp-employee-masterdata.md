# Client questions — ProcessEmployeeADPProvABCS

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · How does the employee master-data file reach ADP today, and how should Boomi deliver it?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The FTP upload to ADP is disabled in the source; the file is only written and PGP-encrypted on the SOA host. How the files reach ADP today is unknown (manual pickup? another job?). Before building delivery, confirm the delivery method, the ADP GVS endpoint and protocol, and the PGP public key; account details will be configured directly in Boomi.

**If this is not answered:** Boomi PGP-encrypts the file and sends it by SFTP to the ADP GVS endpoint ADP specifies.

**The choices as we understand them:**

- Boomi sends to ADP by SFTP
- Someone uploads it manually
- Another tool sends it

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · For each mapping defect found (first employee's title, gender and marital status reused; TIN default 0 instead of nine zeros; dependants' birth date and gender taken from the employee; empty previous-employer client ID; missing status filter on the first employee), should Boomi fix it or reproduce today's output?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Defects in xFORM_NEWHIRE_HCM_TO_FILE.xsl and the BPEL affect payroll data. Confirm with payroll whether each should be fixed or reproduced, and validate against the ADP Philippines specification.

**If this is not answered:** Fix all of them and validate the file with ADP before go-live.

**The choices as we understand them:**

- Fix all defects
- Reproduce current output
- Decide per defect

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Are the ADP identifiers still current: client user GVETASKUS, company DMEKA, payroll area 7M, earliest start date 26 Sep 2017 and the file name PP3208_…_PHOHCM_HRMD01_DUT8G2I.SAP?

**Why we are asking:** The BEGDA floor 20170926, ABKRS 7M, header USRID GVETASKUS, CNAME DMEKA and the file name are hardcoded. Confirm the current values with ADP.

**If this is not answered:** Keep the current values as environment-configurable properties.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Who should be told when an employee cannot be included in the ADP file?

**Why we are asking:** Today a generic notification list gets an email with the subject "Test" and the raw error.

**If this is not answered:** Send a clear per-employee error summary to the payroll team list.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->
