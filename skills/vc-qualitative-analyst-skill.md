---
name: vc-qualitative-analyst
description: >
  Downstream agent that synthesizes a COMPANY.md qualitative brief to give meaning to
  vc-financial-analyzer handoff reports. Triggers when the user asks to "complete the VC
  package", "write the company brief", "add qualitative context", "make the financials
  make sense", or provides a vc_financial_results.md / financial data without a COMPANY.md.
  Handles three input states: (A) vc_financial_results.md only → research-first,
  (B) raw company data/pitch deck but no COMPANY.md → synthesize then verify,
  (C) both files present → enrich and cross-reference.
  Outputs COMPANY.md + an annotated vc_investment_brief.md that bridges both documents
  for the VC reader. Core analytical lenses: industry value chain position, industry
  status quo, geographic footprint, and government policy exposure.
---

# VC Qualitative Analyst Skill

> **Purpose:** Give the financial trend charts a *story*. The upstream skill answers
> "How is the engine running?" This skill answers "Where is the car going, who is driving,
> **where on the map are they**, and **which lane of the industry highway are they in**?"
>
> **Trigger phrases:** "complete the VC package", "write the company brief",
> "add qualitative context to financials", "make the numbers mean something",
> "research [company name] for VC", "I only have the handoff report",
> "generate COMPANY.md", or any state where `vc_financial_results.md` exists
> but `COMPANY.md` does not.

---

## Philosophy: Four Lenses, One Story

Financial signals are lagging indicators. To give them meaning, every section of
this brief is read through four lenses simultaneously:

```
LENS 1 — Value Chain Position
  Where does this company sit in its industry's production/delivery chain?
  Upstream (raw inputs), midstream (infrastructure/platform), or downstream (end user)?
  Who are they dependent on? Who is dependent on them?
  → This determines margin structure, leverage, and disruption vulnerability.

LENS 2 — Industry Status Quo
  What is the existing system the company is disrupting or plugging into?
  What are the incumbents, the legacy workflows, and the switching costs already in place?
  → This determines how fast adoption can realistically happen.

LENS 3 — Geographic Footprint (HQ + Customer Locations)
  Where is the company incorporated and operating from?
  Where are its current and target customers located?
  → Different answers = different labor cost, talent pool, and sales cycle realities.

LENS 4 — Government Policy & Regulatory Exposure
  Which governments have jurisdiction over the company's operations AND its customers?
  What policies (data, sector regulation, trade, subsidy, licensing) currently apply
  or are likely to apply in the next 2–3 years?
  → A 🟢 growth signal can flip 🔴 overnight if a regulation changes in the customer's country.
```

> "VCs invest in lines, not dots — but lines only matter if the destination is worth
> reaching, the road is legal to drive on, and the company controls enough of the
> supply chain to stay on it." The upstream skill draws the line. This skill validates
> the destination, the route, and the roadblocks.

**Core principle:** Every section of COMPANY.md must connect back to at least one
metric in `vc_financial_results.md` AND at least one of the four lenses above.

---

## Input State Detection (Run This First)

Before any research or writing, classify which state the user is in:

```
STATE A — Financial handoff only
  Files present:  vc_financial_results.md  ✓
  Files present:  COMPANY.md               ✗
  Files present:  raw company data          ✗
  → Action: Identify company name + industry from handoff → web research across
            all four lenses → write COMPANY.md → write vc_investment_brief.md

STATE B — Raw company data, no financial handoff
  Files present:  vc_financial_results.md  ✗
  Files present:  COMPANY.md               ✗
  Files present:  pitch deck / data room / user description  ✓
  → Action: Extract geo + industry chain position from provided data first →
            verify via web search → synthesize COMPANY.md

STATE C — Both documents present
  Files present:  vc_financial_results.md  ✓
  Files present:  COMPANY.md               ✓ (draft or partial)
  → Action: Check if geo and chain position are already documented →
            fill gaps → cross-reference all financial flags →
            flag any regulatory risk that contradicts a 🟢 financial signal
```

**Detection heuristic:** If `COMPANY_NAME` is visible in the financial results but no
company description exists → assume State A. If the user pastes a pitch deck or
describes the company in prose → State B. If both are uploaded → State C.

**Before writing anything**, answer these two questions from whatever input is available:
1. What industry value chain position does this company occupy? (one sentence)
2. What countries/regions does it operate in and sell to? (a list)

If you cannot answer both questions from the available input, they become your first
two research targets before any other section is drafted.

---

## Step 1 — Parse the Financial Handoff

Read `vc_financial_results.md` and extract these anchor points.
They will be referenced throughout COMPANY.md:

```
financial_anchors = {
    company_name:     ""   # from > Company: field
    data_range:       ""   # from > Data range: field
    revenue_latest:   ""   # Revenue (Latest Qtr) — Latest Value column
    revenue_signal:   ""   # 🟢/🟡/🔴
    growth_rate:      ""   # QoQ Growth Rate — Latest Value
    growth_signal:    ""   # signal emoji
    burn_rate:        ""   # Burn Rate — Latest Value
    runway_months:    ""   # Runway — Latest Value
    runway_signal:    ""   # signal
    burn_multiple:    ""   # Burn Multiple — Latest Value
    burn_signal:      ""   # signal
    gross_margin:     ""   # Gross Margin — Latest Value
    margin_signal:    ""   # signal
    valuation_range:  ""   # Credible range from Section 4
    auto_flags:       []   # all items from Section 5
    forecast_revenue: ""   # Revenue forecast end-value
    forecast_runway:  ""   # Runway forecast end-value
}
```

**Lens mapping — read each flag through the four lenses:**

| Financial Flag | Lens 1: Chain | Lens 2: Status Quo | Lens 3: Geo | Lens 4: Policy |
|---------------|--------------|-------------------|------------|----------------|
| 🔴 RUNWAY CRITICAL | Dependent on a single upstream supplier or platform draining cash? | How fast do incumbents in this space typically raise? | Is VC ecosystem thin in this HQ region? | Are licensing or compliance costs eating runway? |
| 🔴 HIGH BURN MULTIPLE | What chain node requires the most sales effort to unlock? | Are legacy customers hard to convert? | Is the sales team remote from the customer geo? | Does regulatory friction extend the sales cycle? |
| 🟢 GROWTH ACCELERATING | Did an upstream player open access or cut costs? | Did a status-quo incumbent fail or exit? | Did a new market/region open? | Did a favorable subsidy or mandate unlock demand? |
| 🔴 GROWTH DECELERATING | Did an upstream dependency tighten or reprice? | Are incumbents fighting back effectively? | Did a key customer geo enter a macro slowdown? | Did a regulation restrict the product or its customers? |
| 🔴 LOW GROSS MARGIN | Stuck at a low-value node in the chain? | Competing on price against a legacy player? | Are COGS elevated by local infrastructure costs? | Import/export tariffs or local content requirements? |
| 🟢 All signals green | What chain position creates this margin structure? | What status-quo inertia still lies ahead? | Which geos are untapped and at what entry cost? | What upcoming policy could accelerate or threaten this? |

---

## Step 2 — Research Protocol (States A and B)

### 2a. Research in This Exact Order

**Block 1 — Establish geo + chain position FIRST (non-negotiable)**

```
"[company name] headquarters location founded"
"[company name] customers OR clients geography"
"[company name] industry value chain OR supply chain position"
"[company name] crunchbase"          ← confirms HQ, stage, investors
```

Do not proceed to Block 2 until you know:
- Country/city of HQ
- Country/region of primary customers (not just "global")
- One-sentence description of where in the value chain they sit

**Block 2 — Market and competitive context**

```
"[company name] competitors"
"[industry] market size [customer region]"    ← use customer geo, not company HQ geo
"[industry] [customer region] regulation OR policy 2024 2025"
"[company name] techcrunch OR sifted OR the information"
```

**Block 3 — Policy and regulatory environment**

```
"[industry] regulation [customer country/region]"
"[industry] government policy [customer country/region] 2025 2026"
"[industry] [company HQ country] licensing OR compliance"
"[industry] [customer country] data privacy OR data localization"   (if software/data company)
```

**Block 4 — Founder and team**

```
"[founder name] linkedin OR background"
"[company name] team founding story"
"[founder name] previous company OR exit"
```

### 2b. Research Quality Rules

| Rule | Rationale |
|------|-----------|
| Always cite source URL per claim | VC diligence is adversarial — traceable facts only |
| Geo claims require a primary source | "Serves global customers" is not a geo claim |
| Policy claims require a government or reputable legal/news source | Blog opinions on regulation are not diligence |
| Market size must match the customer geo, not company HQ | A Taiwan company selling to Japan operates in Japan's market |
| Flag any unverifiable claim as `[UNVERIFIED — research needed]` | Honest gaps beat confident hallucinations |
| Cap total research at 8–10 searches before writing | Avoid research paralysis |

### 2c. Financial–Research Mapping (with geo and chain lenses)

| Financial Signal | Core Research Question | Geo Angle | Chain / Policy Angle |
|----------------|----------------------|-----------|-------------------|
| 🔴 RUNWAY CRITICAL | Is a round being raised? | Is the VC ecosystem in the HQ region deep enough for the round size needed? | Are compliance costs consuming cash before revenue arrives? |
| 🔴 HIGH BURN MULTIPLE | What is the GTM motion? | Does the customer geo require in-person / local relationship selling? | Does the chain position structurally require long enterprise cycles? |
| 🟢 GROWTH ACCELERATING | What caused the inflection? | Did a new geo open? New distribution partner? | Did an upstream partner open access? Did a policy mandate create demand? |
| 🔴 GROWTH DECELERATING | Market saturation or competition? | Did a key customer geo enter recession or tighten budgets? | Did a regulatory change restrict the product in the customer country? |
| 🔴 LOW GROSS MARGIN | Services component? Hardware? Infra? | Are COGS elevated by local infrastructure or import costs in customer geo? | Is the company stuck at a low-value node in the chain? |

---

## Step 3 — Write COMPANY.md

Write each section in this order. Each section requires:
- A **financial bridge** — sentence connecting to a metric from the handoff
- A **geo + chain note** — sentence connecting to Lens 3 or Lens 1/4

---

### Section 1: Company Identity & Value Chain Position

This is the new anchor section. It replaces a generic "about us" and gives the VC
an immediate map of where the company sits before any other context is read.

```markdown
# [Company Name] — Investment Brief

> **Stage:** [Seed / Series A / B / Growth]
> **Sector:** [e.g., B2B SaaS / Fintech / Healthcare AI]
> **HQ Location:** [City, Country]
> **Primary Customer Geographies:** [List actual countries/regions — never "global"]
> **Data Range Analyzed:** [from financial handoff]

## 1. Company Identity & Value Chain Position

**What they do (one sentence, no jargon):**
[Smart-outsider test. A non-specialist must understand this immediately.]

**Where they sit in the industry chain:**
[Describe position: upstream / midstream / downstream. Name the players
directly above and below them.]

Example for a B2B logistics SaaS:
"[Company] sits midstream in the logistics chain — between warehouse management
systems (upstream: SAP, Oracle WMS) and last-mile delivery networks (downstream:
DHL, local couriers). They sell to 3PL providers and regional freight forwarders
who sit between these two layers."

**Industry chain diagram (text):**

  [Upstream: suppliers / platforms / regulators the company depends on]
                          ↓
          ╔══════════════════════════╗
          ║   [COMPANY sits here]   ║
          ╚══════════════════════════╝
                          ↓
  [Downstream: customers / end users / distribution the company serves]

**Chain-position implications:**
- Dependent on: [what upstream players control that this company needs]
- Depended on by: [what downstream players need from this company]
- Margin structure: [why the chain position explains the gross margin %, not just the model]
- Disruption risk: [can an upstream player bypass them? can a downstream player build it themselves?]
```

**Financial bridge (required):**
> *"The [gross_margin]% gross margin [trend direction] is directly explained by
> the company's [midstream / downstream / upstream] chain position in [industry],
> where players at this node typically operate at [benchmark]% gross margin.
> [If below benchmark: the gap suggests pricing pressure from [adjacent chain player]
> or a services component not yet automated.]
> [If above: this reflects [specific chain leverage or moat at this node].)"*

---

### Section 2: Geographic Footprint & Regulatory Jurisdiction

This is the most important new section. Financial signals have no stable meaning
without knowing which governments have jurisdiction and what they are currently doing.

```markdown
## 2. Geographic Footprint & Regulatory Jurisdiction

### 2a. Operational Geography

| Dimension | Location(s) | Notes |
|-----------|------------|-------|
| HQ / Legal entity | [Country, City] | Primary legal and tax jurisdiction |
| Engineering / R&D | [Country, City] | Labor cost, talent pool, IP ownership |
| Sales offices | [Countries] | Local market access and relationships |
| Primary customer base | [Countries / Regions] | **Most important for policy analysis** |
| Target expansion markets | [Countries] | Future regulatory risk to model now |

### 2b. Regulatory Environment by Customer Jurisdiction

For EACH customer geography identified above, assess:

**[Customer Country/Region 1 — e.g., Japan]**
- **Sector regulation:** [Specific laws or regulators governing this industry here]
- **Data policy:** [e.g., APPI, data localization requirements, cross-border transfer rules]
- **Market access requirements:** [Licensing, local partnership mandates, foreign ownership caps]
- **Current policy trend:** [Tightening / Stable / Loosening — with a specific example or citation]
- **Government incentive / subsidy:** [Programs that could accelerate demand from this geo]
- **Risk horizon:** [Specific pending legislation or scheduled regulatory review]

**[Customer Country/Region 2 — e.g., EU]**
[Same structure repeated]

**[HQ Country — assess separately if different from customer geo]**
- **Export controls:** [Any restrictions on selling the product or data to current customer geos]
- **Foreign investment screening:** [Rules that may affect VC from certain countries investing here]
- **Tax / IP regime:** [R&D tax credits, IP box regimes, repatriation rules relevant to VC return]

### 2c. Geo-Financial Connection

[Write 3–5 sentences explicitly answering:
- Why does this geo mix produce this specific revenue trend?
- Why does selling into [customer country] explain the burn rate or burn multiple?
- Is the runway sufficient to survive a regulatory review in the primary customer market?
- Which geo is driving growth, and is that geo's policy environment stable?]
```

**Financial bridge (required):**
> *"The [growth_rate] QoQ growth [in the context of selling into customer geo] must
> be read against [specific regulatory or macro condition in that geo].
> [If 🟢: this growth is occurring despite / because of [specific policy condition].]
> [If 🔴: deceleration may reflect [regulatory or macro headwind in customer geo],
> not just internal execution.]
> The [runway_months] runway [is / is not] sufficient to navigate [specific process —
> e.g., licensing review, data compliance audit, procurement cycle] if required in
> [customer country]."*

---

### Section 3: Industry Status Quo & Tailwinds

```markdown
## 3. Industry Status Quo & Tailwinds

**The Legacy Baseline (by customer geography):**
[How is this problem solved today in each major customer geo?
The status quo often differs by country — name the specific incumbents or
processes in the primary customer geo, not just globally.]

**The Macro Shift:**
[Specific technology or behavior change driving disruption. 1–2 paragraphs. Cite sources.
Note whether the shift is happening at the same speed across all customer geos or
faster/slower in specific markets — and why.]

**Market Size (by customer geography):**
| Geography | TAM | SAM | SOM | Methodology |
|-----------|-----|-----|-----|-------------|
| [Primary customer geo] | $XB | $XM | $XM | [bottoms-up / top-down, source] |
| [Secondary geo] | $XB | $XM | $XM | [source] |

Important: Use the market size of the customer geography, not the company's HQ country.

**Status quo incumbents in primary customer geo:**
[Name specific legacy players — local ones matter more than global brand names here.]

**Chain disruption angle:**
[Is the company disrupting an existing node in the chain, or creating a new node?
Disruption = it threatens an existing player's revenue.
New node = it extends the chain and creates a dependency others didn't have before.
Each has different competitive dynamics and incumbent response patterns.]
```

**Financial bridge (required):**
> *"The [gross_margin]% gross margin [trend] is consistent with [SaaS / marketplace /
> services / hardware] economics in [customer geo], where best-in-class peers in this
> segment operate at [benchmark]%.
> [If the company is displacing a legacy player that charged via a different model,
> explain the pricing transition and its expected impact on the margin trajectory.]*"

---

### Section 4: Moat & Competitive Landscape

```markdown
## 4. Moat & Competitive Landscape

**Direct Competitors (prioritize those in primary customer geo):**
| Competitor | HQ | Stage | Key Differentiator vs [Company] |
|-----------|-----|-------|-------------------------------|
| [Name] | [Country] | [Series X] | [How they differ] |

Note: Local competitors in the customer geography are often more dangerous than
well-known global names. A local player with regulatory licenses, local language
support, and established enterprise relationships is a formidable moat for others.

**Indirect Competitors / Status Quo Alternatives:**
[What do customers in [customer geo] actually use today if this product doesn't exist?
The alternative may be culturally or operationally specific to that market —
e.g., in Japan, many enterprise workflows run on fax and email longer than in other markets.]

**Defensibility (Moat):**
- [ ] Network effect (note: network effects are often local — cross-geo networks are rare)
- [ ] Proprietary data loop (note: data collected in [customer geo] may be subject to local data sovereignty laws)
- [ ] High switching cost / deep integration (specifically with [local systems or platforms in customer geo])
- [ ] Regulatory moat (licensed in [geo] when competitors are not yet — first-mover in compliance)
- [ ] Deep tech / replication lag
- [ ] Local brand / distribution (especially relevant in relationship-driven markets: Japan, Korea, SE Asia, MENA)
- [ ] Chain position lock-in (controls a node that upstream and downstream players cannot easily bypass)
```

**Financial bridge (required):**
> *"The [burn_multiple]× burn multiple [trending direction] suggests the GTM motion
> is [efficient / costly]. In [customer geo], [enterprise sales / PLG / channel]
> motions at this chain position typically run [benchmark range].
> [If above benchmark: this may reflect the structural cost of [local relationship
> building / regulatory compliance / local procurement cycles] in [geo] — which
> is a real cost, but also creates a moat once established.]
> [If below: the company may have a distribution moat via [specific local channel]
> that competitors would need years to replicate.]*"

---

### Section 5: Go-To-Market Strategy

```markdown
## 5. Go-To-Market Strategy

**Motion:** [Direct enterprise sales / Product-led growth / Channel / Hybrid]

**Why this motion fits this geography:**
[GTM motions are not universal — they must match the buying culture of the customer geo.
Explain the cultural fit or flag the mismatch.

Examples:
- Enterprise direct sales is necessary in Japan because procurement requires long relationship-building. PLG rarely works as primary motion in Japan.
- PLG works well in US/EU SaaS because developers self-serve and can expense tools.
- Channel partnerships are often required in regulated industries (e.g., fintech in SE Asia) because local licensed intermediaries control market access.

If the current motion doesn't match the customer geo's buying culture, flag it as a risk.]

**Ideal Customer Profile (ICP):**
[Job title] at [company size and type] in [specific country or region],
experiencing [specific trigger event or regulatory requirement].

Example:
"Head of Compliance at 500–5000 person financial institutions in Singapore and
Hong Kong, navigating MAS and SFC regulatory requirements that went into effect in 2024."

**Customer Acquisition:**
[How do they find and close customers? Named channels with rough CAC.
Is the channel local (referrals, local resellers, trade associations) or
global (inbound, SEM)? Local channels are slower to build but harder to replicate.]

**Current Traction by Geography:**
| Geography | Customer Count / ARR | Notes |
|-----------|--------------------|----|
| [Geo 1] | [number or range] | [logo quality, deal size range] |
| [Geo 2] | [number or range] | [early / pilot / established] |

**Geographic expansion plan:**
[Which geo is next? What is the market entry strategy?
What regulatory hurdle or chain-access requirement must be cleared before entering?
What is the cost (time + cash) to establish the local presence needed to sell there?]
```

**Financial bridge (required):**
> *"The burn multiple of [burn_multiple]× implies roughly $[derived] in burn per
> $1 of new ARR added. This [is / is not] consistent with a [GTM motion] selling
> into [customer geo], where [benchmark context — e.g., Japan enterprise sales
> structurally requires 18–24 month cycles with local partner support, which elevates
> burn multiples above PLG benchmarks even at efficient companies].
> [If entering a new geo: the burn multiple is likely to [increase / stay flat] during
> the expansion period before local revenue offsets the entry cost.]*"

---

### Section 6: Founder-Market Fit & Team

```markdown
## 6. Founder-Market Fit & Team

**Why this team:**
[Specific unfair advantage — prior domain experience, local market relationships,
technical depth, language / cultural access to the primary customer geo.
Avoid generic "passionate team" language.]

**Geo-specific founder advantage:**
[Does the founding team have authentic access to the primary customer geography?
This is one of the most underrated signals in cross-border investing.

A founder who grew up in the customer country, worked there, or has deep local
relationships has an advantage that is very hard to replicate and very hard to fake.
Flag explicitly if this advantage is absent — it becomes a key hiring risk and a
realistic cap on how fast the company can penetrate the primary customer market.]

**Chain-specific expertise:**
[Does the team have deep domain knowledge of the specific chain position they occupy?
A logistics SaaS founder who ran a 3PL knows exactly what the customer needs and
which upstream integrations matter. A founder who never worked in logistics is
selling into the chain blind.]

**Founder backgrounds:**
| Name | Role | Relevant Prior Experience | Customer Geo Access | Chain Expertise |
|------|------|--------------------------|-------------------|----------------|
| [Name] | CEO | [Specific company, role, outcome] | [Native / Established / Building] | [Deep / Functional / Learning] |
| [Name] | CTO | [Specific technical depth] | [Relevant / N/A] | [Relevant / N/A] |

**Key hires and gaps:**
[Current team coverage. Specifically flag:
1. Is there a senior person who is fluent in the customer geography's business culture?
2. Is there someone with regulatory/compliance expertise for the primary customer geo?
3. What is the most critical missing hire for the next phase of geo expansion?]
```

**Financial bridge (required):**
> *"The [revenue_signal] revenue trajectory and [burn_signal] burn discipline across
> [data_range], in the context of selling into [customer geo], suggests the team
> [has / has not yet] demonstrated the ability to acquire and retain customers in
> a market that [describe specific difficulty: relationship-driven, regulatory-heavy,
> incumbent-dominated, procurement-slow, etc.].
> This [de-risks / raises questions about] the credibility of the geographic expansion
> plan and the burn multiple trajectory during that expansion."*

---

### Section 7: Key Risks & Open Questions

```markdown
## 7. Key Risks & Open Questions

### 7a. Financial Risks (from upstream handoff)
[Paste each 🔴 flag from Section 5 of vc_financial_results.md, with qualitative
root-cause analysis through the geo and chain lenses.]

Example:
"🔴 RUNWAY CRITICAL — The [runway_months] runway must be read against [customer geo]'s
procurement cycles, which average [X months] for [company/deal size]. If the pipeline
is weighted toward large enterprise or government customers in [geo], the effective
cash-to-revenue conversion lag may exceed the available runway."

### 7b. Regulatory Risks (by customer jurisdiction)
[For each customer geography, list the top 1–2 policy risks from Section 2b.
Rate each: 🔴 Active threat / 🟡 Emerging risk / 🟢 Monitoring only]

| Jurisdiction | Policy Risk | Likelihood | Timeline | Impact if Triggered |
|-------------|------------|-----------|----------|---------------------|
| [Geo] | [Specific regulation or pending bill] | 🔴/🟡/🟢 | [Date or window] | [Revenue, operational, or valuation impact] |

### 7c. Value Chain Risks
[What happens if an upstream dependency raises prices, closes API access, or
builds competing functionality? What happens if a downstream customer is acquired,
or pivots away from needing this product? Name the specific players.
Rate concentration: is >30% of revenue or cost dependent on a single chain player?]

### 7d. Research Gaps
[List each [UNVERIFIED — research needed] item from earlier sections.]

### 7e. Diligence Questions for the VC
1. [Question derived from 🔴 runway flag + customer geo procurement cycle]
2. [Question about regulatory compliance status in primary customer geo]
3. [Question about upstream chain dependency concentration]
4. [Question about local competitors not well-documented online — ask founders directly]
5. [Question about founding team's authentic relationship depth in customer geo]
6. [Question about what specific policy event could double or halve TAM in primary geo]
```

---

## Step 4 — Write vc_investment_brief.md

The bridge document. One file, complete picture for the VC.

```markdown
# VC Investment Brief — [Company Name]
*Synthesized from: vc_financial_results.md + COMPANY.md*
*Generated: [date]*

---

## Executive Summary (TL;DR)

[4–5 sentences max. Must include: chain position + geo + financial signal + key risk.]

"[Company] is a [chain position, e.g. midstream logistics infrastructure] company
headquartered in [HQ city] selling to [customer geos]. It addresses [problem] via
[GTM motion]. Across [data_range], revenue has grown [growth_rate] QoQ with a
[burn_multiple]× burn multiple [trending direction].
The primary geo/policy risk is [specific regulatory or market-access risk in customer geo].
The key diligence question: [most important open question — usually about geo access,
chain dependency, or regulatory exposure]."

---

## Signal Cross-Reference Table

| Financial Signal | Qualitative Explanation | Geo / Chain Lens | Confidence |
|-----------------|------------------------|-----------------|------------|
| [Revenue trend] | [Product / market reason] | [Which geo is driving this? Is the chain node expanding?] | 🟢/🟡/🔴 |
| [Burn multiple] | [GTM efficiency reason] | [Does the customer geo's buying culture explain the level?] | |
| [Gross margin]  | [Business model explanation] | [Does the chain position explain the margin structure?] | |
| [Runway]        | [Fundraising context] | [Is runway enough for next geo's regulatory process or procurement cycle?] | |
| [Growth rate]   | [Market expansion reason] | [Which geo is driving growth? Is that geo's policy environment stable?] | |

Confidence key: 🟢 Verified from primary source | 🟡 Inferred from research | 🔴 Unverified

---

## Policy Watch List

[This table is the senior VC's edge. It captures every regulatory risk across all
customer and operational geos so nothing surprises after term sheet.]

| Jurisdiction | Regulation / Policy | Status | Est. Timeline | Impact on [Company] |
|-------------|---------------------|--------|--------------|---------------------|
| [Geo] | [Specific law or pending bill] | Active / Pending / Proposed | [Date or window] | [Revenue risk / acceleration opportunity] |

---

## VC Conviction Checklist

Score each dimension 1–5. This is diligence, not a pitch.

| Dimension | Score (1–5) | Evidence |
|-----------|------------|---------|
| Problem severity | | |
| Market size — credible, in customer geo | | |
| Industry chain position — defensible | | |
| Geo-culture fit of GTM motion | | |
| Competitive moat in customer geo | | |
| Regulatory exposure — manageable | | |
| Team geo access — authentic | | |
| Team chain expertise — deep | | |
| Financial health | | |
| **Composite** | **/45** | |

Scoring guide:
5 = Best-in-class evidence found
4 = Strong signal, minor gaps
3 = Mixed / inconclusive
2 = Weak signal or concerning trend
1 = Red flag or missing entirely

---

## Recommended Next Steps

- [ ] [Specific diligence action per 🔴 financial flag + geo context]
- [ ] [Verify regulatory compliance status in [primary customer geo]]
- [ ] [Map upstream chain dependencies — what is the concentration risk?]
- [ ] [Reference check: founder's claimed relationships in [customer geo]]
- [ ] [Customer interview target: [ICP] in [geo] — ask about alternatives and switching cost]
- [ ] [Legal review: foreign ownership / export control rules if VC is from [different country]]
- [ ] [Policy watch: schedule review of [specific pending regulation] in [geo] before close]
```

---

## Step 5 — Output File Convention

```
vc_package/
  vc_financial_results.md      ← from upstream skill (quantitative)
  vc_analysis_charts.zip       ← from upstream skill (charts)
  COMPANY.md                   ← from this skill (qualitative, 7 sections)
  vc_investment_brief.md       ← from this skill (synthesis + policy watch list)
```

---

## Reliability Rules (Read Before Every Run)

| Rule | Why |
|------|-----|
| **Never skip geo detection.** If you don't know where the customers are, stop and research first. | Market size, regulatory risk, and GTM benchmarks are all geo-specific. |
| **Never use "global" as a customer geography.** Break it down to actual countries or regions. | "Global customers" is a pitch claim. Diligence requires specifics. |
| **Market size must match the customer geo, not the HQ geo.** | A Taiwan company selling to Japan operates in Japan's market, not Taiwan's. |
| **Policy risks must name a specific law or regulatory body, not just "regulatory risk."** | VCs need to know exactly what to track and when to worry. |
| **The chain diagram must name actual upstream and downstream players.** | Saying "midstream" without naming adjacent players is an incomplete map. |
| **Never hallucinate a number.** Flag as `[UNVERIFIED]` if unsourced. | Fabricated numbers destroy trust in adversarial diligence. |
| **Never fabricate competitor names.** Only list companies found in research. | A false competitive landscape is worse than an incomplete one. |
| **Never fabricate founder credentials.** Write "not publicly confirmed — verify directly." | Misrepresenting a team is a diligence failure. |
| **Financial bridges and geo+chain notes are required in every section.** | Without them, the qualitative and quantitative remain two separate documents. |
| **State A: research Block 1 before anything else.** Geo + chain first. | You cannot correctly interpret any financial signal without knowing where and in which chain position. |
| **State B: verify pitch deck claims against research.** Pitch decks are marketing. | Founders optimize for narrative. Research optimizes for truth. |
| **State C: surface contradictions.** If COMPANY.md claims regulatory-friendly market but research shows a pending restriction, flag it prominently. | Contradictions between narrative and reality are the most valuable diligence finding. |
| **Always populate Section 7.** Even if all signals are green. | There are always risks. An empty risk section signals an incomplete brief. |

---

## Iteration Log

| Version | Date | Change |
|---------|------|--------|
| v1.0 | 2026-05-17 | Initial skill — 3-state input detection, 6-section COMPANY.md, financial bridge sentences, reliability rules |
| v1.1 | 2026-05-17 | **Major revision per senior mentor feedback** — Added four analytical lenses (value chain position, industry status quo, geographic footprint, government policy exposure) as the philosophical core. Section 1 rewritten as "Company Identity & Value Chain Position" with chain diagram and disruption risk analysis. Section 2 replaced with "Geographic Footprint & Regulatory Jurisdiction" covering all customer geos with per-jurisdiction regulatory assessment. Lens-mapping table added to Step 1. Block 1 geo+chain research made mandatory before all other research. All 7 sections now require both a financial bridge AND a geo+chain note. GTM section requires geo-culture fit analysis. Team section requires geo access + chain expertise assessment. Section 7 expanded with dedicated regulatory risk table, chain risk subsection, and geo-specific diligence questions. vc_investment_brief.md extended with Policy Watch List, geo/chain column in cross-reference table. Conviction checklist updated to /45 with geo-specific dimensions. Reliability rules updated. |
| v1.x | — | *(your next iteration here)* |

---

*End of skill document. Edit freely — this is your living reference.*
*Pair with: vc-financial-analyzer (upstream), research-analyst (optional parallel run)*
