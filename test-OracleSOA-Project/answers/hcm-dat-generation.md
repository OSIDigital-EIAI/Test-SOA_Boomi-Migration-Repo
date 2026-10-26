# Client questions — ProcessDATGenerationABCSImpl

Click the pencil at the top right to edit this file, type each answer **between the two
comment lines** under its question, then choose **Propose changes**. That opens a pull
request we review — you do not need to know git, and nothing you write takes effect until
we have read it.

Leave a question alone if you do not know the answer; blank means still outstanding, not
declined. Type `[not applicable]` on its own line to say a question does not apply.

---
### Q-001 · Where should the link between each candidate and their HCM load be kept, now that the legacy SOA database table goes away?

**This one blocks the build.** We cannot complete this interface without it.

**Why we are asking:** The XREF data (XREF_UCMID_TABLE) is not in the export, and Boomi Cross Reference Tables are static. Decide where the candidateID-&gt;contentID map lives at runtime; dat-profile-generation-rest reads it to fetch HDL errors.

**If this is not answered:** A small table in a TaskUs-managed database reached through a Boomi Database connection.

**The choices as we understand them:**

- Database table
- Boomi-managed store
- Not needed (pass contentID directly)

**Answer:**

<!-- answer:Q-001 -->
<!-- /answer:Q-001 -->

### Q-002 · Which employee data layout is used in production for new hires: the original (2017) layout or the Jobvite layout with repeating education and previous-employment details?

**Why we are asking:** The PROD_MDS EmployeeEBM.xsd is the 2017 layout. Confirm which EBO the production DAT generation is compiled against, because the profile depends on it.

**If this is not answered:** The PROD_MDS Employee/V1 layout.

**The choices as we understand them:**

- Original layout
- Jobvite layout

**Answer:**

<!-- answer:Q-002 -->
<!-- /answer:Q-002 -->

### Q-003 · Should rehires reuse the original assignment and work-terms numbers, or are the different numbers used today ("E&lt;id&gt;", "ET_&lt;id&gt;") intended?

**Why we are asking:** Hire and rehire files use different key patterns; mismatches can create duplicate assignments in HCM.

**If this is not answered:** Keep the legacy patterns unchanged.

**Answer:**

<!-- answer:Q-003 -->
<!-- /answer:Q-003 -->

### Q-004 · Are salary, salary basis and talent-profile records for new hires created elsewhere, so that they can stay out of this service?

**Why we are asking:** Those files are switched off here; dat-profile-generation-rest appears to produce them.

**If this is not answered:** Yes, they stay in dat-profile-generation-rest.

**Answer:**

<!-- answer:Q-004 -->
<!-- /answer:Q-004 -->
