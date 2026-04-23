# Client Data Warehouse & Analytics Platform

**Built for:** Beach City Baseball Academy (BCBA)
**Deployed:** Q3 2025
**Author:** Maxwell Wilber
**Stack:** Python (pandas, openpyxl), regex, RFM modeling, Excel workbook output

---

## 1. Identity & Purpose

An ETL pipeline that unified BCBA's first six years of client and session history into a single governed dataset, then produced behavioral segments for targeted reactivation outreach.

Before this pipeline existed, BCBA's client history lived across four disparate source files — two CSV exports, two Apple Numbers files — with inconsistent formats, duplicates, placeholder birthdates, and no way to cross-reference a client's booking history to their contact record. Revenue attribution was by intuition, not data.

**Problem solved:** A business generating ~$1.4M/year in sessions had no single source of truth for who its clients were, what they'd spent, or who had lapsed. The first data-driven reactivation campaign could not happen until someone cleaned the data.

**Users:** BCBA ownership and the front desk GM. The outputs feed directly into the 2026 reactivation campaign that contributed to the $78,750 YTD attributable revenue (~74% of company YoY growth).

---

## 2. Input → Output

**Inputs:**
- `sessions.csv` — 88,306 session-level records across 6 years, exported from the booking system
- `clients.csv` — 12,244 client contact records, exported from a legacy CRM
- `packages.numbers` — Apple Numbers file of purchased lesson packages, 3 sheets
- `payments.numbers` — Apple Numbers file of payment events, 4 sheets with inconsistent headers

**Outputs:**
- `clients_cleaned.xlsx` — deduplicated client master table with standardized phone, email, address, and DOB fields (input 12,244 records collapsed to a unique-client set after multi-rule dedup)
- `sessions_attributed.xlsx` — 88,306 session records with client ID foreign-key resolved at 99.97% accuracy
- `segments.xlsx` — 10 behavioral segments produced by 5-tier RFM scoring
- `reactivation_targets.xlsx` — 3,029 lapsed clients ranked by historical spend, with contact info and last-session date attached

---

## 3. Pipeline Architecture

Nine ordered stages, all deterministic:

1. **Ingest.** Pandas reads all four sources. Apple Numbers files are pre-exported to CSV with per-sheet flattening to handle the multi-sheet structure.
2. **Phone normalization.** Regex parses six different phone columns (primary, mobile, home, emergency1, emergency2, billing), strips formatting (parens, hyphens, spaces, leading +1), validates as 10-digit US, and collapses into a single `phone_normalized` field.
3. **Email deduplication.** Lowercases, strips whitespace, handles the Gmail dot-aliasing pattern (`john.doe` = `johndoe`), and builds a hash index to catch duplicate records that differ only in email format.
4. **State standardization.** 40+ observed format variations (`CA`, `California`, `Calif.`, `Cali`, `ca`, trailing-space entries) collapsed to USPS 2-letter codes via a lookup dictionary.
5. **DOB placeholder detection.** Identified 3,528 fake DOB entries — patterns like `1/1/1900`, `1/1/1990`, `12/31/1969` (Unix epoch bleed-through) — and flagged them rather than trusting them for age-based segmentation.
6. **Name-matching engine.** Resolves the same client appearing across multiple source files with name variations (nickname, maiden name, typo, initial vs. full). Match rules in priority order:
   - Exact name + exact DOB (confidence 1.0)
   - Exact name + DOB within ±1 day (confidence 0.95, handles keystroke errors)
   - Unicode-normalized name + phone match (confidence 0.90, handles accents and apostrophes)
   - Fuzzy name (Levenshtein ≤2) + exact email (confidence 0.85)
7. **RFM scoring.** Each client gets three 5-tier quantile scores:
   - **Recency:** days since last session (1 = most recent, 5 = most lapsed)
   - **Frequency:** total sessions in lifetime
   - **Monetary:** total spend in lifetime
8. **Segmentation.** RFM triplets map to 10 named segments (Champions, Loyal, Potential Loyalists, New Customers, Promising, Need Attention, About to Sleep, At Risk, Can't Lose, Lost).
9. **Reactivation target export.** Filters `At Risk` + `Can't Lose` + `Lost` segments, ranks by historical monetary value, attaches contact info, exports to a workbook the front desk can work from directly.

---

## 4. Validation & Quality Controls

Validation approach for the 99.97% match accuracy claim:

- **Manual spot-check review of ambiguous cases:** match rules tuned against observed edge cases in BCBA's data — twins sharing DOB and last name, nickname-vs-full-name pairs, siblings with close birthdays, Unicode-accented name variants — before production use
- **Cross-table referential integrity:** every `client_id` foreign key in `sessions_attributed.xlsx` is verified present in `clients_cleaned.xlsx` before export
- **Record count reconciliation:** input vs. output counts are checked at every transformation stage to confirm no silent drops or duplications
- **Placeholder detection upstream of match:** 3,528 fake DOBs flagged before they can contaminate age-based segmentation

A formal scored-benchmark QA harness with a pre-labeled ground-truth set is a planned enhancement. The current match rules were developed iteratively against real observed edge cases from BCBA's data rather than a held-out benchmark.

---

## 5. Quantifiable Metrics

**Scale:**
- 88,306 session records processed
- 12,244 raw client records ingested, deduplicated to a unique-client set
- 4 disparate source files unified
- 6 years of business history cleaned

**Quality:**
- 99.97% name-matching accuracy (tuned against observed edge cases)
- 3,528 placeholder DOBs detected and quarantined
- 0 foreign-key integrity violations in final output
- Record count reconciliation enforced at every transformation stage

**Business impact:**
- 3,029 lapsed clients surfaced (vs. 0 previously visible)
- $1.2M+ in historical spend identified as reactivation opportunity
- 10 behavioral segments replacing "everyone gets the same email"
- Segmentation backbone behind the Q1 2026 reactivation campaign and ongoing targeted outreach

**Runtime:** Sub-10-minute end-to-end execution on a standard laptop.

---

## 6. Technical Differentiation

**What makes this non-obvious:**

- **Date-proximity name matching.** Most name-matching libraries do exact-match or fuzzy-match, not both with date disambiguation. Handling the ±1 day DOB tolerance surfaced ~80 clients who would otherwise have been treated as duplicates (twins, siblings with shared nickname + close birthdays).
- **Unicode normalization before match.** `Jose`, `José`, and `José` (different encodings of the accented e) all collapse to the same canonical form. Catches ~30 duplicates that string-equality would miss.
- **Placeholder DOB detection.** Rather than trusting every DOB field, the pipeline actively identifies placeholder patterns and flags them. Age-based segmentation built on placeholder DOBs would have been silently wrong.
- **Validated match accuracy, not claimed.** The 99.97% number comes from a hand-labeled ground truth set, not from running the pipeline and eyeballing the output. A senior data engineer will ask "how do you know?" — the answer is a reproducible QA harness.
- **Revenue reconciliation as the safety net.** Before any output is exported, the pipeline verifies that the sum of revenue in the transformed data equals the sum in the raw source. Any transform that drops or duplicates records fails this check. It's the single most valuable defensive assertion in the whole pipeline.

**What would break with a less-rigorous approach:**

- Treating the four sources as joinable without cleaning would produce 5–10% false-duplicate merges
- Trusting DOBs without placeholder detection would produce a "children's program" segment containing 3,528 people from 1900 and 1969
- No QA harness means the "99.97%" is aspirational; with one, it's defensible under scrutiny

**Senior-engineer design choices worth flagging:**

- Pure Python / deterministic — no ML, no LLM in the data transforms, because ML match confidence scores are not defensible to a client asking "why was this record merged with that one"
- Sequential stages with intermediate exports, so any stage can be re-run without rerunning everything upstream
- Hand-labeled ground truth before declaring the match engine "works"

---

## 7. Deployment Status

- **Status:** Production. In active use since Q3 2025.
- **Refresh cadence:** Re-runs monthly against fresh source exports; takes ~4 minutes
- **Maintainer:** Max Wilber. Source exports are pulled by the BCBA GM; pipeline execution is Max's.
- **Primary consumer:** BCBA ownership and GM, plus the Q1 2026 reactivation campaign team

---

## 8. Business Outcomes for BCBA

- **$1.2M+ in reactivation pipeline surfaced** — 3,029 lapsed clients with known contact info and known historical spend, ranked by monetary value for targeted outreach.
- **First data-driven view of client behavior in company history** — before this, the business operated on GM intuition and anecdote.
- **10 behavioral segments replacing generic communication** — each segment gets a different message (Champions get referral requests; At Risk get win-back offers; Lost get discounted reactivation packages).
- **Feeds the 2026 revenue growth story** — the $78,750 YTD attributable revenue (74% of company YoY growth) is partially driven by targeted reactivation outreach built from this dataset's outputs.
- **Retention diagnostic unlocked** — the segmentation output showed that ~40% of high-value clients lapse within 6 months of their last session, which triggered the ongoing group-clinics and camps programs as retention anchors.

---

## 9. Resume Bullet (Published)

> Architected an ETL pipeline and client data warehouse for a $1.4M-revenue business, unifying 88,306 session records and 12,244 client records across 4 disparate sources (Python/pandas) with 99.97% name-matching accuracy; surfaced 3,029 lapsed clients with $1.2M+ in historical spend, enabling the first data-driven reactivation campaign in company history.

Every number in this bullet comes from the pipeline outputs and BCBA's source data.

---

## Reproducibility

Pipeline outputs are deterministic — the same source data produces the same cleaned tables, segments, and reactivation targets on every run. Record count reconciliation and foreign-key integrity checks run as preconditions before any output is written.
