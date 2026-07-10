# Lloyds Banking Group — Consumer Lending (Homes) Platform

## AI/ML & Agentic AI Business Use Cases

> A practitioner-oriented catalogue of high-value use cases for the **Consumer Lending – Homes** (mortgages / home loans) platform. It is organised in four layers that build on one another:
>
> 1. **Core Machine Learning** — the predictive backbone (risk, propensity, pricing).
> 2. **Market Mix Modelling (MMM)** — marketing & commercial spend effectiveness.
> 3. **Time-Series Forecasting** — demand, arrears and portfolio dynamics over time.
> 4. **Agentic AI** — an orchestration layer *on top of* the models that solves real, day-to-day problems for consumers.
>
> Every use case is written as a **business problem → business value → data & modelling approach → KPIs / success metrics → risks & controls**, so it can be taken into a discovery or prioritisation workshop as-is.

---

## Table of Contents

- [Context & Guiding Principles](#context--guiding-principles)
- [Value-at-a-glance summary](#value-at-a-glance-summary)
- [Layer 1 — Core Machine Learning Use Cases](#layer-1--core-machine-learning-use-cases)
  - [ML-1: Affordability & Probability-of-Default (PD) scoring](#ml-1-affordability--probability-of-default-pd-scoring)
  - [ML-2: Mortgage application → completion propensity (conversion)](#ml-2-mortgage-application--completion-propensity-conversion)
  - [ML-3: Product & rate personalisation / retention at maturity](#ml-3-product--rate-personalisation--retention-at-maturity)
  - [ML-4: Early financial-difficulty (pre-arrears) risk model](#ml-4-early-financial-difficulty-pre-arrears-risk-model)
- [Layer 2 — Market Mix Modelling Use Cases](#layer-2--market-mix-modelling-use-cases)
  - [MMM-1: Mortgage acquisition marketing effectiveness & budget optimisation](#mmm-1-mortgage-acquisition-marketing-effectiveness--budget-optimisation)
  - [MMM-2: Broker vs. direct channel mix & intermediary incentive optimisation](#mmm-2-broker-vs-direct-channel-mix--intermediary-incentive-optimisation)
- [Layer 3 — Time-Series Forecasting Use Cases](#layer-3--time-series-forecasting-use-cases)
  - [TS-1: Mortgage demand & application-volume forecasting](#ts-1-mortgage-demand--application-volume-forecasting)
  - [TS-2: Arrears & impairment (ECL) early-warning forecasting](#ts-2-arrears--impairment-ecl-early-warning-forecasting)
  - [TS-3: Prepayment / redemption & balance run-off forecasting](#ts-3-prepayment--redemption--balance-run-off-forecasting)
- [Layer 4 — Agentic AI Use Cases (Consumer-facing)](#layer-4--agentic-ai-use-cases-consumer-facing)
  - [AG-1: "Home Buying Copilot" — end-to-end mortgage journey assistant](#ag-1-home-buying-copilot--end-to-end-mortgage-journey-assistant)
  - [AG-2: "Affordability & Deposit Coach" — get mortgage-ready agent](#ag-2-affordability--deposit-coach--get-mortgage-ready-agent)
  - [AG-3: "Rate & Remortgage Guardian" — proactive switching agent](#ag-3-rate--remortgage-guardian--proactive-switching-agent)
  - [AG-4: "Financial Resilience Companion" — arrears prevention & support agent](#ag-4-financial-resilience-companion--arrears-prevention--support-agent)
  - [AG-5: "Document & Application Concierge" — friction-removal agent](#ag-5-document--application-concierge--friction-removal-agent)
- [How the layers connect](#how-the-layers-connect)
- [Data foundations](#data-foundations)
- [Responsible AI, regulation & governance](#responsible-ai-regulation--governance)
- [Suggested delivery roadmap](#suggested-delivery-roadmap)

---

## Context & Guiding Principles

The **Consumer Lending – Homes** platform is responsible for helping people buy, own and stay in their homes: first-time buyers, home-movers, remortgagers, buy-to-let and later-life lending. It sits at the intersection of three commercial goals and one regulatory obligation:

- **Grow responsibly** — originate quality mortgage lending at target margin.
- **Retain & deepen** — keep customers at maturity and grow lifetime value.
- **Operate efficiently** — reduce cost-to-serve and time-to-decision.
- **Do right by the customer** — evidence **Consumer Duty** good outcomes, fair value, and support for those in financial difficulty.

Every use case below is deliberately mapped to **at least one consumer outcome**, not only to a P&L line, because under the FCA **Consumer Duty** the two must be delivered together.

**Design principles used throughout:**

- **Human-in-the-loop by default** for any lending, pricing or forbearance *decision*. AI recommends and drafts; accountable humans (or governed decision engines) decide.
- **Explainability first** — models used in credit decisions must produce reason codes (SHAP / scorecard points) to satisfy adverse-action and Consumer Duty requirements.
- **Fairness & inclusion** — models are monitored for disparate impact across protected characteristics; the goal is *widening* responsible access to home ownership, not narrowing it.
- **Agentic AI is an orchestration layer, not a new risk engine** — agents call the governed ML/MMM/TS models and approved knowledge sources; they never invent credit or pricing decisions.

---

## Value-at-a-glance summary

| ID | Use case | Type | Primary business value | Primary consumer value |
|----|----------|------|------------------------|------------------------|
| ML-1 | Affordability & PD scoring | Core ML (classification) | Lower expected credit loss, faster & more consistent decisions | Fairer, faster, more transparent decisions |
| ML-2 | Application→completion propensity | Core ML (classification) | Higher conversion, lower wasted cost per completion | Fewer drop-outs; help offered where they get stuck |
| ML-3 | Product/rate personalisation & retention | Core ML (uplift/propensity) | Higher retention & NIM at maturity | The right product at the right time, less overpaying |
| ML-4 | Early financial-difficulty model | Core ML (classification) | Lower arrears & impairment, Consumer Duty evidence | Early, empathetic help *before* missing a payment |
| MMM-1 | Acquisition marketing MMM | Market Mix Modelling | Reallocate marketing spend for higher ROI | Relevant, less wasteful marketing |
| MMM-2 | Channel & broker mix MMM | Market Mix Modelling | Optimise channel/incentive economics | Guidance through preferred channel |
| TS-1 | Demand / volume forecasting | Time series | Capacity, funding & pricing planning | Shorter wait times, consistent service |
| TS-2 | Arrears / ECL early-warning | Time series | Provisioning accuracy, capital efficiency | Portfolio-level support planned ahead of shocks |
| TS-3 | Prepayment / run-off forecasting | Time series | Balance-sheet & retention planning | Proactive retention offers before they leave |
| AG-1 | Home Buying Copilot | Agentic AI | Conversion + cost-to-serve | Confidence & clarity through a stressful journey |
| AG-2 | Affordability & Deposit Coach | Agentic AI | Future pipeline of qualified applicants | Realistic plan to become mortgage-ready |
| AG-3 | Rate & Remortgage Guardian | Agentic AI | Retention, reduced attrition | Avoid SVR/overpaying; timely switching |
| AG-4 | Financial Resilience Companion | Agentic AI | Lower arrears, Consumer Duty outcomes | Non-judgemental help in hard times |
| AG-5 | Document & Application Concierge | Agentic AI | Lower processing cost, faster time-to-offer | Less paperwork stress, fewer re-requests |

---

# Layer 1 — Core Machine Learning Use Cases

These are the **predictive foundations**. They are supervised models trained on historical lending and behavioural data. They can deliver value on their own (embedded in the origination and servicing platforms) and later become the "tools" the agentic layer calls.

---

## ML-1: Affordability & Probability-of-Default (PD) scoring

**Business problem**
Credit decisioning must simultaneously (a) avoid lending to people who cannot sustainably afford the loan (protecting them and the bank) and (b) avoid wrongly declining creditworthy applicants (lost growth, poor outcome, potential fairness issues). Traditional scorecards can be rigid, slow to recalibrate, and weak at capturing non-linear interactions (e.g. income volatility × cost-of-living × LTV).

**Business value**
- Reduced **expected credit loss (ECL)** through sharper rank-ordering of default risk.
- Higher **automated decision rate** (fewer manual referrals) → lower cost-to-serve and faster offers.
- More consistent, auditable, and **fairer** decisions with explicit reason codes.
- Better calibrated **risk-based pricing** and capital (input to IRB PD where permitted).

**Consumer value**
- Faster, more transparent yes/no with clear reasons; fewer "computer says no" experiences.
- Protection from being lent unaffordable amounts (Consumer Duty "avoiding foreseeable harm").

**Data & modelling approach**
- **Target:** default within 12–24 months (90+ DPD or default definition), plus a separate affordability-stress target.
- **Features:** applicant income & income stability, existing credit commitments, bureau attributes, LTV/LTI, deposit source, property type, stress-tested payment-to-income, open-banking cash-flow features (with consent).
- **Models:** regularised logistic regression / scorecard as the explainable baseline; gradient-boosted trees (XGBoost/LightGBM) for lift; **SHAP** for reason codes; **calibration** (Platt/isotonic) so scores map to true PD.
- **Validation:** out-of-time testing, PSI stability monitoring, fairness testing (approval-rate and error-rate parity across protected groups).

**KPIs / success metrics**
Gini / AUC and KS on out-of-time sample; ECL reduction at constant approval rate; auto-decision rate; decline-reason coverage; fairness disparity within agreed tolerance; override rate.

**Risks & controls**
Model risk governance (SR 11-7-style / PRA SS1/23), adverse-action explainability, bias monitoring, human review of edge cases, challenger scorecard retained for governance.

---

## ML-2: Mortgage application → completion propensity (conversion)

**Business problem**
A large share of mortgage applications never complete — customers drop out at DIP, document upload, valuation, or between offer and completion. The business cannot tell *which* live applications are at risk of stalling, or *why*, so intervention is reactive and generic.

**Business value**
- Higher **application-to-completion conversion**, directly increasing lending volume from existing demand (cheaper than buying new leads).
- Smarter allocation of scarce underwriter / broker-support time to the applications most likely to complete *with help*.
- Reduced pipeline leakage and wasted processing cost.

**Consumer value**
- Proactive help exactly where they get stuck instead of silent abandonment.
- Shorter, less stressful journey to offer/completion.

**Data & modelling approach**
- **Target:** completion within N days of application.
- **Features:** journey/event telemetry (stage timestamps, idle time, re-requested docs), applicant profile, product, channel, external factors (rate moves), prior contact history.
- **Models:** gradient-boosted classifier with time-to-event framing (survival analysis / discrete-time hazard) to predict *both* likelihood and expected stall stage; **uplift modelling** to estimate which interventions (call, reminder, doc help) actually change the outcome.
- **Serving:** near-real-time scoring on the live pipeline feeding case-management prioritisation.

**KPIs / success metrics**
Conversion uplift vs. control, cost per completion, pipeline cycle time, intervention efficiency (incremental completions per contact).

**Risks & controls**
Avoid "pressure selling" — interventions must be supportive, not coercive; suppression rules for vulnerable customers; A/B test all nudges.

---

## ML-3: Product & rate personalisation / retention at maturity

**Business problem**
When a fixed-rate deal ends, customers either **product-transfer**, **remortgage away**, or lapse onto the higher Standard Variable Rate (SVR). Mispricing retention offers loses either margin (too generous) or customers (too mean), and lapsing onto SVR can be a poor customer outcome.

**Business value**
- Higher **retention / product-transfer rate** at maturity → protects the back-book and Net Interest Margin (NIM).
- Efficient use of pricing "budget": discounts concentrated where they change behaviour (uplift), not given to those who would stay anyway.

**Consumer value**
- The **right product at the right time**, reducing the risk of silently overpaying on SVR.
- Personalised, fair offers aligned to Consumer Duty fair-value rules.

**Data & modelling approach**
- **Targets:** churn/switch-away propensity; response/uplift to a given retention offer.
- **Features:** deal end date, current vs. market rate gap, LTV, equity, life-stage signals, engagement, prior switching behaviour.
- **Models:** propensity + **uplift/causal** models (two-model or meta-learners) to rank customers by *incremental* retention per pound of discount; optimisation layer for offer selection under a margin constraint.

**KPIs / success metrics**
Retention rate at maturity, NIM retained, incremental retention per £ of discount, SVR-lapse rate reduction, fair-value assessment pass.

**Risks & controls**
Fair-value & price-discrimination governance, avoid detriment to disengaged/vulnerable customers (they should get *good* offers too, not worse ones).

---

## ML-4: Early financial-difficulty (pre-arrears) risk model

**Business problem**
By the time a customer misses a mortgage payment, options are limited and stress is high. The bank needs to identify customers drifting toward difficulty **weeks or months before** the first missed payment, to offer support early.

**Business value**
- Lower **arrears roll-rates and impairment**; cheaper, earlier interventions.
- Strong **Consumer Duty** evidence of proactive support and avoiding foreseeable harm.

**Consumer value**
- Early, empathetic, non-judgemental outreach and options **before** a credit-file-damaging missed payment.

**Data & modelling approach**
- **Target:** transition into arrears (e.g. 30+ DPD) within a forward window.
- **Features:** current-account inflow/outflow trends, minimum-payment behaviour on other products, transaction-level stress signals (with consent), payment-holiday history, macro overlays (rates, energy prices), demographic/vulnerability flags handled with care.
- **Models:** discrete-time hazard / gradient boosting with a **cost-sensitive** objective; strict monitoring for fairness so support is offered equitably.

**KPIs / success metrics**
Lead-time before first missed payment, % of at-risk customers contacted, roll-rate reduction, cure rate after intervention, customer-reported helpfulness.

**Risks & controls**
Vulnerability handling, data-minimisation and consent for transaction data, careful tone (support not debt-chasing), FCA forbearance rules.

---

# Layer 2 — Market Mix Modelling Use Cases

MMM quantifies how **marketing and commercial levers** (media spend, promotions, pricing, channel investment) drive business outcomes, controlling for external factors (interest rates, housing market, seasonality). Unlike last-click digital attribution, MMM is privacy-friendly (aggregate data) and captures long-run and offline effects — well suited to a considered, high-value purchase like a mortgage.

---

## MMM-1: Mortgage acquisition marketing effectiveness & budget optimisation

**Business problem**
Marketing spends across TV, digital, search, social, sponsorships, price-comparison sites and brand — but cannot confidently answer: *which channels actually drive mortgage applications and completions, at what saturation, and how should next quarter's budget be split?* Digital attribution over-credits last-click and ignores the long consideration window and rate/housing-market effects.

**Business value**
- **Reallocate the acquisition budget** to higher-ROI channels — typically 10–20%+ efficiency gains without extra spend.
- Quantify **diminishing returns / saturation** per channel to avoid over-investing.
- Separate **base demand** (driven by rates, housing market, brand) from **incremental** marketing-driven demand, so credit isn't wrongly assigned.

**Consumer value**
- More **relevant, less wasteful** marketing; investment shifts toward genuinely useful information and channels customers value.

**Data & modelling approach**
- **Target:** weekly mortgage applications / completions (and by segment/region).
- **Drivers:** media spend by channel with **adstock/carryover** and **saturation (Hill/log)** transformations, price/rate competitiveness, promotions, plus **control variables**: Bank of England base rate, house-price index, seasonality, macro sentiment.
- **Models:** Bayesian MMM (e.g. PyMC-Marketing / Robyn-style) for uncertainty-aware ROI and response curves; **budget-optimisation** step to maximise expected applications under a spend constraint.
- **Calibration:** where possible, calibrate with geo-experiments / lift tests to strengthen causal claims.

**KPIs / success metrics**
Marketing ROI (cost per incremental completion), forecast accuracy of applications, decomposition stability, budget-shift lift validated by holdout experiments.

**Risks & controls**
MMM shows correlation-plus-priors, not perfect causality — triangulate with experiments; refresh regularly as rate environment changes the base.

---

## MMM-2: Broker vs. direct channel mix & intermediary incentive optimisation

**Business problem**
A large share of mortgages is originated via **intermediaries/brokers**, the rest direct (branch, phone, digital). Leadership needs to understand the true marginal economics of each channel — acquisition cost, conversion, quality/retention, and the effect of proc-fees/incentives — to decide where to invest capacity and how to structure intermediary support.

**Business value**
- Optimise the **channel mix and intermediary investment** for profitable, quality volume (not just volume).
- Understand how broker incentives, service levels (time-to-offer) and product availability shift intermediary flow.

**Consumer value**
- Customers are supported through their **preferred channel** with adequate capacity, improving service and reducing waits.

**Data & modelling approach**
- MMM/econometric model with **channel investment, service metrics, proc-fee/incentive levels, product competitiveness** as drivers of channel-level volume and quality; control for rates and market conditions.
- Combine with ML-1/ML-2/ML-3 outputs to weight for **loan quality and retention**, not just origination.

**KPIs / success metrics**
Profit-per-channel, quality-adjusted acquisition cost, broker satisfaction/service SLAs, retention by channel.

**Risks & controls**
Intermediary conduct & fair-value rules; ensure incentive structures don't create customer-outcome conflicts.

---

# Layer 3 — Time-Series Forecasting Use Cases

Time-series models forecast **how key quantities evolve over time**, enabling proactive planning for capacity, funding, provisioning and retention.

---

## TS-1: Mortgage demand & application-volume forecasting

**Business problem**
Application volumes swing sharply with interest-rate moves, seasonality (spring/summer moving season), stamp-duty changes and rate-lock deadlines. Under-forecasting causes backlogs, long waits and poor service; over-forecasting wastes underwriting capacity. Funding and pricing also depend on expected volumes.

**Business value**
- Better **operational capacity planning** (underwriting, valuations, contact centre) → shorter cycle times.
- Improved **funding and pricing** decisions aligned to expected pipeline.

**Consumer value**
- **Shorter waits and consistent service** even during rate-driven surges.

**Data & modelling approach**
- **Series:** weekly/daily applications, DIPs, completions, by product/region/channel.
- **Drivers/exogenous:** base rate & swap curves, house-price indices, seasonality, competitor rate moves, marketing calendar (link to MMM).
- **Models:** classical (SARIMAX/ETS) baselines; ML/global models (gradient boosting on lag features, Prophet, temporal deep-learning) with exogenous regressors; probabilistic forecasts (prediction intervals) for capacity risk planning; scenario forecasts for "+100bps rate" shocks.

**KPIs / success metrics**
Forecast accuracy (MAPE/WAPE, pinball loss for intervals), capacity utilisation, SLA adherence during peaks, planning cycle improvement.

**Risks & controls**
Regime changes (rate shocks) break naïve models — maintain scenario overlays and human review.

---

## TS-2: Arrears & impairment (ECL) early-warning forecasting

**Business problem**
Macro shocks (rising rates, energy prices, unemployment) feed through to mortgage arrears with a lag. The bank must forecast **portfolio-level arrears roll-rates and expected credit losses** to provision adequately (IFRS 9), plan support capacity, and satisfy stress-testing — while ML-4 handles the individual-customer view.

**Business value**
- More accurate, **forward-looking IFRS 9 ECL** and capital planning; fewer provisioning surprises.
- Ability to **pre-position support capacity** ahead of a forecast deterioration.

**Consumer value**
- Support teams and forbearance options are **resourced ahead of shocks**, so help is available when a wave of customers needs it.

**Data & modelling approach**
- **Series:** stage-transition / roll-rate matrices, arrears buckets, cure rates by segment/vintage.
- **Drivers:** macroeconomic scenarios (base/upside/downside), rate paths, unemployment, HPI, cost-of-living indices.
- **Models:** transition-matrix / vintage models, macro-linked regression, and probabilistic time-series; scenario-conditioned forecasting feeding ECL and stress tests.

**KPIs / success metrics**
ECL forecast error vs. actuals, stage-migration prediction accuracy, provisioning stability, stress-test alignment.

**Risks & controls**
Model risk governance, scenario governance, avoid pro-cyclicality; align with finance & risk sign-off.

---

## TS-3: Prepayment / redemption & balance run-off forecasting

**Business problem**
Mortgages redeem early (moving, remortgaging away, overpayments) and this **prepayment behaviour** drives balance run-off, interest-rate risk in the banking book (IRRBB), retention planning and hedging. Poor prepayment forecasts lead to mis-hedged books and missed retention opportunities.

**Business value**
- Better **balance-sheet, hedging (IRRBB) and liquidity** management.
- Sharper **retention planning** — anticipate redemption waves (e.g. maturity cohorts) and act early (links to ML-3 and AG-3).

**Consumer value**
- Timely, **proactive retention offers** so customers aren't left to drift onto SVR or churn unnecessarily.

**Data & modelling approach**
- **Series:** redemption/prepayment rates (CPR/SMM) by vintage, product, rate incentive (gap between customer rate and market), seasonality.
- **Models:** prepayment hazard/survival models, econometric models linking prepayment to the rate incentive and housing activity, probabilistic time-series for cohort-level run-off.

**KPIs / success metrics**
Prepayment forecast accuracy, hedge effectiveness, retention capture rate on forecast redemption cohorts.

**Risks & controls**
Behavioural regime shifts with rate cycles; combine with ML-3 uplift to avoid over/under-offering.

---

# Layer 4 — Agentic AI Use Cases (Consumer-facing)

The agentic layer sits **on top of** the models above. Agents use LLM reasoning to understand a customer's goal, then **plan and orchestrate** calls to the governed ML/MMM/TS models, product and policy knowledge bases, and transactional tools (via APIs/MCP), keeping a human in the loop for decisions. The aim is to turn a fragmented, stressful, jargon-heavy journey into a guided, supportive one — solving **real consumer problems**.

> **Guardrails common to all agents:** consented data only; no autonomous credit/pricing/forbearance decisions (agents recommend and prepare, humans/decision-engines decide); full audit trail; retrieval grounded in approved sources (no hallucinated policy/rates); vulnerability detection routes to human support; Consumer Duty and financial-promotion compliance checks on any generated content.

---

## AG-1: "Home Buying Copilot" — end-to-end mortgage journey assistant

**Real consumer problem**
Buying a home is one of life's most stressful, confusing and infrequent financial events. Customers don't know how much they can borrow, what documents are needed, what the jargon means, where they are in the process, or what to do next. They juggle estate agents, solicitors, surveyors and the lender with no single guide.

**What the agent does**
- Understands the customer's situation and goal (e.g. "first-time buyer, £40k saved, looking around £280k").
- Calls **ML-1 (affordability/PD)** to give an *indicative, explainable* borrowing range and the key factors, and **ML-2 (conversion)** to detect where they might get stuck.
- Explains next steps, jargon and timelines in plain English; tracks live application stage (via platform APIs) and proactively flags what's needed next.
- Coordinates the moving parts (checklist for solicitor/survey), sends reminders, and answers questions 24/7, escalating to a human adviser for advice/decisions.

**Business value**
Higher conversion (fewer drop-outs), lower cost-to-serve via deflected simple queries, and richer engagement data.

**Consumer value**
Clarity, confidence and reduced anxiety; a single always-on guide through a daunting process.

**KPIs**
Conversion uplift, journey completion time, self-serve resolution rate, CSAT/NPS, escalation quality.

---

## AG-2: "Affordability & Deposit Coach" — get mortgage-ready agent

**Real consumer problem**
Many aspiring buyers (especially first-time buyers) are declined or don't apply because they don't know **what's holding them back** or **what to change**. Generic "improve your credit score" advice isn't actionable for *their* situation.

**What the agent does**
- With consent, reviews the customer's affordability picture and runs **ML-1** in "what-if" mode to identify the **specific, personalised levers** (e.g. "clearing this £3k loan and adding £2k deposit would move you from decline to an indicative offer around £X").
- Builds a realistic **savings/deposit plan** with milestones and government schemes; simulates timelines ("on this plan you could be ready in ~11 months").
- Nurtures the relationship over time and re-checks readiness — building a **future pipeline of qualified applicants**.

**Business value**
Creates tomorrow's qualified applicants, deepens relationships, and turns declines into future approvals — a growth *and* good-outcome play.

**Consumer value**
A concrete, personalised path to home ownership instead of an unexplained "no".

**KPIs**
Decline-to-approval conversion over time, plan adherence, future application rate, financial-wellbeing indicators.

---

## AG-3: "Rate & Remortgage Guardian" — proactive switching agent

**Real consumer problem**
Customers silently roll onto expensive SVR or overpay because they forget their deal is ending, find switching confusing, or don't realise a better option exists. This is a classic Consumer Duty "sludge"/inertia harm.

**What the agent does**
- Monitors deal-end dates and, using **ML-3 (retention/uplift)** and **TS-3 (prepayment)**, reaches out **ahead of maturity** with a personalised, plain-English comparison of options (product transfer vs. remortgage), including total cost.
- Explains trade-offs, answers questions, pre-fills the switch, and books a human adviser where advice is required.
- Flags when *staying put* or an external option is genuinely better for the customer (fair-value integrity builds trust).

**Business value**
Higher retention/product-transfer rates, protected NIM, reduced attrition — targeted with uplift so discounts go where they matter.

**Consumer value**
No accidental SVR overpayment; the right switch at the right time with minimal effort.

**KPIs**
Retention at maturity, SVR-lapse reduction, switch completion rate, customer trust/CSAT.

---

## AG-4: "Financial Resilience Companion" — arrears prevention & support agent

**Real consumer problem**
Customers heading into financial difficulty often avoid contact out of fear or shame until it's too late, damaging their credit file and wellbeing. Support can feel intimidating and hard to access.

**What the agent does**
- Uses **ML-4 (pre-arrears)** and **TS-2 (portfolio early-warning)** to identify customers who may be struggling and initiates **early, empathetic, non-judgemental** outreach (opt-in, carefully toned).
- Offers a private, 24/7 space to explore options (budget review, payment plans, forbearance types, external debt-advice signposting) and explains impacts clearly.
- **Prepares** tailored support options and warm-hands-off to specialist human teams for any decision; detects vulnerability signals and routes appropriately.

**Business value**
Lower arrears roll-rates and impairment; strong, auditable Consumer Duty evidence of proactive, fair support.

**Consumer value**
Early, dignified help **before** a missed payment; reduced stress and protected credit standing.

**KPIs**
Early-engagement rate, arrears prevented/cured, customer-reported helpfulness, safe escalation of vulnerable cases.

**Special controls**
This agent must be exceptionally carefully governed: opt-in, sensitive tone, no automated adverse actions, mandatory human involvement in forbearance, and rigorous vulnerability handling.

---

## AG-5: "Document & Application Concierge" — friction-removal agent

**Real consumer problem**
The single biggest source of mortgage-journey frustration is **document collection**: unclear requirements, wrong uploads, repeated re-requests, and delays waiting on payslips, bank statements and ID. Customers feel they're in a black box.

**What the agent does**
- Generates a **personalised, dynamic document checklist** based on the applicant's profile and product (via ML-2's stall-stage insight).
- Guides uploads, uses document AI to **pre-validate** completeness/quality *before* submission (right document, in date, legible), reducing re-requests.
- Gives real-time status ("valuation booked, one payslip outstanding") and chases only what's genuinely missing.

**Business value**
Lower processing cost, faster time-to-offer, reduced pipeline stalls (complements ML-2).

**Consumer value**
Far less paperwork stress, fewer frustrating re-requests, and transparency on exactly what's needed and why.

**KPIs**
First-time-right document rate, re-request reduction, time-to-offer, drop-out at document stage, CSAT.

---

## How the layers connect

```
                 ┌─────────────────────────────────────────────────────────┐
                 │                AGENTIC AI LAYER (Layer 4)                 │
                 │  AG-1 Copilot · AG-2 Coach · AG-3 Guardian ·             │
                 │  AG-4 Resilience Companion · AG-5 Concierge              │
                 │  (LLM reasoning + planning + tool/API/MCP orchestration) │
                 └───────────────▲───────────────▲───────────────▲─────────┘
                                 │ calls as tools │               │
        ┌────────────────────────┴───┐   ┌────────┴─────────┐   ┌─┴───────────────────┐
        │  CORE ML (Layer 1)         │   │  MMM (Layer 2)   │   │  TIME SERIES (L3)   │
        │  ML-1 PD/affordability     │   │  MMM-1 acq. mix  │   │  TS-1 demand        │
        │  ML-2 conversion           │   │  MMM-2 channel   │   │  TS-2 arrears/ECL   │
        │  ML-3 retention/uplift     │   │                  │   │  TS-3 prepayment    │
        │  ML-4 pre-arrears          │   │                  │   │                     │
        └────────────────────────────┘   └──────────────────┘   └─────────────────────┘
                                 ▲                                        ▲
                                 └──────────── DATA FOUNDATION ──────────┘
             (application, bureau, transaction/open-banking [consented], servicing,
              marketing spend, macroeconomic & housing-market data, journey telemetry)
```

- **MMM (Layer 2)** and **TS-1 (demand)** share drivers (marketing calendar, rates) and inform each other: MMM explains *why* demand moves; TS-1 forecasts *how much*.
- **ML-3 (retention)** and **TS-3 (prepayment)** together drive **AG-3 (Guardian)**.
- **ML-4 (pre-arrears)** and **TS-2 (portfolio arrears)** together drive **AG-4 (Resilience Companion)** — individual + portfolio views.
- **ML-1 / ML-2** power **AG-1, AG-2, AG-5** across the acquisition journey.

---

## Data foundations

| Data domain | Examples | Used by |
|-------------|----------|---------|
| Application & origination | DIP/full application, product, LTV/LTI, valuation, stage events | ML-1, ML-2, TS-1, AG-1/5 |
| Credit bureau | Scores, commitments, defaults, searches | ML-1, ML-4 |
| Transaction / open banking (consented) | Income, spend, cash-flow stress signals | ML-1, ML-4, AG-2/4 |
| Servicing & arrears | Payment history, roll-rates, forbearance, redemptions | ML-3, ML-4, TS-2, TS-3, AG-3/4 |
| Marketing | Spend by channel, campaigns, promotions | MMM-1, MMM-2, TS-1 |
| Macro & housing market | Base rate, swap curves, HPI, unemployment, energy/CoL | MMM, TS-1/2/3, ML overlays |
| Journey telemetry | Digital events, idle time, doc re-requests | ML-2, AG-1/5 |

**Cross-cutting needs:** consent & data-minimisation, feature store for reuse across models, a governed model registry, and an agent tool/API (or MCP) layer exposing models and knowledge to the agentic layer safely.

---

## Responsible AI, regulation & governance

- **FCA Consumer Duty** — every use case maps to a good-outcome (avoiding foreseeable harm, fair value, consumer support, consumer understanding). Retention and arrears use cases especially must evidence fairness.
- **Fair lending / equality** — monitor models for disparate impact; retain explainable challengers; document adverse-action reasons.
- **Model risk management** — PRA **SS1/23** model risk principles; independent validation, ongoing monitoring (PSI/drift), and clear model owners.
- **IFRS 9 / stress testing** — TS-2 outputs must align with finance & risk governance.
- **Data protection** — UK GDPR lawful basis, consent for transaction/open-banking data, purpose limitation, data-minimisation.
- **Agentic-specific controls** — no autonomous financial decisions; grounded retrieval only; audit trails; human-in-the-loop for advice/decisions; vulnerability routing; financial-promotion compliance on generated content.

---

## Suggested delivery roadmap

Sequenced by dependency and value, **not** by calendar time:

1. **Foundations** — data/feature store, model registry, governance guardrails.
2. **Core ML that de-risks and grows** — ML-1 (affordability/PD) and ML-2 (conversion) first; they underpin the agents and deliver standalone value.
3. **Time series for planning** — TS-1 (demand) for operations; TS-2/TS-3 for risk & balance-sheet.
4. **MMM** — MMM-1 (acquisition), then MMM-2 (channel), to optimise commercial spend and feed demand forecasts.
5. **Retention & support ML** — ML-3 and ML-4, unlocking the highest-value consumer agents.
6. **Agentic layer, staged** — start with lower-risk, high-clarity agents (AG-5 Concierge, AG-1 Copilot), then AG-2/AG-3, and finally AG-4 (Resilience Companion) with the strongest governance.

Each step should ship with success metrics, a control/fairness assessment, and a human-in-the-loop design before scaling.

---

*This document is a business-use-case catalogue for discussion and prioritisation. It intentionally contains no production code; all modelling approaches are indicative and must go through the group's model-risk, data-protection and Consumer Duty governance before build.*
