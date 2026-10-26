# Client questions — ProcessPBLProvABCSImpl

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Are the Philippine de minimis ceilings still Rice 1,500, Christmas 400, Uniform 400 with the remainder as Laundry, and are these the correct element names in payroll?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The split is fixed in code and directly affects pay and tax; it must be confirmed before rebuilding.

**If this is not answered:** Keep the current ceilings and names, held in a configurable table.

**The choices as we understand them:**

- Keep as today
- Provide new ceilings/names

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Should employees with no allowance or a zero allowance be skipped, rather than given an empty or zero allowance entry?

**Why we are asking:** The current check meant to skip them never works, so empty or zero entries are sent to payroll.

**If this is not answered:** Skip empty and zero amounts.

**The choices as we understand them:**

- Skip them
- Keep zero entries

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Is the employee's SSS number always the same as their HCM person number and assignment number suffix?

**Why we are asking:** The payroll file identifies employees by SSS number; if they differ, entries load against the wrong or no person.

**If this is not answered:** Yes, as today.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Should non-Philippine employees' taxable allowances also be loaded, given that every batch is submitted to the PH Legislative Data Group?

**Why we are asking:** Taxable Allowance rows are created for all countries but loaded only into the PH data group.

**If this is not answered:** Limit the service to PH employees.

**The choices as we understand them:**

- PH only
- All countries (per-country group)

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · Should the process wait for the payroll load to finish and report failures to payroll?

**Why we are asking:** Today the flow is only submitted; failed loads are not reported.

**If this is not answered:** Poll the flow status and email failures to the IT service desk list.

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->
