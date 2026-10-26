# Client questions — ProcessCreateEPMSUser_sch

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · How does EPMS receive the new-hire file today, given that the upload step in the integration is switched off?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** Its FTP push is disabled. Confirm the EPMS file layout (PROD vs DEV schema) and how the new-hire file is delivered to EPMS today.

**If this is not answered:** Boomi uploads the file to the EPMS server by SFTP.

**The choices as we understand them:**

- Boomi uploads by SFTP
- Someone collects it manually
- Another tool picks it up

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Should the EPMS profile file include country code, country name, legal entity and creative title?

**Why we are asking:** The Create XSL emits CountryCode, CountryName, LegalEntityName and CreativeTitle, which exist only in the DEV_MDS EPMSWrite.xsd, and omits JobCode, which exists in PROD_MDS.

**If this is not answered:** Use the PROD layout (with JobCode, without the four extra fields), as the Update job does.

**The choices as we understand them:**

- PROD layout
- DEV layout with country fields

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Which EPMS server, account and protocol should Boomi use to deliver files?

**Why we are asking:** The legacy account details are hardcoded and must be replaced; SFTP is preferred.

**If this is not answered:** SFTP to the existing server with a new service account.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->
