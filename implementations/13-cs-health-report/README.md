# cs-health-report — Implementation #23

**Built for:** RevSend Customer Success
**Shipped:** 2026-04-19
**Author:** Maxwell Wilber
**Package:** `cs-health-report.skill` (~27 KB)

---

## 1. Identity & Purpose

A Claude skill that wraps Max's internally-built "CS Health Score Pipeline v2.1" and turns his daily Customer Success morning triage into a single natural-language prompt producing a shareable priorities-first PDF.

The pipeline already existed. What shipped in this project is the skill package (`SKILL.md` + `references/` + `scripts/` + `assets/`), the reportlab PDF renderer (~700 lines), a sample dataset, a three-case eval harness, and the benchmark tooling. This is an AE-authored AI productivity tool, not a data-engineering deliverable.

**Problem solved:** Max's morning ritual — reconciling HubSpot activity, product usage trends, and Stripe billing state into a priority-ordered contact list — was an estimated ~15-minute context switch each morning, inconsistent between days, and produced no artifact shareable with managers or handoffs.

**Users:** Max Wilber (AE/CS at RevSend) is the primary user. The skill is installed at the RevSend org level, so any teammate invoking it in Cowork can generate the same report.

---

## 2. Input → Output

**Live-pipeline inputs:**
- HubSpot REST API: companies, deals, owners, contacts, engagement activities
- MySQL read replica: product usage events, session counts, feature adoption
- Stripe API: active subscriptions, invoice status, MRR, `cancel_at_period_end` flags
- Environment config: `REPORT_DATE`, `CS_HEALTH_HISTORY_FILE_ID` for week-over-week deltas

**Render-only input path:** an existing `cs_report_v2.json` from a prior pipeline run.

**Output:** Single multi-page PDF (`cs_morning_report_YYYY-MM-DD.pdf`) with seven canonical sections:
1. Header (date, portfolio owner, timestamp)
2. Dashboard tiles — 2×4 grid: HEALTHY / MONITOR / AT_RISK / CRITICAL counts; total MRR; avg health score; urgent-signal count; trend-vs-last-week
3. Today's Priorities — URGENT → WARNING → ONBOARD → QBR → CHECK-IN, with prescribed action text per signal
4. Churn Alerts — grouped by tier with raw signal evidence
5. Check-ins Due — 30-day and 90-day QBRs, new-customer onboarding
6. Trend Alerts — week-over-week health score movement
7. All-Accounts table — sorted lowest-score-first
8. (Optional) Deep dives — pillar breakdown + key contacts + signal evidence for named accounts, on request only

**Delivery:** PDF written to the user's Cowork-mounted workspace folder. On-demand only. No email or Slack push (explicit product decision).

---

## 3. Pipeline Architecture

Ordered steps from `scripts/run_pipeline.sh` + `SKILL.md` + `scripts/render_pdf.py`:

1. **Input discovery.** Skill-loader resolves input: workspace folder → uploaded attachments → `/tmp/cs_report_v2.json`. Passes JSON forward or proceeds to live run.
2. **Environment validation.** Shell script validates `HUBSPOT_ACCESS_TOKEN`, `STRIPE_API_KEY`, `MYSQL_*` creds. Missing HubSpot → hard abort. Missing Stripe or MySQL → continues with degraded scoring.
3. **Dependency install.** `pip install pymysql requests stripe reportlab --break-system-packages` (idempotent).
4. **Credential patch.** Idempotent Python regex replacing hardcoded MySQL creds at lines 110–115 of `cs_pipeline_v2.py` so the same script works in any environment.
5. **Pipeline execution.** `python cs_pipeline_v2.py` pulls all three sources, computes scores, writes `cs_report_v2.json`. Deterministic. No LLM.
6. **JSON shape validation.** Verifies required keys (`accounts`, `summary`, `priorities`, `signals`); fails with a clear error otherwise.
7. **PDF rendering.** `render_pdf.py` reads the JSON and builds a multi-page PDF via reportlab. Enforces URGENT-before-WARNING ordering; quiet-day fallback messaging; account-sort lowest-score-first.
8. **Deep-dive rendering (optional).** Same script re-runs with `--deep-dive NAME` flags and appends pillar breakdowns for named accounts.
9. **Output delivery.** PDF copied to the workspace folder and presented via a `computer://` link.

**Conditional branches and fail-closed checks:**
- JSON-exists branch: skips steps 2–6, goes straight to rendering.
- MySQL unavailable: Product-Usage pillar zeroed, degraded state logged in JSON.
- Stripe unavailable: Payment-Health pillar defaults to neutral, logged.
- HubSpot unavailable: hard abort — no credible report possible without it.
- Empty priorities: PDF renders a "no priority actions today" block plus top-3 HEALTHY accounts as expansion candidates. Implemented in `render_pdf.py`, not LLM-decided.

**Runtime:**
- Render-only path (JSON exists): ~27.9 s mean (benchmark data).
- Baseline Claude without skill averaged 50.3 s on the same task — 44% slower.
- Full pipeline path: depends on HubSpot API latency; time a live run for a real number.

---

## 4. Claude Integrations Used

**Claude products / runtime:**
- Claude Agent SDK / Cowork mode — host environment
- Execution model: whichever Claude model is powering the active Cowork session. Skill is model-agnostic and does not call the Claude API directly from its scripts.

**MCPs invoked at runtime:** None. The skill deliberately bypasses MCPs in favor of direct HubSpot REST + MySQL replica + Stripe SDK for determinism and full control over scoring inputs. A future version could add the HubSpot MCP as an alternate input path for teammates without direct API credentials.

**External (non-Claude) integrations inside the pipeline:**
- HubSpot REST API (via `requests`) — companies, deals, owners, contacts, engagement activities
- MySQL replica (via `pymysql`) — product usage events, sessions, feature adoption
- Stripe Python SDK — subscriptions, invoices, MRR, `cancel_at_period_end` flags

---

## 5. Quantifiable Metrics

Shipped 2026-04-19. Production metrics do not yet exist. Only reproducible, eval-derived numbers are claimed below.

**Eval results (from `build_benchmark.py` against `iteration-1/*/grading.json`):**
- With skill: 31 / 31 assertions passed across 3 test cases (100%)
- Baseline Claude without skill: 26 / 31 passed (83.8%)
- Delta: +16.2 percentage points
- Baseline failure modes observed: missing avg-health-score tile, missing urgent-signal count, no "Today's Priorities" heading, missing pillar breakdowns in deep dives, no quiet-day expansion fallback

**Execution time (eval mean):**
- With skill: 27.9 s per run
- Baseline: 50.3 s per run
- Delta: –44.5%

**Not yet measurable (requires production data or a formal baseline stopwatch):**
- Manual baseline timing (currently an estimate — needs a real measurement)
- Time saved per report
- Cumulative hours redeployed
- Token cost per run
- Error / rework rate in production
- Retention, expansion, or NPS impact

---

## 6. Technical Differentiation

**What makes this non-obvious:**
- **Progressive-disclosure skill architecture.** `SKILL.md` is intentionally minimal (~120 lines). The model only loads `references/run_pipeline.md`, `references/scoring_spec.md`, `references/signal_catalog.md`, or `references/pdf_layout.md` when the current task requires that depth. Keeps per-call context cost low while keeping the full spec available.
- **Deterministic where deterministic wins.** Pillar scoring is pure Python. PDF layout is reportlab. The LLM orchestrates — it decides whether to render deep-dives, which accounts were named in a natural-language request, and how to explain output — but it does not hallucinate numeric scores or reshape the PDF structure.
- **Dual-entry design.** Same skill handles "run the pipeline from scratch" and "render from the JSON I already have." Matters because full runs take real time; users often want a re-render against the prior day's data without re-pulling.
- **Prefix-driven priority ordering.** `render_pdf.py` parses `URGENT:` / `WARNING:` / `ONBOARD:` / `QBR:` / `CHECK-IN:` prefixes and enforces render order. Means a future LLM pass can append priorities to the JSON without needing to resort them.
- **Explicit quiet-day branch.** When `priorities == []`, the renderer switches to an expansion-scan message plus top-3 HEALTHY accounts. Baseline Claude without the skill silently produces an empty section or hallucinates priorities to fill the space — observed in the eval.
- **Eval-driven development.** Three test cases with hard assertions graded by `pypdf` text extraction — not vibes. Reproducible.
- **Idempotent credential patching.** Regex-patches the hardcoded MySQL creds in the pipeline script, so the same script works across dev/prod without forking.

**What would break with a single prompt:**
- PDF layout consistency (drifts per run)
- Numeric-score fidelity (LLM would eyeball, not compute)
- URGENT-before-WARNING ordering guarantee
- Quiet-day behaviour (single-prompt baseline empirically failed this case)
- Reproducible artifacts — a single prompt doesn't ship a PDF file

**Senior-engineer design choices worth flagging:**
- Keeping the LLM out of scoring math
- Eval harness before "ship it"
- Dual entry points instead of hard-coupling to a fresh pipeline run
- Fail-closed on the only non-degradable dependency (HubSpot); fail-open on Stripe / MySQL with explicit logging
- Progressive disclosure in the skill rather than one mega `SKILL.md`

---

## 7. Deployment Status

- **Status:** Shipped to Max's RevSend org. v1 prototype, ready for daily use.
- **Deployment date:** 2026-04-19
- **Trigger:** On-demand, via natural-language invocation in Cowork. Trigger phrases: "CS report", "health check", "morning CS briefing", "who should I focus on today", "which accounts are at risk", "churn signals", "QBR list", "30-day check-ins", "run the CS pipeline", plus direct references to `cs_pipeline_v2.py` or `cs_report_v2.json`. Not scheduled — explicit product decision.
- **Distribution:** `cs-health-report.skill` (~27 KB) packaged artifact. Installed at the org level.
- **Maintainer:** Max Wilber.

---

## 8. Business Outcomes for RevSend

Shipped 2026-04-19. Zero production data points yet.

**Observable now:** Morning ritual compressed from a multi-tool context switch to a single natural-language prompt producing a shareable PDF artifact.

**Expected, not yet measured:**
- Time-per-morning from ~15 min to under 1 min (requires before/after stopwatch data)
- Consistency of prioritization across days
- Shareability — PDFs can be forwarded to a manager or a handoff teammate without hand-writing a summary

**Retention / expansion / NPS impact:** unknown — requires a 60–90 day comparison window before claiming anything.

**Measured outcomes will be added here after 30–60 days of production use.**

---

## 9. Resume Bullet (Published)

> Built and shipped a Claude skill (`cs-health-report`) that generates a daily Customer Success priority report for a 14-account book of business by orchestrating HubSpot, MySQL, and Stripe inputs through a deterministic Python scoring pipeline with a reportlab PDF renderer; eval-graded at 100% vs. 83.8% baseline Claude and 44% faster execution across 3 test cases with 31 assertions.

Every number in this bullet is reproducible from `build_benchmark.py` + `iteration-1/*/grading.json`.

---

## Reproducibility

All benchmark numbers in this document are reproducible by running:

```bash
python build_benchmark.py
```

against the test cases in `iteration-1/*/grading.json`. Assertions are graded by `pypdf` text extraction, not subjective review.
