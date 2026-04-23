# outbound-sales-agent — Packaged Claude Skill

**Built for:** RevSend (B2B SaaS corporate gifting platform)
**Deployed:** 2025
**Author:** Maxwell Wilber
**Package:** Claude skill — opinionated system prompt

---

## 1. Identity & Purpose

A Claude skill that functions as a dedicated outbound sales strategist for RevSend. It gives Claude deep, in-context knowledge of RevSend's product differentiators, ICP and buyer personas, competitive landscape, sales motion, and messaging principles — so every invocation produces RevSend-specific strategy, copy, and pipeline assets on demand rather than generic LLM output.

**Problem solved:** Before this skill existed, asking an LLM for help with RevSend outbound meant starting every conversation by explaining the product, the ICP, the competitive positioning, and the sales motion before any real work could happen. The LLM would then default to generic sales advice — "send gifts to build relationships" — instead of drawing on RevSend's specific differentiators (open-catalog Chrome extension, 60-second CRM-to-checkout, month-to-month pricing against annual-contract incumbents). The skill front-loads all of that context so every invocation starts from RevSend's specific reality.

**Users:** Currently used by 4 SDRs (including Max) and the 2 co-founders at RevSend. Installed at the org level; any team member can invoke it in Cowork and receive the same context-rich responses.

---

## 2. Invocation → Output

**Invocation:** Natural-language prompts in any Cowork session with the skill installed. The skill figures out which of four modes applies and responds accordingly:

- **Targeting & Research** — "Who should I go after?" "Build me a prospect list." "Score this account list."
- **Messaging & Copy** — "Write a cold email to this prospect." "Draft a sequence for this account." "Rewrite this LinkedIn DM."
- **Pipeline & Organization** — "Build me a campaign tracker." "Audit my current pipeline." "What should I do next with this contact?"
- **Strategy & Planning** — "Refine my ICP." "Build a playbook for this campaign." "Review my outbound strategy."

**Output:** The skill matches the output medium to the ask. Quick strategy or feedback questions get conversational answers. Substantive deliverables — prospect lists, email sequences, pipeline trackers, playbooks — get produced as files (`.xlsx`, `.md`, or `.docx` depending on the ask). Forcing a spreadsheet when a paragraph will do is an anti-pattern the skill is explicitly designed to avoid.

---

## 3. Skill Architecture

The skill is an engineered system prompt — not a wrapper around other scripts, not a multi-file reference loader. It is a single, carefully-structured body of context that shapes Claude's responses for every outbound-related task. The structure:

**Role framing.** Opens with "You are an outbound sales strategist and execution partner for RevSend" and explicitly rejects the "chatbot that spits out email templates" default. Shapes behavior before any content is loaded.

**RevSend product knowledge section.** Open catalog via Chrome extension, 60-second CRM-to-checkout, digital-to-handwritten notes, seat-based SaaS pricing ($19/mo Starter, month-to-month — a wedge against Sendoso/Reachdesk annual contracts), native HubSpot/Salesforce attribution, hybrid digital + physical sending. These differentiators are the substrate every piece of outreach has to connect back to.

**ICP and buyer personas.** Sweet spot (50–500 employees, U.S.-focused, Salesforce/HubSpot users, active SDR/AE motion). Four personas with pain points and messaging angles: SDR/BDR Leaders, Sales/Revenue Ops, CS/Retention Leaders, HR/People Ops.

**Competitive battle cards.** Per-competitor objection responses — for Sendoso/Reachdesk ("they have a bigger catalog," "we need global sending," "we already have a contract") and for Brilliant ("they offer white-glove creative services"). Every response is pre-mapped to a RevSend differentiator.

**Sales motion and discovery questions.** 14-day trial, $100 credits, 5-step sales flow from Discovery to Conversion. Five discovery questions specifically engineered to tilt toward RevSend's strengths (e.g., "How fast can a rep send a personalized gift today — from inside Salesforce/HubSpot?").

**Use case playbooks.** Stalled Deal Re-Engagement, Demo Show-Rate Boost, Closed-Lost Reactivation, Customer Expansion & Retention — each with trigger conditions, gift recommendations, and cadence outlines.

**Messaging principles.** "Lead with the prospect's world, not yours." "One idea per message." "Short > long." "Specificity beats cleverness." "Always include a clear, low-friction CTA." "Gift-forward when relevant." These act as implicit output filters on any copy the skill produces.

**Working-with-Max section.** Explicit instructions on how to interact: give real feedback on Max's own copy (not sycophantic agreement), match response to ask (file vs. chat), reference Max's actual stack (HubSpot, Instantly, Prosp_AI, Airtable) instead of suggesting generic new tools.

---

## 4. What Makes This Non-Obvious

**Opinionated over neutral.** A generic "sales AI assistant" prompt is deliberately neutral so it works for any company. This skill is the opposite — it only works for RevSend, by design. That specificity is what pulls output quality far above what a generic assistant produces.

**Response-mode calibration.** The skill explicitly refuses to force a deliverable when a conversational answer would serve better, and refuses to give a paragraph when the user clearly needs a file. Most skills default to one output mode regardless of ask; this one matches medium to request.

**Embedded anti-sycophancy instruction.** "If he shows you his own copy or strategy, give real feedback. Tell him what's weak and how to fix it." Explicitly prevents the skill from becoming a yes-machine — which would otherwise be the LLM's default for a user whose work it's reviewing.

**Tool-stack grounding.** References Max's actual tools (HubSpot for CRM, Instantly and Prosp_AI for outbound, Airtable for competitor matrix) and his actual teammates by name (Nick Natale on demos, Katherine McDermott on marketing, Jonathan Nahin on leadership). Recommendations stay inside his real workflow instead of drifting into "install a new tool" territory.

**Messaging principles as output filters.** "Specificity beats cleverness" and "one idea per message" aren't just advice to the user — they constrain the skill's own copy output. Bad output is filtered by construction rather than rejected post-hoc.

**Per-persona talk tracks.** Built-in one-liners for SDR/Sales Leaders, Customer Success, and RevOps/Finance so the skill can pivot tone and angle based on who the prospect is.

**What would break with a generic prompt:**

- Output would default to "send gifts to build relationships" rather than RevSend-specific positioning against incumbents
- Competitive objection responses would be invented from training data rather than aligned with Max's actual battle cards
- ICP output would reference generic B2B personas rather than RevSend's specific 50–500 employee, U.S.-focused, Salesforce/HubSpot sweet spot
- Copy output wouldn't pass the "one idea per message" filter
- Recommendations would suggest adding tools Max doesn't use

**Senior-engineer prompt-design choices worth flagging:**

- Organized around user intent (4 modes) rather than information taxonomy, because invocation framing drives better LLM behavior than reference-style organization would
- "How to Think About Your Role" framing ahead of knowledge content, because role conditioning shapes response patterns more effectively than content alone
- Explicit anti-sycophancy instruction, because LLMs default to agreement
- Explicit response-mode guidance, because LLMs default to their most elaborate available output
- Real names and real tools throughout, because specificity in the prompt produces specificity in the output

---

## 5. Quantifiable Metrics

**Adoption:**
- 4 SDRs (including Max) and 2 co-founders currently use it
- Installed at RevSend org level; adoption scales with the team

**Depth of embedded context:**
- 6 core RevSend differentiators with per-differentiator talking points
- 4 buyer personas with pain points and messaging angles
- 3 competitors with per-objection response scripts
- 4 use case playbooks with cadence outlines
- 5 discovery questions engineered to surface RevSend strengths
- 6 explicit messaging principles acting as copy-output filters
- 4 response modes with tailored output types

**Efficiency (based on Max's own usage):**
- Per-prospect prep time compressed from ~30 min manual cross-reference to ~3 min skill invocation
- Template selection, talk track selection, and objection responses are rule-driven by persona/competitor/use case rather than rep judgment — removes "wrong talk track for this prospect" errors by design

---

## 6. Technical Differentiation

**Prompt engineering over tool engineering.** This skill isn't an agent that calls tools — it's an engineered prompt that shapes LLM behavior through structure. The skill's value is entirely in how the context is organized and framed, not in any code that executes.

**Role conditioning before content.** The "How to Think About Your Role" section deliberately precedes the product knowledge. Behavioral framing shapes LLM responses more effectively than content knowledge alone; most prompt authors lead with content and bury role conditioning at the bottom.

**Opinionated positioning baked in.** Every product differentiator in the prompt is paired with the specific competitor wedge it creates. The LLM can't produce neutral positioning because the prompt doesn't give it a neutral frame to work from.

**Output mode as a first-class concern.** Most skills produce one type of output. This one has explicit guidance on when to produce a file, when to give a paragraph, when to brainstorm conversationally, and when to deliver a deep strategic document — calibrated to the ask rather than the skill's defaults.

---

## 7. Deployment Status

- **Status:** Production. In active use by the RevSend outbound team since 2025.
- **Distribution:** Claude skill installed at the RevSend org level.
- **Invocation:** Natural-language prompts in any Cowork session; no per-user configuration required.
- **Maintainer:** Max Wilber. Updates to skill content (new competitors, updated ICPs, new playbooks) propagate to all users on next invocation.

---

## 8. Business Outcomes for RevSend

- **RevSend-specific output from every invocation.** Copy, strategy, and deliverables are grounded in the actual product and ICP — not hallucinated from generic sales training data.
- **Per-prospect prep time compressed ~90% for Max's own usage.** ~30 min manual cross-reference → ~3 min skill invocation. Team-wide measurement pending as adoption scales.
- **Institutional knowledge made invokable.** A new rep doesn't need to read every internal doc before their first prospect call — they can ask the skill and get context-rich answers in real time.
- **Quality floor raised across the team.** Messaging principles ("one idea per message," "specificity beats cleverness") and embedded battle cards constrain output quality by design rather than by rep-by-rep review.
- **Competitive positioning defensible in any conversation.** Objection responses and competitive wedges are pre-built; reps don't have to remember them under pressure.

---

## 9. Resume Bullet (Published)

> Built a packaged Claude skill (`outbound-sales-agent`) for a B2B SaaS gifting platform — an opinionated system prompt that front-loads RevSend's product differentiators, ICP, competitive battle cards, sales motion, and messaging principles into a single invokable context, producing RevSend-specific outbound strategy, copy, and pipeline assets on demand rather than generic LLM output. Currently used by 4 SDRs and the 2 co-founders; compressed my own per-prospect prep time ~90% (30 min → 3 min) and removed variance in template/talk-track selection by encoding it as rule-driven logic in the prompt.

---

## Reproducibility

The skill is a system prompt. LLM invocations have inherent non-determinism, but the structural outputs (ICP classifications, objection responses, playbook cadences, messaging principle adherence) are constrained by the embedded rules and have been consistent across invocations in practice.
