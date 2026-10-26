# Client questions — ProcessEmployeeINTReqABCSImpl

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Which system sends new hires to the TaskUs "/emp" web service today, and is it still in use?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The caller is not identified in the export; the endpoint and its authentication must be preserved only if it is used.

**If this is not answered:** Keep the endpoint with the same request/response format.

**The choices as we understand them:**

- In use (name the system)
- No longer used

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Should one request be able to carry several hires?

**Why we are asking:** The format allows several, but only the first is processed today.

**If this is not answered:** Process every hire in the request and return one result per hire.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->
