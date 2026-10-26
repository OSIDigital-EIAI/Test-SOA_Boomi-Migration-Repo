# Client questions — ProcessGetPersonNumber

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Should the person-number lookup only consider people created in HCM today, and if so in which time zone is "today"?

**Why we are asking:** A hire loaded on a previous day, or across a date boundary, is never found.

**If this is not answered:** Look back three days in UTC.

**The choices as we understand them:**

- Today (UTC)
- Today (Manila time)
- Last few days
- Any creation date

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · If several HCM people share the same national ID or candidate ID, which one should be used?

**Why we are asking:** Today the first row returned is used with no ordering.

**If this is not answered:** Use the most recently created person and report the duplicate.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->
