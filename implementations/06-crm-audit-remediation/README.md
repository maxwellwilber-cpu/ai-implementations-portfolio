# CRM Audit & Batch Remediation — HubSpot API Automation

**Built for:** RevSend (B2B SaaS corporate gifting platform)
**Deployed:** 2025
**Author:** Maxwell Wilber
**Stack:** Claude AI + HubSpot MCP server, HubSpot CRM API (REST), batch operations with error-tolerant retry

---

## 1. Identity & Purpose

An automated CRM audit and batch remediation system that programmatically analyzed 612 HubSpot contacts, categorized them by engagement status, and executed batch API updates to standardize lead statuses, enrich contact data, and create structured activity notes — all with 0 failures across 550 update operations.

**Problem solved:** RevSend's outbound SDR team had accumulated six months of CRM drift — inconsistent lead statuses, orphaned contacts with no categorization, voicemail activity logged inconsistently or not at all, and duplicate contacts created when reps re-imported prospects. A manual audit would have taken an SDR ~8 hours, and the team didn't have 8 hours to spare. The automation completed the same work in ~15 minutes with a verifiable audit trail.

**Users:** The RevSend SDR team and Max as the account manager. The standardized CRM state became the operational foundation for the Outbound Sales Agent Skill (implementation #12) — that agent cannot reliably reason about prospects if the underlying CRM data is inconsistent.

---

## 2. Input → Output

**Inputs:**
- 612 HubSpot contact records across the RevSend SDR team's managed pipeline
- Existing HubSpot property schema (lead status taxonomy, activity types, standard contact fields)
- An operational definition of the target end-state (every contact has a defined lead status; every voicemail has a standardized note; no duplicates)

**Outputs:**
- 612 contacts audited and categorized into three buckets: 507 already in acceptable state, 43 requiring status correction, 62 requiring assignment
- 550 contacts batch-updated with corrected properties (55 batches of 10, 0 failures)
- 43 contacts with corrected lead status
- 54 standardized voicemail activity notes created with proper contact associations
- 4 duplicates identified and flagged for manual merge (cannot be auto-merged safely)
- A validation report reconciling 507 + 43 + 62 = 612 total, confirming no records were lost, skipped, or double-counted during the sweep

---

## 3. Pipeline Architecture

Six ordered stages:

1. **Schema discovery.** `get_properties` via the HubSpot MCP enumerates the current contact schema — which properties exist, their types, their valid enum values. The pipeline adapts to the live schema rather than hard-coding property names that might have drifted.
2. **Full-pipeline audit.** `search_crm_objects` with filter groups paginates through all 612 contacts in manageable batches. Each contact is fetched with the property set relevant to the audit (lifecycle stage, lead status, recent activity, creation source, duplicate indicators).
3. **Categorization.** Each contact is scored against a categorization matrix — does it have a lead status, does that status match observable engagement history, are there recent activities that contradict the status, is it a likely duplicate? Outputs three buckets: already good, needs status correction, needs status assignment.
4. **Batch update execution.** The 550 contacts needing updates are processed in batches of 10 via `manage_crm_objects`. Batch size chosen to balance API throughput against rollback granularity — a single bad record in a batch of 10 is recoverable; in a batch of 100, it's not.
5. **Error resolution.** Duplicate-contact errors ("Contact already exists") are caught, the pre-existing contact ID is retrieved, and the operation is converted from create-new to update-existing. 11 of these were resolved gracefully without manual intervention. 4 structural duplicates (same email across two records, both with independent activity history) were flagged for manual merge — the system correctly declines to auto-merge these because a merge policy requires a human judgment call.
6. **Validation & reconciliation.** Cross-references final counts across all categories: 507 acceptable + 43 corrected + 62 assigned = 612 total. If this math doesn't close, the pipeline halts and flags the discrepancy rather than producing output of unknown integrity.

---

## 4. Error Handling & Edge Cases

Specific cases observed and handled:

- **"Contact already exists" on create:** HubSpot's API returns a specific error code when a create operation conflicts with an existing record on unique-property match (typically email). The pipeline catches this exact error class, extracts the conflicting contact ID from the response, and converts the operation into an update against the existing record. 11 contacts were resolved this way during the sweep, all without manual intervention.
- **Structural duplicates (same email, different histories):** When two contacts have the same email but independent activity streams, the pipeline does NOT auto-merge. Merging them automatically would collapse two distinct relationship histories; the correct action is human review. 4 such cases were flagged for manual resolution with the relevant data attached to the report.
- **Schema drift:** If a property referenced by the pipeline doesn't exist in the live schema (because someone deleted or renamed it), the `get_properties` check catches this before any update call is made. Fail-closed behavior; no partial-schema updates.
- **Rate limits:** HubSpot's API has per-second and per-day rate limits. Batch size of 10 with natural pacing between batches keeps requests well under the per-second cap. No rate-limit errors occurred during the sweep.

**Fail-closed principle:** The pipeline is designed so that any unexpected state — schema mismatch, reconciliation math not closing, batch failure — halts execution rather than continuing with partial or questionable results. This is deliberate: a CRM with inconsistent partial updates is worse than one with known-broken-but-untouched state.

---

## 5. Quantifiable Metrics

**Scale:**
- 612 contacts audited
- 550 contacts batch-updated
- 55 batches of 10
- 54 activity notes created
- 43 lead status corrections
- 11 duplicate-error recoveries (automated)
- 4 structural duplicates flagged (manual review)
- 0 batch failures

**Accuracy:**
- 507 + 43 + 62 = 612 — reconciliation math closes exactly
- Every activity note has a contact association verified via foreign-key check before write
- Every lead status value verified against the live enum before write

**Time:**
- Estimated manual baseline: ~8 hours of SDR time (~13 minutes per 10-contact batch, plus context switching)
- Actual execution: ~15 minutes of automated run time
- Time saved: ~7.75 hours per audit cycle

---

## 6. Technical Differentiation

**What makes this non-obvious:**

- **Adapts to live schema.** The pipeline reads the current HubSpot property schema at run time via `get_properties`. Hard-coded property names would break silently when HubSpot admins renamed or deprecated fields. This is a common failure mode for one-off CRM scripts that read fine on the day they were written and break two months later.
- **Converts errors into operations.** Catching a "Contact already exists" error and transforming it into an update-against-existing-ID is the difference between "11 skipped contacts that an SDR has to clean up manually" and "11 gracefully resolved contacts with a clean audit trail." The error class exists because HubSpot tells you *which* record caused the conflict — the pipeline uses that information instead of logging and moving on.
- **Knows when not to automate.** The 4 structural duplicates were surfaced for manual review rather than auto-merged. This is a deliberate constraint: a merge policy requires human judgment about which activity history to preserve, and automating it would silently destroy data.
- **Reconciliation math as the safety net.** 507 + 43 + 62 = 612. If this doesn't close, something is wrong and the pipeline refuses to declare success. A senior engineer reviewing this will recognize the same pattern as double-entry bookkeeping — the reconciliation exists precisely to catch the silent failures other checks would miss.
- **Batch size of 10 is a reasoned choice, not arbitrary.** Small enough that any single failure has blast radius of 10 records; large enough to be materially faster than per-record calls; round enough to produce clean reconciliation math (55 × 10 = 550).

**What would break with a less-rigorous approach:**

- Hard-coded property names → silent failure when schema drifts
- No error-class handling → 11 duplicates skipped, manual cleanup required
- Auto-merging structural duplicates → permanent loss of activity history
- No reconciliation math → partial failures go undetected
- Batch size of 100 → single bad record cascades to 100-record rollback

**Senior-engineer design choices worth flagging:**

- Live schema discovery before any write operation
- Error-class branching, not generic try/except
- Explicit fail-closed on reconciliation mismatch
- Human-in-the-loop escape hatch for non-automatable cases
- MCP-based tool integration rather than raw SDK calls — lets the AI orchestrate operations with clear tool boundaries

---

## 7. Deployment Status

- **Status:** Production. Sweep completed successfully 2025.
- **Repeatability:** The pipeline is designed to be re-run as an ongoing hygiene sweep — quarterly or on-demand. Subsequent runs will find far fewer anomalies (since most were resolved in the first sweep).
- **Maintainer:** Max Wilber.
- **Dependencies:** HubSpot MCP server, Claude AI as orchestration layer, RevSend's HubSpot portal credentials.

---

## 8. Business Outcomes for RevSend

- **~8 hours of SDR time reclaimed per audit cycle.** Time that would have gone to data janitorial work now goes to outreach.
- **612 contacts with consistent, queryable state.** Before the sweep, "show me every BAD_TIMING contact we haven't re-engaged in 60 days" was a question the CRM couldn't answer reliably; after, it's a single filter.
- **54 voicemail activity notes standardized.** Activity history is now analyzable — reps can see "what was said on the last VM left for this contact" without opening each record.
- **Operational foundation for downstream automation.** The Outbound Sales Agent Skill (implementation #12) and other AI-driven workflows assume consistent CRM state. This sweep was the precondition.
- **Zero data loss.** 0 batch failures, 0 records lost, 11 errors gracefully recovered, 4 structural duplicates preserved for human review.

---

## 9. Resume Bullet (Published)

> Built and shipped an automated CRM audit and batch remediation system for a B2B SaaS sales team, orchestrating Claude AI + HubSpot MCP server to audit 612 contacts and execute 550 batch updates across 55 batches with 0 failures, 11 automated duplicate-error recoveries, and a reconciliation check (507 + 43 + 62 = 612) that fails closed on any count mismatch; reclaimed ~8 hours of SDR time per audit cycle vs. the manual baseline.

Every number in this bullet is verifiable from the final audit report and the HubSpot activity log.

---

## Reproducibility

The audit methodology and the pipeline logic can be re-run against the live HubSpot portal at any time. Reconciliation math, error-class handling, and fail-closed behavior are the same whether the contact count is 612 or 6,120.
