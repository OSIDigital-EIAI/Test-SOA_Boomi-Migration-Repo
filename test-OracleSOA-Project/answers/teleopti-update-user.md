# Client questions — ProcessUpdateTeleOptiUser

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Should employee changes in HCM update TeleOpti automatically, and should a manager's change also update everyone reporting to them?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** All TeleOpti update calls are disabled in the source; confirm whether TeleOpti updates (and the manager cascade) should exist in the target.

**If this is not answered:** Not built.

**The choices as we understand them:**

- Update the person only
- Update person and reports
- No TeleOpti updates

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Which time zone should TeleOpti records use: Singapore (as on creation) or Western Europe (as in the update code)?

**Why we are asking:** The two services set different time zones.

**If this is not answered:** Singapore Standard Time, as on creation.

**The choices as we understand them:**

- Singapore
- W. Europe
- By work location

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Is the eligibility category spelled "Eligible for OTND" or "Eligible for OT ND" in HCM?

**Why we are asking:** Creation and update check different spellings, so one of them never matches.

**If this is not answered:** Use the value found in HCM for both.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->
