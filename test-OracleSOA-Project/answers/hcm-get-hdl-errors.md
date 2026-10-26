# Client questions — getHDLErrorsProvABCSImpl

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Will the custom BI Publisher report "TaskUs HDL Status Report" stay in HCM and remain available to the integration user after the migration?

**Why we are asking:** The service depends entirely on this report, which is not part of the export.

**If this is not answered:** Yes, unchanged at the same path.

**The choices as we understand them:**

- Keep report as is
- Use HCM standard load status instead

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Which HCM integration user should run the HDL status report from Boomi?

**Why we are asking:** Legacy sends a stored user name and password inside the report request; the Boomi connection needs a named integration user with access to the report.

**If this is not answered:** The same HCM integration user as the other HCM connections.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->
