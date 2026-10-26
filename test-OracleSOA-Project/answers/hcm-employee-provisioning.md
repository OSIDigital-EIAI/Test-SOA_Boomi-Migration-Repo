# Client questions — ProcessEmployeeHCMProvABCSImpl

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Should the recruiting systems be told whether each hire was actually created in HCM, rather than just "received"?

**Why we are asking:** Today the caller is answered before any work is done and worker creation is not awaited, so failures are invisible to the recruiting side.

**If this is not answered:** Return the real outcome (created, rejected for missing supervisor, or failed).

**The choices as we understand them:**

- Return real outcome
- Acknowledge only (as today)

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · When a hire's supervisor is not yet in HCM, should the hire be held and retried automatically, or wait for HRIS to resend it?

**Why we are asking:** Today the hire is dropped after an email to HRIS.

**If this is not answered:** Email HRIS and wait for a resend, as today.

**The choices as we understand them:**

- Email HRIS, manual resend
- Hold and retry daily

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Can the Interviewed API key that appears in the integration configuration be rotated?

**Why we are asking:** The key is visible in the exported configuration of a switched-off call and should be considered exposed.

**If this is not answered:** TaskUs rotates it before go-live.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->
