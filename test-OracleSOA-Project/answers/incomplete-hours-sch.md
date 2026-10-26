# Client questions — ProcessIncompleteHours_Sch

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Is the weekly incomplete-hours report still required, who uses it, and where should the file go?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** PayrollWeeks.dvm covers only pay dates 2018-02-03 to 2019-01-03, so the process does nothing today. No consumer of the output CSV exists in the export. Confirm whether this report is still required, its schedule, where the file should go, and the source of payroll week dates.

**If this is not answered:** Not migrated until a consumer is identified.

**The choices as we understand them:**

- Retire it
- Keep it with a new calendar and destination

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Where should payroll-week start and end dates come from in future?

**Why we are asking:** The calendar is a fixed table that ended in January 2019 and is matched only on exact pay dates.

**If this is not answered:** A Boomi cross-reference table maintained by payroll each year.

**The choices as we understand them:**

- Maintained table
- Derive from HCM payroll calendar

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Is an incomplete week measured against 40 hours with regular hours and planned, birthday and unplanned leave counting, or as 8 hours per time entry as in the ADP transactional file?

**Why we are asking:** The two legacy jobs calculate incomplete hours differently.

**If this is not answered:** 40 hours per week with the four counted types.

**The choices as we understand them:**

- 40 hours per week
- 8 hours per time entry

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · How often should the report run?

**Why we are asking:** No schedule is in the export.

**If this is not answered:** On each pay date (3rd and 18th).

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->
