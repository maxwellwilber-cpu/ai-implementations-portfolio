# outbound-sales-agent — Packaged Claude Skill

**Built for:** RevSend (B2B SaaS corporate gifting platform)
**Deployed:** 2025
**Author:** Maxwell Wilber
**Package:** `outbound-sales-agent.skill`

---

## 1. Identity & Purpose

A packaged Claude skill that encapsulates the complete RevSend outbound sales workflow — ICP logic, messaging selection, sequence management, CRM procedures — into a reusable, deployable agent configuration installable at the RevSend org level.

**Problem solved:** Before this skill existed, executing an outbound sales motion at RevSend required an SDR to cross-reference six separate documents: the Enterprise Sales Intelligence master (implementation #1), the ICP Framework (#2), the Messaging Framework (#3), the Industry Story Lookup (#4), the Competitive Battle Cards (#5), and the HubSpot Operating System (#7). Any question — "what sequence do I run on this lead?" "which template fits this ICP?" "how do I log this voicemail?" — required a manual traversal. The skill collapses all six into a single natural-language interface.

**Users:** The RevSend SDR team. Any rep can invoke the skill in Cowork with a question like "draft a first-touch sequence for this prospect" and receive an ICP-aligned, messaging-template-backed, sequence-rule-compliant answer in one pass.

---

## 2. Invocation → Output

**Invocation:** Natural-language prompts in any Cowork session with the skill installed. Examples:
- "Draft a sequence for this prospect: [prospect info]"
- "What template should I use for the day-3 follow-up on this account?"
- "How should I log the voicemail I just left for [contact]?"
- "Which ICP does this company fit?"
- "The prospect mentioned they're evaluating Sendoso — what's my play?"

**Output:** Task-appropriate response pulling from the relevant subsystem:
- Sequence drafting → full sequence with per-touch message, timing, and CTA
- Template selection → specific template from the 30+ library, with the "why this fits" rationale
- Activity logging → the exact HubSpot note format to paste (per the operating system standard)
- ICP classification → one of 5 tiers with the reasoning
- Competitive positioning → the battle card for that competitor with the specific talking points

---

## 3. Skill Architecture

Progressive-disclosure architecture matching the pattern Anthropic uses for its official skills:

**Core `SKILL.md`:** ~100 lines. Contains only the skill identity, the trigger phrases, and the top-level decision tree for "which subsystem does this question belong to?" Deliberately minimal to keep per-invocation context cost low.

**Reference files** (loaded on demand, only when the current question requires that depth):
- `references/icp-decision-tree.md` — the 5-tier ICP framework with qualification questions and boundary cases
- `references/messaging-selection.md` — the 30+ template library with selection rules per ICP × touch point
- `references/sequence-logic.md` — cadence rules, escalation criteria, breakup conditions
- `references/hubspot-procedures.md` — activity note formats, status transition rules, logging standards
- `references/competitive-plays.md` — the 5 battle cards with kill-shot talking points and objection responses
- `references/industry-stories.md` — the 22-story lookup indexed by vertical

**Loading rule:** `SKILL.md` routes each question to the one or two reference files it needs. A messaging-template question loads `messaging-selection.md` and (if ICP isn't specified) `icp-decision-tree.md`. A logging question loads only `hubspot-procedures.md`. The full reference library is never loaded at once.

This matters because in a naive single-prompt approach, you'd either (a) try to stuff all six subsystems into one mega-prompt (context bloat, degraded response quality), or (b) ask the user to repeatedly re-specify context every turn (terrible UX). Progressive disclosure is the middle path: deep knowledge available on demand, lightweight context by default.

---

## 4. Reference File Structure

Each reference file is authored to be loadable as standalone context — no cross-dependencies that would require loading multiple references for a single question. When a question *does* require multiple references (e.g. "draft a full sequence for an Enterprise HR Tech prospect evaluating Sendoso"), the skill loads them deliberately rather than preemptively.

**Example: sequence drafting request**
1. `SKILL.md` recognizes "draft a sequence" as the primary intent
2. Parses prospect info to determine if ICP is specified; if not, loads `icp-decision-tree.md` first to classify
3. Loads `messaging-selection.md` to select templates for the ICP × touch points
4. Loads `sequence-logic.md` to apply cadence rules
5. If the prospect mentions a specific competitor, additionally loads `competitive-plays.md`
6. Assembles the output with citations back to the reference files used

**Example: activity logging request**
1. `SKILL.md` recognizes "log a voicemail" as the primary intent
2. Loads only `hubspot-procedures.md`
3. Returns the exact note format to paste, with the 5 required fields populated from the conversation

A naive "all-in-one" skill would load all six references for both cases; this one loads what's needed.

---

## 5. Quantifiable Metrics

**Scale:**
- 6 subsystems unified behind a single natural-language interface
- 30+ messaging template variants accessible via selection logic
- 22 client stories indexed by industry for on-demand retrieval
- 5 competitor battle cards with objection-handling scripts
- 5 ICP tiers with qualification questions

**Efficiency:**
- Prep time per prospect call: ~30 min manual cross-reference → ~3 min skill invocation
- Template selection accuracy: determined by the selection rules in `messaging-selection.md`, not by rep judgment (removes the "I used the wrong template for this ICP" failure mode)
- Onboarding time for new SDRs: compressed further than the Enterprise Sales Intelligence doc alone achieved, because reps can ask the skill questions rather than re-reading the doc

**Operational:**
- Invokable by any RevSend team member with the skill installed in their Cowork session
- Deployed at the org level; no per-user configuration required

---

## 6. Technical Differentiation

**What makes this non-obvious:**

- **Progressive-disclosure architecture.** Not every question needs every reference file. A logging question doesn't need the ICP framework. A template question doesn't need sequence logic. The architecture explicitly optimizes for "load the minimum context required for this question" — the same pattern Anthropic uses for its own skills.
- **Deterministic routing in `SKILL.md`, generative depth in references.** The top-level routing logic ("is this an ICP question or a messaging question or a logging question?") is rule-based — the skill reads the question and picks the reference file deterministically. Only inside the references does the LLM do generative work like drafting actual message content. This keeps routing reliable while preserving flexibility where it matters.
- **Template selection rules instead of template selection by vibe.** A rep asking "which template should I use?" gets an answer derived from the 5-ICP × 6-touch-point matrix, not from the LLM's gut. If two templates are close, the rule chooses the one with the higher historical reply rate. Removing rep judgment from template selection removes the single biggest source of messaging variance across the team.
- **Composes on top of prior implementations.** This skill isn't a standalone — it references and invokes logic from the Enterprise Sales Intelligence system (#1), the ICP Framework (#2), the Messaging Framework (#3), the CRM Audit protocol (#6), and the HubSpot Operating System (#7). Each of those had to exist before this skill could be built; together they form a composable stack rather than six independent artifacts.

**What would break with a less-rigorous approach:**

- All-subsystem loading per invocation → context bloat and degraded response quality
- Generative routing (LLM decides which subsystem applies) → unreliable and non-reproducible
- No composition with upstream implementations → skill drifts from the source-of-truth documents as those evolve
- No deterministic template selection → rep variance reintroduces the "wrong template for this ICP" failure mode

**Senior-engineer design choices worth flagging:**

- Progressive disclosure over monolithic context
- Deterministic routing over generative routing at the top level
- Rule-based template selection over vibes-based selection
- Skill as a composition layer over upstream systems, not a replacement for them
- Org-level installation rather than per-rep configuration

---

## 7. Deployment Status

- **Status:** Production. Installed at the RevSend org level.
- **Distribution:** `outbound-sales-agent.skill` packaged artifact.
- **Trigger phrases:** Documented in `SKILL.md` and in onboarding materials — "draft a sequence," "what template," "how to log," "which ICP," "what's my play on [competitor]," and direct references to any of the underlying subsystem documents.
- **Maintainer:** Max Wilber. Skill source lives in the RevSend shared repository.
- **Update cadence:** Reference files are updated as the underlying subsystems evolve (new competitor → update `competitive-plays.md`; new ICP learning → update `icp-decision-tree.md`). Core `SKILL.md` rarely changes.

---

## 8. Business Outcomes for RevSend

- **Prep time per prospect call compressed ~90%.** ~30 min manual cross-reference → ~3 min skill invocation.
- **Template selection variance eliminated.** Rep-to-rep inconsistency in which template gets used for which ICP was a structural failure mode; the skill's rule-based selection removes it.
- **Institutional knowledge accessible by invocation.** A new SDR doesn't have to read all 615 lines of the Enterprise Sales Intelligence doc before their first prospect call — they can ask the skill questions as they come up.
- **Composable stack.** Each upstream implementation (master strategy, ICPs, messaging, CRM ops) gets more leverage because the skill makes them invokable rather than just readable.

---

## 9. Resume Bullet (Published)

> Designed and shipped a packaged Claude skill (`outbound-sales-agent`) that encapsulates a B2B SaaS company's complete outbound sales workflow — ICP classification, template selection, sequence management, CRM logging, competitive plays — behind a single natural-language interface, using progressive-disclosure architecture (minimal core SKILL.md + 6 reference files loaded on demand) to keep context costs low while preserving deep subsystem knowledge; compressed per-prospect prep time ~90% (30 min → 3 min) and eliminated template-selection variance across the SDR team.

---

## Reproducibility

The skill structure is reproducible from the repository: `SKILL.md` contains the routing logic, each reference file is a standalone subsystem document, and the composition with upstream implementations (#1, #2, #3, #6, #7) is explicit in the reference file cross-links. Any engineer inspecting the skill can trace routing → reference loading → output assembly step by step.
