# Client questions — ProcessGetGSuitAccessToken

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Will TaskUs's Google administrators create a new OAuth client or service account for Boomi and revoke the credentials used by the current integration?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The current Google credentials are written in the integration code and must be considered exposed; the Boomi Google connection cannot be set up without new ones.

**If this is not answered:** A new service account with domain-wide delegation for the Admin SDK user and alias scopes, created by TaskUs, with the old credentials revoked at cut-over.

**The choices as we understand them:**

- Service account with delegation
- New OAuth client with refresh token

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Can the custom Java Google service on the separate server be retired once Boomi calls Google directly?

**Why we are asking:** It is only a relay to Google; other systems may also use it.

**If this is not answered:** Yes, retire it after cut-over.

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->
