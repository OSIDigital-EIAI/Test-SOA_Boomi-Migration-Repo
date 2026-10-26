# Client questions — ProcessHCMExtractSyncProv + ProcessHCMExtractProvABCS

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Which Oracle HCM reporting (BI) service account should the migrated integrations use to query HCM data?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The current account is stored in a shared lookup table in plain text; the Boomi connection cannot be configured until a supported account is named and issued.

**If this is not answered:** Reuse the existing integration BI account with a newly rotated password stored only on the Boomi connection.

**The choices as we understand them:**

- Reuse the existing account, rotate its password
- Create a dedicated Boomi integration account in HCM

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Can any of these HCM queries return more than a few thousand rows (for example the full list of active employees), and must every row be returned?

**Why we are asking:** The current service never asks for further pages, so large results may be silently cut off; this decides whether paging must be built.

**If this is not answered:** Implement paging so that all rows are always returned.

**The choices as we understand them:**

- Yes - always return all rows
- No - results are always small, one page is enough

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · If HCM cannot be queried (for example HCM is unavailable or a query is wrong), should the calling integration stop with an error rather than continue as if no employees were found?

**Why we are asking:** One of the two current versions returns an empty answer on any error, so scheduled jobs may skip employees without anyone knowing.

**If this is not answered:** Raise the error to the calling integration, which then notifies support.

**The choices as we understand them:**

- Stop and raise an error
- Continue with no rows (as the asynchronous version does today)

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Is it acceptable for the integrations to log on to HCM reporting once per run instead of keeping a long-lived shared session?

**Why we are asking:** The current session cache has no direct equivalent in Boomi; logging on per run is simpler but adds logon calls and may hit HCM session limits.

**If this is not answered:** Keep reusing a stored session and log on again only when HCM reports it has expired, as today.

**The choices as we understand them:**

- Reuse a stored session (as today)
- Log on once per integration run

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->

### Q-005 · Does TaskUs plan to keep using the Oracle BI web services for HCM queries, or should integrations move to HCM REST APIs or BI Publisher reports where possible?

**Why we are asking:** The BI SOAP web services are an older interface; moving would change how all callers query HCM.

**If this is not answered:** Keep the BI web services (executeSQLQuery) as today.

**Answer:**

<!-- answer:Q-005 -->
<!-- /answer:Q-005 -->
