# AI Implementations Portfolio

Maxwell Wilber | Seattle, WA | [LinkedIn](https://linkedin.com/in/maxwellwilber)

Between early 2025 and mid 2026 I built AI systems for two businesses. Beach City Baseball Academy is a youth sports facility that did $1.41M in 2025. RevSend is a B2B SaaS gifting platform. I worked with both as an independent consultant while teaching myself the technical side, and this is the record of what came out of it.

## How this is organized

Some of this work is software. Pipelines, API automations, packaged Claude skills, things that run and produce output on their own. The rest is strategy work: frameworks, playbooks, operating procedures. Both mattered to the businesses, but they are different kinds of work and they read better apart.

Ten systems, then nine strategy assets. If you want to know what I can build, start with the first section.

## Start here

Two of these have public code you can clone and run:

- **[client-data-cleaner](https://github.com/maxwellwilber-cpu/client-data-cleaner)** is a working version of the BCBA data warehouse below, running on generated data. 100% matching precision measured against known ground truth, zero wrong merges.
- **[checkpoint](https://github.com/maxwellwilber-cpu/checkpoint)** validates AI output against declared rules. It catches invented numbers and citations that point nowhere.

And one has a measured before and after: **cs-health-report** scored 100% on its eval set against 83.8% for baseline Claude without it. That's the number I'd point at first.

---

## Production systems

### cs-health-report (RevSend)

A daily Customer Success briefing for a 14 account book. It pulls from HubSpot, a MySQL product usage replica, and Stripe, scores each account on product usage, CS engagement and payment health, then renders a priorities-first PDF.

The design decision I care about: the scoring math is deterministic Python and the PDF layout is reportlab. Claude orchestrates the run and decides what to look at, but it never touches the numbers. Scoring is exactly the kind of job an LLM should not be doing, and keeping it out was the point.

Full spec: [implementations/13-cs-health-report/](implementations/13-cs-health-report/)

It fails closed on HubSpot, because without HubSpot there is no credible report. It fails open with degraded scoring on Stripe and MySQL, since those missing just makes the picture thinner.

I built a three case eval harness with pypdf assertion grading to check it. **100% pass rate, 31 of 31 assertions, against 83.8% for baseline Claude. Runs 44% faster, 27.9s versus 50.3s.** Shipped April 19, 2026 and installed at the org level.

### Client data warehouse (BCBA)

Six years of client history lived across four files that didn't talk to each other. Two CSV exports and two Apple Numbers files, with inconsistent formats, duplicate people, and fake birthdays. A business doing $1.41M a year could not answer "which clients have we lost."

I built a pipeline that unified all of it. 88,306 session records and 12,244 client records ingested. Phone numbers normalized across six separate columns, emails deduplicated, 40+ state format variations collapsed, 3,528 placeholder birthdays flagged rather than trusted. A name matching engine resolves the same person across files despite nicknames, typos and maiden names, using tiered rules with confidence scores. Sessions link to the right client at **99.97% accuracy**.

RFM scoring on top of that produced behavioral segments and surfaced **3,029 lapsed clients with $1.2M+ in historical spend**, ranked and with contact info attached. That became the reactivation campaign. The bigger win was that the business went from unable to ask behavioral questions at all to answering new ones off the same clean dataset.

A generalized version with public code is at [client-data-cleaner](https://github.com/maxwellwilber-cpu/client-data-cleaner). Full spec: [implementations/01-client-data-warehouse/](implementations/01-client-data-warehouse/)

### Outbound Sales Agent skill (RevSend)

A Claude skill that front-loads RevSend's product differentiators, buyer personas, competitive battle cards and messaging principles, so every invocation produces RevSend specific output instead of generic sales advice.

It's organized around four response modes: targeting and research, messaging and copy, pipeline and organization, strategy and planning. It references the actual stack the team uses and teammates by name rather than suggesting generic tools.

Full spec: [implementations/12-outbound-sales-agent-skill/](implementations/12-outbound-sales-agent-skill/)

Installed at the org level. **Used by 4 SDRs and both co-founders.** It cut my own per-prospect prep from about 30 minutes to about 3. Team-wide measurement is still pending, so I won't claim it.

### CRM audit and batch remediation (RevSend)

An automated audit of RevSend's HubSpot instance. It analyzed 612 contacts through paginated API searches, categorized them by engagement status, and executed batch updates to standardize lead statuses and create structured activity notes.

**550 batch updates with 0 failures.** 43 status updates, 54 voicemail activity notes created with contact associations, 11 duplicate contacts caught and updated instead of duplicated. Counts were cross-referenced across categories to confirm they reconciled to 612. Work that would have taken an SDR around 8 hours ran in about 15 minutes.

Built on the HubSpot API through an MCP server integration with Claude. Full spec: [implementations/06-crm-audit-remediation/](implementations/06-crm-audit-remediation/)

### Sponsor intelligence and competitive analysis (BCBA)

I scraped 26 competitor websites, structured 500+ sponsor prospects into a categorized workbook, and ran tier analysis across 7 leagues. The analysis turned up a gap: nobody was selling anything under $1,000, which meant small local businesses had no way in.

That finding became a product. I designed the full tier stack around it, including a $275/month Community Partner tier specifically to close that gap.

BCBA had no sponsorship program before this. It now has **21 partners, $50K+ raised in the first six months, and roughly $110K annualized run rate.** Chrome extension scraping, Python and openpyxl for the dedup and categorization, 14 tab workbook as the output.

### Brand asset production pipeline (BCBA)

An AI to design pipeline built on the Canva MCP server. Claude generates copy against the brand context document, picks a template that matches the content type, populates text and images through MCP tools, and exports an editable Canva design for review.

It collapsed copywriting and design from two steps into one. Used for social posts and most sponsor facing material from Q4 2025 on.

### PDF document generation (BCBA)

Python and ReportLab producing four branded document types: coach one-pagers, sponsorship decks, sponsor flyers, program overviews. Custom layouts, embedded images, QR codes through the qrcode library.

It removed the need for third party design tools on routine documents. Per-document production went from around two hours in Canva with manual edits to about ten minutes of directed iteration. The coach flyer went through three revision cycles in under half an hour.

### Sponsor content calendar automation (BCBA)

Every sponsor contract carries deliverables: posts, shoutouts, email features, event mentions. Tracking 21 of those by hand is how deliverables get missed, and missed deliverables are how sponsors churn.

I mapped each partner's contract terms to their cadence and automated the tracking. Python datetime validation against contract terms, Google Sheets as the backend, Zapier for advance notifications. It handles monthly, quarterly and event-based rules and surfaces anything overdue.

The business can now sell multi-year deals with confidence, because fulfillment is systematized rather than remembered.

### Apollo.io contact enrichment (RevSend)

A pipeline that enriches HubSpot contacts with verified emails, current job titles and LinkedIn URLs through Apollo.io's bulk matching API. HubSpot records go out, Apollo matches on name and company, enriched data comes back and writes to the contact record.

It removed the manual research step that had been capping outreach volume.

### Instagram content analysis (BCBA)

A scraping and analysis pipeline that pulled 51 Instagram posts through Chrome extension MCP, navigating the login wall and both the grid and Reels tabs, then structured them into a CSV across 8 analytical dimensions.

The report covers engagement rates, content types, posting timing and how sponsor integrations actually performed. Those findings feed the brand context document, so new posts go out with some evidence behind them instead of guesswork.

---

## Strategy and knowledge assets

Not software. These are documents, frameworks and operating procedures I built with AI as the execution engine. Several of them are behind the numbers in the section above.

**Enterprise sales intelligence system (RevSend).** A 615 line operational source of truth that consolidated six fragmented sales assets into one document: a 52 slide competitive deck, a 47 slide client use case deck, 125+ competitor case study URLs, a 27,000 word outreach playbook, and a 102 company prospect research list. Ten interconnected sections with relational cross references. It's the onboarding document for new SDR hires as the team scales. Full spec: [implementations/02-enterprise-sales-intelligence/](implementations/02-enterprise-sales-intelligence/)

**ICP framework (RevSend).** Five ranked buyer profiles with firmographic criteria, pain points, entry angles and objection handling, built from 22 existing client profiles cross referenced against the prospect research. Every message and sequence is calibrated against it.

**Multi-channel messaging framework (RevSend).** A template matrix, five ICPs by touch point, producing 30+ message variants. Each one carries a subject line, an opening hook mapped to buyer pain, a value prop tied to an ICP specific use case, and a note explaining why it works.

**Competitive battle cards (RevSend).** Five direct competitors distilled from a 52 slide deck into per-competitor cards with positioning, weaknesses to press on, and objection responses written as things you'd actually say out loud.

**Industry story lookup (RevSend).** 22 client stories extracted from a slide deck and indexed by industry, each with the stat to quote and the conversation moment to use it in. Discovery call prep dropped from about 20 minutes to about 5.

**HubSpot operating system (RevSend).** Lead status taxonomy across HubSpot's 8 built in statuses with criteria for every transition, standardized note formats, re-engagement rules, and duplicate resolution. This is what made the CRM audit above possible: you cannot automate a process nobody has defined.

**LinkedIn DM diagnostic (RevSend).** An audit comparing actual sent DMs against the messaging framework, which surfaced five recurring failure patterns. Each one comes with a rewritten example rather than a note saying to do better.

**Brand management hub (BCBA).** A 557 line brand context document built from a manual brand audit, website extraction, analysis of 51 Instagram posts, and stakeholder interviews. Nine sections covering voice, visual identity, audience and content strategy. Every downstream AI session pulls from it, which is what keeps the brand from drifting across channels. Content production went from around 45 minutes a piece to about 10.

**AI outreach plan and playbooks (BCBA).** 500+ prospects sorted into four tiers, five category specific pitch strategies, and a five touch follow up cadence. This ran the sponsorship drive that signed 21 partners against a target of 16.

---

## What I work with

Python (pandas, openpyxl, ReportLab, pytest), the Claude API and Claude Code, MCP server integrations (HubSpot, Apollo.io, Chrome, Canva), HubSpot and Apollo.io APIs, Zapier, Google Sheets, Vellum, regex and ETL work, RFM modeling, and eval driven development.

The thread through most of it: figure out what the business actually needs, design the system, direct AI to build it, then check the output against reality before anyone relies on it. The checking part is where most of the work is.

## Related

- **[client-data-cleaner](https://github.com/maxwellwilber-cpu/client-data-cleaner)** turns messy multi-source customer exports into one clean master list. Every merge logged, accuracy measured against ground truth.
- **[checkpoint](https://github.com/maxwellwilber-cpu/checkpoint)** validates AI output against rules you declare. Catches invented numbers, bad citations and placeholder text.
- **[fas-case-study](https://github.com/maxwellwilber-cpu/fas-case-study)** is a 27 node AI financial analysis pipeline with 73 validation checks, built on Vellum. 96.2/100 quality score on real client data.
- **[evs](https://github.com/maxwellwilber-cpu/evs)** is the Python framework that validates the FAS pipeline. 43 tests, caught 12 of 12 planted errors plus 3 real ones.

---

I build AI automations and agents for businesses, currently as an AI implementation specialist at a Seattle AI startup. [LinkedIn](https://linkedin.com/in/maxwellwilber) | [GitHub](https://github.com/maxwellwilber-cpu)
