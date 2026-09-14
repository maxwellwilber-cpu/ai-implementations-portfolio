# CRM Audit and Batch Remediation (HubSpot API)

**Built for:** the platform, a B2B SaaS corporate gifting platform
**Deployed:** 2025
**Stack:** Claude with the HubSpot MCP server, HubSpot CRM REST API, batch operations

---

## What it is

An automated audit of the platform's HubSpot instance. It analyzed 612 contacts, categorized them by engagement status, and executed batch updates to standardize lead statuses and create structured activity notes. 550 update operations ran with zero failures.

**The problem.** The outbound SDR team had six months of CRM drift behind it: inconsistent lead statuses, contacts with no categorization at all, voicemails logged in whatever format the rep felt like, and duplicate records created when people re-imported prospect lists. Cleaning that by hand is roughly eight hours of SDR time, and nobody had eight hours.

**Who used it.** The SDR team. I was at the SaaS company as a contracted employee handling customer success and automations. The cleaned-up CRM state became the foundation for the Outbound Sales Agent skill. That agent cannot reason usefully about a prospect if the underlying record is wrong.

---

## Input and output

**In:**
- 612 HubSpot contact records across the SDR team's pipeline
- The existing HubSpot property schema: lead status taxonomy, activity types, standard contact fields
- A defined target state, meaning every contact carries a lead status, every voicemail has a standardized note, and duplicates are resolved

**Out:**
- 612 contacts audited and sorted into three groups: 507 already fine, 43 needing a status correction, 62 needing a status assigned
- 550 contacts batch updated, which is the 507 plus the 43 corrections, run as 55 batches of 10, with 0 failures
- 54 standardized voicemail activity notes created with contact associations
- 11 duplicate-contact errors resolved automatically during the run
- 4 structural duplicates flagged for manual merge
- Category counts checked against the total: 507 + 43 + 62 = 612

---

## How it ran

1. **Schema read.** `get_properties` through the HubSpot MCP returns the current contact schema: which properties exist, their types, their valid enum values. This is what the update step writes against.
2. **Audit.** `search_crm_objects` with filter groups paginates through all 612 contacts, pulling the property set relevant to the audit: lifecycle stage, lead status, recent activity, creation source.
3. **Categorization.** Each contact is checked against the target state. Does it have a lead status? Does that status match its observable activity history? Is there recent activity that contradicts it? That produces the three groups above.
4. **Batch updates.** 550 contacts are written through `manage_crm_objects` in batches of 10. That is the 507 already in an acceptable state plus the 43 corrections, with lead status written explicitly so the whole book carries one taxonomy instead of a mix of set and inferred values. The 62 needing a status assigned are handled separately, because giving a status to a contact that has never had one is a judgment call, not a standardization.
5. **Error resolution.** Create operations that collide with an existing record return a "Contact already exists" error. Those were caught, the existing contact ID retrieved, and the operation redirected to update that record instead of creating a new one. 11 contacts resolved this way with no manual work.
6. **Count check.** Final category counts were cross-referenced against the total to confirm nothing was lost, skipped or double counted.

---

## Edge cases

**Duplicate collisions on create.** HubSpot returns a specific error when a create conflicts with an existing record on a unique property, usually email. Those were converted into updates against the existing contact. 11 during this run.

**Structural duplicates.** Two contacts sharing an email but carrying separate activity histories were not merged automatically. Merging them would collapse two distinct relationship records into one, and picking which history survives is a judgment call. 4 were flagged for manual review with their data attached.

**Rate limits.** HubSpot enforces per-second and per-day API limits. Batches of 10 with normal pacing between them stayed well under the per-second cap. No rate limit errors occurred.

---

## Numbers

| | |
|---|---|
| Contacts audited | 612 |
| Contacts batch updated | 550 (507 + 43) |
| Batches | 55 |
| Batch failures | **0** |
| Activity notes created | 54 |
| Lead status corrections | 43 |
| Duplicate errors resolved automatically | 11 |
| Structural duplicates flagged for review | 4 |
| Category reconciliation | 507 + 43 + 62 = 612 |

**Time.** Manual baseline was roughly 8 hours of SDR work, counting context switching. The run took about 15 minutes.

---

## What the work actually required

The API calls are the easy part. Three things took the thinking:

**Defining the target state before touching anything.** "Clean CRM" is not a specification. Deciding what every contact must have, what counts as an acceptable status, and what makes a record wrong is the work that makes automation possible at all. That definition came out of the HubSpot operating system I had written for the team first.

**Deciding what not to automate.** The 4 structural duplicates could have been merged programmatically. They were not, because choosing which activity history survives a merge is a business decision and getting it wrong destroys relationship data that cannot be recovered. Knowing where to stop matters more than how much you can automate.

**Checking the counts.** 507 + 43 + 62 = 612 is simple arithmetic, and it is the only thing standing between "the script finished" and "the script did what it was supposed to." A sweep that silently skips 40 contacts still reports success without it.

---

## Deployment

Ran as a one-time remediation sweep against the platform's live HubSpot portal in 2025. The categorization logic and the note format were carried forward as the team's ongoing standard.

## Outcome for the platform

Lead statuses became consistent across the pipeline, which made sequence targeting reliable. Voicemail activity became searchable, so questions like "which voicemails from last week never got a follow-up" became answerable. Roughly 8 hours of SDR time per audit cycle went back to selling.

## Resume line

Built an automated CRM audit and batch remediation system for a B2B SaaS sales team using Claude with the HubSpot MCP server, auditing 612 contacts and executing 550 batch updates across 55 batches with 0 failures and 11 automated duplicate-error recoveries; reclaimed roughly 8 hours of SDR time per audit cycle.

## Reproducibility

The audit logic can be re-run against the live portal at any time. The categorization rules, the note format and the count check work the same whether the contact count is 612 or 6,120.
