# Client questions — ProcessINTReadFileABCSImpl (REST CreateEmployee)

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Does any system other than the Jobvite file intake call the TaskUs "/create" hire service?

**Why we are asking:** If not, the endpoint can be dropped and the Jobvite intake can call the hire validation directly in Boomi.

**If this is not answered:** Only the Jobvite file intake uses it; no external endpoint is published.

**The choices as we understand them:**

- Only the Jobvite intake
- Other systems too

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Should the Jobvite side be told which hires were rejected, rather than always receiving "received successfully"?

**Why we are asking:** Today the reply is sent before any validation.

**If this is not answered:** Return per-hire outcomes.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->
