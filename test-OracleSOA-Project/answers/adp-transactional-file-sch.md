# Client questions — ProcessADPTransProvABCS

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Where should the payroll-week dates for incomplete hours come from, and which rule applies: 8 hours per time entry or 40 hours per week?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** As exported this interface cannot produce correct incomplete-hours records: PayrollWeeks.dvm has no weekOne/weekTwo columns and no key 2018-03-01, the week-2 range end uses weekOne, and the window update passes null values. Confirm the payroll-week calendar source, which incomplete-hours rule applies (8 h per entry here vs 40 h per week in incomplete-hours-sch), and the run schedule.

**If this is not answered:** Use the PayrollWeeks table keyed by the current payroll period start and 8 hours per time entry, as coded.

**The choices as we understand them:**

- 8 hours per time entry
- 40 hours per week
- Another rule

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · How often and at what time should the ADP transactional file be produced?

**Why we are asking:** No schedule for this job is in the export.

**If this is not answered:** Once per payroll cut-off, triggered manually until a schedule is agreed.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Should the transactional file be delivered to ADP the same way as the master-data file?

**Why we are asking:** The upload is disabled here as well; delivery is decided under adp-employee-masterdata.

**If this is not answered:** Same PGP + SFTP delivery as the master-data file.

**The choices as we understand them:**

- Same as master data
- Different method

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Are the wage codes in the recurring (P0014) and one-off (P0015) lists still complete and correct?

**Why we are asking:** The code lists are hardcoded in the map; any new pay element would be silently omitted.

**If this is not answered:** Keep the current lists in a configurable cross-reference table.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->
