# Home Loan MCP Server on GCP — Delivery Plan (2 Sprints)

> **Project:** Bank Home Loan Knowledge Assistant via Model Context Protocol (MCP)
> **Sponsor:** Home Loan Department, Retail Banking
> **Goal:** Build a secure, GCP-deployed **MCP server** that exposes the bank's home-loan knowledge and data so that AI clients (**Gemini, ChatGPT, and Claude**) can answer customer/agent questions accurately and traceably — and do it as a **reusable framework** that other departments (Personal Loans, Cards, Deposits, etc.) can adopt with minimal effort.
> **Cloud:** Google Cloud Platform (GCP) only.
> **Cadence:** 2 sprints, each **14 working days**.

---

## 1. Executive Summary

The Home Loan department needs an AI assistant that answers questions strictly from **the bank's own approved content and data** — product sheets, eligibility rules, interest-rate cards, EMI logic, document checklists, FAQs, and policy documents — rather than from an LLM's general knowledge.

The clean way to do this in 2026 is the **Model Context Protocol (MCP)**. We build **one MCP server** that exposes:

- **Tools** (callable functions) such as `calculate_emi`, `check_eligibility`, `get_interest_rates`, `list_required_documents`.
- **Resources / Retrieval** (a RAG search tool, e.g. `search_homeloan_kb`) backed by the bank's documents.

Any MCP-compatible client can then call this server:

- **Claude** — native MCP support (Claude Desktop and API connectors).
- **ChatGPT / OpenAI** — via the Agents SDK / Responses API MCP connector and custom GPT actions.
- **Gemini** — via the Gemini API / Vertex AI function-calling bridge and the Gen AI SDK's MCP support.

The whole thing is deployed on GCP (Cloud Run + Vertex AI + supporting services) and packaged as a **"MCP-in-a-box" framework** so other departments reuse the same skeleton.

> **Note on the VS Code blocker:** You mentioned MCP servers cannot be enabled in VS Code for testing. This plan **does not depend on VS Code**. We test the server with the **MCP Inspector**, automated `pytest` clients, a hosted **MCP Inspector on Cloud Run**, and direct connections from Claude Desktop / OpenAI / Gemini. VS Code is optional, not on the critical path. See [Section 8 — Testing Strategy](#8-testing-strategy-no-vs-code-required).

---

## 2. Objectives & Success Criteria

| # | Objective | Measurable Success Criteria |
|---|-----------|------------------------------|
| O1 | Answer home-loan questions from bank data only | ≥ 95% of test questions answered with a citation to a source document; "I don't know / please contact branch" returned when out of scope |
| O2 | Connect to all three LLM clients | Working demo from Claude, ChatGPT, and Gemini against the same MCP server |
| O3 | Deploy on GCP | Server reachable on a private/authenticated Cloud Run URL; passes health checks; auto-scales |
| O4 | Reusable for other departments | A new department onboarded from the template in < 1 day using config + content only (no core code changes) |
| O5 | Security & compliance | No PII leakage; audit log of every tool call; auth enforced; passes security review |
| O6 | Accuracy of calculators | EMI / eligibility results match the bank's reference spreadsheet to the paisa for 100% of test cases |

**Definition of Done (project-level):** All Sprint 1 + Sprint 2 stories `Done`, security review passed, runbook + onboarding guide published, demo recorded for all three LLM clients, framework template repo tagged `v1.0`.

---

## 3. Scope

### In Scope
- One production MCP server for **Home Loans**.
- Tools: EMI calculator, eligibility check, interest-rate lookup, document checklist, fees/charges lookup, RAG knowledge search.
- RAG pipeline over approved home-loan documents.
- GCP deployment (Cloud Run, Vertex AI, Cloud Storage, etc.).
- Reusable framework/template + onboarding guide.
- Connectors/configs for Claude, ChatGPT, Gemini.
- Testing harness independent of VS Code.
- Observability, audit logging, security baseline.

### Out of Scope (this release)
- Performing actual loan transactions / write-backs to core banking.
- Customer-authentication / account-specific data (read of personal account balances) — *flagged as Phase 2*.
- Fine-tuning a custom LLM (we use RAG + tools, not fine-tuning).
- Mobile/web chat front-end (clients are the LLM apps themselves).
- Migrating other departments' content (we deliver the template; departments load their own data).

---

## 4. Solution Architecture (GCP)

### 4.1 High-Level Flow

```
  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │   Claude    │     │  ChatGPT/   │     │   Gemini /  │
  │  (Desktop / │     │  OpenAI     │     │  Vertex AI  │
  │   API)      │     │  Agents SDK │     │  Gen AI SDK │
  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
         │  MCP (HTTP/SSE + OAuth/API key)       │
         └───────────────┬──────────────────────┘
                         ▼
              ┌──────────────────────┐
              │  Cloud Load Balancer │  (TLS, WAF/Cloud Armor)
              └──────────┬───────────┘
                         ▼
              ┌──────────────────────────────┐
              │   Cloud Run: MCP Server      │
              │   (FastMCP / streamable-HTTP)│
              │  ┌────────────────────────┐  │
              │  │ Tools:                 │  │
              │  │  calculate_emi         │  │
              │  │  check_eligibility     │  │
              │  │  get_interest_rates    │  │
              │  │  list_required_docs    │  │
              │  │  get_fees_charges      │  │
              │  │  search_homeloan_kb ───┼──┼──► RAG
              │  └────────────────────────┘  │
              └──────┬──────────────┬────────┘
                     ▼              ▼
        ┌─────────────────┐  ┌───────────────────────────┐
        │ Cloud SQL /     │  │ Vertex AI:                │
        │ Firestore       │  │  - Embeddings (text-emb)  │
        │ (rates, rules,  │  │  - Vector Search index    │
        │  product config)│  │  - (opt) Gemini for RAG   │
        └─────────────────┘  └───────────┬───────────────┘
                                         ▼
                              ┌───────────────────────┐
                              │ Cloud Storage (GCS):  │
                              │  approved KB documents│
                              └───────────────────────┘

  Cross-cutting: Secret Manager · IAM · Cloud Logging/Monitoring ·
                 Audit Logs · Cloud Armor · Artifact Registry · Cloud Build
```

### 4.2 GCP Service Mapping

| Concern | GCP Service | Why |
|---------|-------------|-----|
| Run the MCP server | **Cloud Run** | Serverless, scales to zero, easy HTTPS, container-based, supports streamable-HTTP/SSE |
| Container builds & CI | **Cloud Build + Artifact Registry** | Native build/deploy pipeline |
| LLM + embeddings (for RAG) | **Vertex AI** (Gemini, `text-embedding`) | Managed, in-region, IAM-controlled |
| Vector store | **Vertex AI Vector Search** (or `pgvector` on Cloud SQL for smaller volumes) | Semantic retrieval of KB chunks |
| Structured data (rates, rules) | **Cloud SQL (Postgres)** or **Firestore** | Interest rates, eligibility matrices, fees |
| Document store | **Cloud Storage (GCS)** | Source-of-truth approved PDFs/docs |
| Secrets | **Secret Manager** | API keys, DB creds, client tokens |
| Identity & access | **IAM + Workload Identity** | Least-privilege, no static keys in code |
| Edge security | **Cloud Armor + HTTPS LB** | WAF, rate limiting, IP allow-listing |
| Observability | **Cloud Logging, Monitoring, Trace** | Metrics, dashboards, alerts |
| Audit | **Cloud Audit Logs + app-level audit table** | Every tool call recorded |
| Pipeline orchestration (ingest) | **Cloud Run Jobs / Workflows / Pub/Sub** | Document ingestion & re-index |

### 4.3 Recommended Tech Stack
- **Language:** Python 3.12.
- **MCP framework:** official **MCP Python SDK** with **FastMCP**, served over **streamable HTTP** (remote transport) so cloud clients can connect.
- **RAG:** Vertex AI embeddings + Vertex AI Vector Search; LangChain/LlamaIndex optional as glue.
- **API/runtime:** Uvicorn/Starlette (FastMCP handles this), containerized with Docker.
- **IaC:** Terraform (so the whole stack is reproducible per department).
- **CI/CD:** Cloud Build triggers from Git; deploy to Cloud Run.

---

## 5. How the MCP Server Connects to Each LLM

| Client | Connection mechanism | Notes |
|--------|----------------------|-------|
| **Claude** | Native MCP. Claude Desktop "Custom Connector" / `claude_desktop_config.json` for local; **remote MCP connector (URL + OAuth)** for the Cloud Run endpoint. | Easiest path; reference client. |
| **ChatGPT / OpenAI** | **OpenAI Agents SDK** `MCPServerStreamableHttp`, or **Responses API** MCP tool, or a **Custom GPT Action** pointing to the server. | Use OAuth/API-key auth on the server. |
| **Gemini** | **Google Gen AI SDK** supports passing an MCP session to Gemini function-calling; or bridge MCP tools to **Vertex AI function declarations**. | Keeps everything inside GCP. |

> **Key design rule:** the server is **client-agnostic**. We expose standard MCP tools over streamable HTTP with auth; each LLM just needs a small connector config. We deliver all three configs as part of the framework.

---

## 6. Reusable Multi-Department Framework

The whole point is "build once, reuse everywhere." We achieve this by separating **core** (shared code) from **department config + content**.

```
mcp-bank-framework/                ← shared template repo (tag v1.0)
├── core/                          ← DO NOT edit per department
│   ├── server.py                  ← FastMCP app, auth, logging, health
│   ├── rag/                       ← embedding + vector search client
│   ├── tools/                     ← generic tool loader / registry
│   ├── auth/                      ← OAuth / API-key middleware
│   └── observability/             ← logging, tracing, audit
├── departments/
│   └── home_loan/                 ← EXAMPLE department (our deliverable)
│       ├── config.yaml            ← name, models, index id, auth scope
│       ├── tools.py               ← EMI, eligibility, rate lookup, etc.
│       ├── prompts/               ← guardrail/system instructions
│       └── data_manifest.yaml     ← which GCS docs to ingest
├── infra/                         ← Terraform modules (1 per dept = 1 var file)
│   ├── modules/                   ← reusable Cloud Run/VertexAI/SQL modules
│   └── envs/home_loan.tfvars
├── pipelines/                     ← ingestion + reindex jobs
├── tests/                         ← shared test harness + dept test packs
└── docs/                          ← onboarding guide, runbook
```

**Onboarding a new department** = (1) copy `departments/home_loan` → `departments/<new>`, (2) edit `config.yaml`, (3) drop their docs in GCS + update `data_manifest.yaml`, (4) add a `*.tfvars`, (5) run the pipeline. No core code changes. This is **Epic E7** and is what makes O4 measurable.

---

## 7. Security, Compliance & Guardrails (Banking)

- **No hallucinated finance advice:** system prompt + tool design force "answer only from retrieved/approved content; otherwise defer to branch/RM."
- **PII:** Phase-1 scope handles **product/policy** info, not customer account data. PII redaction filter on inputs/outputs as defense in depth.
- **AuthN/AuthZ:** OAuth 2.1 / signed API keys on the MCP endpoint; IAM + Workload Identity inside GCP; per-department auth scopes.
- **Audit:** every tool call logged (who, when, tool, args hash, source docs returned) to an immutable audit sink.
- **Data residency:** all Vertex AI / storage in the bank's approved GCP region.
- **Network:** Cloud Armor (WAF, rate-limit), private ingress where possible, TLS everywhere.
- **Content governance:** only **approved** docs are ingested; ingestion requires a sign-off flag in `data_manifest.yaml`.
- **Model governance:** record which model/version answered; allow per-department model pinning.
- **Reviews:** mandatory InfoSec + Compliance sign-off gate before production (Sprint 2).

---

## 8. Testing Strategy (No VS Code Required)

Because MCP cannot be enabled in your VS Code, we **avoid VS Code entirely** for testing and use these layers:

1. **MCP Inspector (local & hosted):** `npx @modelcontextprotocol/inspector` to interactively call every tool, inspect schemas, and verify responses. Optionally host the Inspector on Cloud Run for the QA team.
2. **Automated unit tests:** `pytest` for each tool's business logic (EMI math, eligibility matrix) against the bank's reference spreadsheet — 100% must match.
3. **Automated MCP client tests:** a Python MCP client in CI that lists tools, calls each one, and asserts on structured output and citations.
4. **RAG quality eval:** a labelled question→expected-source test set; measure retrieval hit-rate and answer groundedness (target ≥ 95%).
5. **Real client smoke tests:** connect the deployed server to **Claude Desktop**, **OpenAI Agents SDK**, and **Gemini Gen AI SDK** and run a scripted Q&A set.
6. **Load & security tests:** k6/Locust load test on Cloud Run; auth bypass + prompt-injection test cases.

CI runs layers 1–4 on every PR via Cloud Build; layers 5–6 are gated pre-production.

---

## 9. Sprint Plan Overview

- **2 sprints × 14 working days each.**
- Team assumption (adjust to your actual roster): 1 Tech Lead, 2 Backend/MCP Engineers, 1 ML/RAG Engineer, 1 DevOps/Cloud Engineer, 0.5 QA, 0.5 Product/BA, plus part-time InfoSec & Compliance reviewers.
- Estimation in **story points (SP)**; ~1 SP ≈ half a day of one engineer. Target velocity ≈ **45–55 SP/sprint** for a team of this size.

| Sprint | Theme | Outcome |
|--------|-------|---------|
| **Sprint 1** | Foundation, MCP server, tools, RAG, GCP base | A working Home-Loan MCP server deployed to a **dev** Cloud Run, callable via MCP Inspector + at least one LLM client |
| **Sprint 2** | Hardening, all 3 LLM clients, security, reusable framework, production deploy | **Production** deployment, all 3 clients demoed, framework template + onboarding guide, security sign-off |

> The remainder (Sections 10–11) is the **Jira backlog**: Epics → Stories with story points, acceptance criteria, and dependencies, organized into the two sprints.

---

## 10. Jira Backlog — Epics

| Epic | Key | Title | Sprint(s) |
|------|-----|-------|-----------|
| E1 | HLMCP-E1 | GCP Foundation & DevOps | S1 |
| E2 | HLMCP-E2 | Core MCP Server | S1 |
| E3 | HLMCP-E3 | Home-Loan Tools (calculators & lookups) | S1 |
| E4 | HLMCP-E4 | Knowledge Base & RAG Pipeline | S1 (→S2) |
| E5 | HLMCP-E5 | LLM Client Integrations (Claude/ChatGPT/Gemini) | S1 (Claude) → S2 (all) |
| E6 | HLMCP-E6 | Security, Compliance & Observability | S1 (base) → S2 (full) |
| E7 | HLMCP-E7 | Reusable Multi-Department Framework | S2 |
| E8 | HLMCP-E8 | Testing, QA & Acceptance | S1 → S2 |
| E9 | HLMCP-E9 | Production Deployment & Handover | S2 |

---

## 11. Jira Tickets by Sprint

> Legend — **Type:** Story/Task/Spike. **SP:** story points. **AC:** acceptance criteria. **Dep:** dependency.

### 🟦 SPRINT 1 — Foundation, MCP Server, Tools, RAG (Dev)

**Sprint 1 Goal:** *A working Home-Loan MCP server deployed to dev Cloud Run, exposing EMI/eligibility/rate/document tools + a basic RAG search, verified via MCP Inspector and reachable from Claude.*

#### Epic E1 — GCP Foundation & DevOps

| Key | Type | Title | SP | Acceptance Criteria | Dep |
|-----|------|-------|----|---------------------|-----|
| HLMCP-101 | Task | Create GCP project, billing, regions, IAM baseline | 3 | Project + dev env created; least-privilege IAM groups; budget alert set | — |
| HLMCP-102 | Task | Enable APIs (Cloud Run, Vertex AI, Vector Search, Cloud SQL, Secret Manager, Artifact Registry, Cloud Build) | 2 | All required APIs enabled via Terraform; documented | 101 |
| HLMCP-103 | Story | Terraform skeleton (modules: Cloud Run, network, IAM, storage) | 5 | `terraform apply` provisions empty stack reproducibly; state in GCS backend | 101,102 |
| HLMCP-104 | Story | CI/CD: Cloud Build pipeline (build → test → deploy to dev Cloud Run) | 5 | Push to main builds container, runs tests, deploys to dev automatically | 103 |
| HLMCP-105 | Task | Artifact Registry + Secret Manager setup | 2 | Container registry live; secrets stored, no creds in code | 102 |

**E1 subtotal: 17 SP**

#### Epic E2 — Core MCP Server

| Key | Type | Title | SP | Acceptance Criteria | Dep |
|-----|------|-------|----|---------------------|-----|
| HLMCP-110 | Spike | Confirm MCP transport (streamable-HTTP) + SDK choice (FastMCP) | 2 | Decision recorded; minimal "hello tool" runs locally | — |
| HLMCP-111 | Story | Scaffold FastMCP server with health check + tool registry | 5 | Server boots, `/health` returns 200, MCP Inspector lists tools | 110 |
| HLMCP-112 | Story | Containerize + deploy MCP server to dev Cloud Run (HTTPS) | 5 | Public/auth'd Cloud Run URL responds to MCP handshake | 111,104 |
| HLMCP-113 | Story | Config-driven tool loading (per-department config.yaml) | 3 | Tools load from config; adding a tool needs no core edits | 111 |
| HLMCP-114 | Task | Structured logging + request/trace IDs | 2 | Each request logged with correlation id to Cloud Logging | 111 |

**E2 subtotal: 17 SP**

#### Epic E3 — Home-Loan Tools

| Key | Type | Title | SP | Acceptance Criteria | Dep |
|-----|------|-------|----|---------------------|-----|
| HLMCP-120 | Story | `calculate_emi` tool | 3 | EMI matches reference sheet to the paisa for all test cases; input validation; typed schema | 113 |
| HLMCP-121 | Story | `check_eligibility` tool (income, age, FOIR, LTV, CIBIL band) | 5 | Returns eligible amount + reasons; matches policy matrix; handles edge cases | 113 |
| HLMCP-122 | Story | `get_interest_rates` tool (by product/tenure/profile) | 3 | Reads current rate card from Cloud SQL/Firestore; returns effective rate + date | 124 |
| HLMCP-123 | Story | `list_required_documents` + `get_fees_charges` tools | 3 | Returns checklist/fees per product/applicant type from config/DB | 124 |
| HLMCP-124 | Task | Seed structured data store (rates, eligibility matrix, fees) | 3 | Cloud SQL/Firestore seeded; admin-updatable; versioned | 103 |

**E3 subtotal: 17 SP**

#### Epic E4 — Knowledge Base & RAG (started in S1)

| Key | Type | Title | SP | Acceptance Criteria | Dep |
|-----|------|-------|----|---------------------|-----|
| HLMCP-130 | Story | GCS bucket + approved-doc intake (with sign-off flag) | 3 | Bucket created; manifest lists approved docs only | 103 |
| HLMCP-131 | Story | Ingestion pipeline: parse → chunk → embed (Vertex AI) → index (Vector Search) | 8 | Docs end up searchable; rerun is idempotent; logged | 130 |
| HLMCP-132 | Story | `search_homeloan_kb` RAG tool with citations | 5 | Returns top-k chunks + source doc citations; out-of-scope → safe "don't know" | 131,113 |

**E4 (S1) subtotal: 16 SP**

#### Epic E5 — LLM Clients (Claude first)

| Key | Type | Title | SP | Acceptance Criteria | Dep |
|-----|------|-------|----|---------------------|-----|
| HLMCP-140 | Story | Connect Claude (Desktop custom connector / remote MCP) to dev server | 3 | Claude calls EMI + KB search successfully; demo recorded | 112,132 |

**E5 (S1) subtotal: 3 SP**

#### Epic E6 — Security & Observability (base)

| Key | Type | Title | SP | Acceptance Criteria | Dep |
|-----|------|-------|----|---------------------|-----|
| HLMCP-150 | Story | API-key/OAuth auth middleware on MCP endpoint | 5 | Unauthenticated calls rejected; keys in Secret Manager | 112 |
| HLMCP-151 | Task | Base monitoring dashboard + uptime/error alerts | 3 | Cloud Monitoring dashboard + alert policies live | 114 |

**E6 (S1) subtotal: 8 SP**

#### Epic E8 — Testing (S1)

| Key | Type | Title | SP | Acceptance Criteria | Dep |
|-----|------|-------|----|---------------------|-----|
| HLMCP-160 | Story | Set up MCP Inspector workflow (local + hosted) for QA | 3 | QA can call all tools via Inspector without VS Code | 111 |
| HLMCP-161 | Story | Unit tests for calculators (EMI/eligibility) in CI | 3 | 100% calculator cases pass in Cloud Build | 120,121,104 |
| HLMCP-162 | Task | Automated MCP client smoke test in CI | 2 | CI lists + calls each tool, asserts output | 132,104 |

**E8 (S1) subtotal: 8 SP**

> **Sprint 1 total ≈ 86 SP across full team.** Sequence the critical path E1→E2→E3/E4→E5; QA (E8) runs in parallel. (If velocity is lower, defer HLMCP-123 fees-portion and HLMCP-151 to early Sprint 2.)

---

### 🟩 SPRINT 2 — Hardening, All Clients, Framework, Production

**Sprint 2 Goal:** *Production deployment with all three LLM clients working, full security/compliance sign-off, RAG quality at target, and a reusable multi-department framework + onboarding guide.*

#### Epic E4 — RAG Quality (continued)

| Key | Type | Title | SP | Acceptance Criteria | Dep |
|-----|------|-------|----|---------------------|-----|
| HLMCP-230 | Story | RAG eval harness + tuning (chunk size, top-k, reranking) | 5 | Groundedness ≥ 95% on labelled set; report published | 132 |
| HLMCP-231 | Story | Scheduled re-index job (Cloud Run Jobs/Workflows) | 3 | New/updated approved docs auto-reindex on schedule | 131 |
| HLMCP-232 | Story | Out-of-scope / refusal guardrails + safe fallback message | 3 | Out-of-scope questions return branch/RM referral, no hallucination | 132 |

**E4 (S2) subtotal: 11 SP**

#### Epic E5 — All LLM Clients

| Key | Type | Title | SP | Acceptance Criteria | Dep |
|-----|------|-------|----|---------------------|-----|
| HLMCP-240 | Story | ChatGPT/OpenAI integration (Agents SDK / Responses API / Custom GPT action) | 5 | ChatGPT answers home-loan Qs via MCP with auth; demo recorded | 250 |
| HLMCP-241 | Story | Gemini integration (Gen AI SDK / Vertex AI function bridge) | 5 | Gemini answers home-loan Qs via MCP; demo recorded | 250 |
| HLMCP-242 | Task | Deliver connector configs + setup docs for all 3 clients | 3 | Copy-paste configs documented for Claude/ChatGPT/Gemini | 240,241 |

**E5 (S2) subtotal: 13 SP**

#### Epic E6 — Security, Compliance & Observability (full)

| Key | Type | Title | SP | Acceptance Criteria | Dep |
|-----|------|-------|----|---------------------|-----|
| HLMCP-250 | Story | OAuth 2.1 / robust auth + per-department scopes | 5 | Scoped tokens; rotation; tested for bypass | 150 |
| HLMCP-251 | Story | Audit logging of every tool call (immutable sink) | 5 | Who/when/tool/args-hash/sources recorded; tamper-evident | 114 |
| HLMCP-252 | Story | PII redaction + prompt-injection defenses | 5 | Injection test suite passes; PII filtered in/out | 132 |
| HLMCP-253 | Task | Cloud Armor (WAF, rate-limit) + HTTPS LB in front of Cloud Run | 3 | WAF rules + rate limits active; verified | 112 |
| HLMCP-254 | Task | InfoSec + Compliance review gate | 3 | Sign-off recorded; findings closed | 250,251,252 |

**E6 (S2) subtotal: 21 SP**

#### Epic E7 — Reusable Multi-Department Framework

| Key | Type | Title | SP | Acceptance Criteria | Dep |
|-----|------|-------|----|---------------------|-----|
| HLMCP-260 | Story | Refactor into core vs department layout (template repo) | 5 | Home-loan runs purely as a department config on shared core | 113 |
| HLMCP-261 | Story | Parameterize Terraform per department (1 tfvars = 1 dept) | 5 | New dept stack stands up from a tfvars file only | 103,260 |
| HLMCP-262 | Story | "New Department" onboarding guide + scaffolding script | 3 | A dummy 2nd department onboarded in < 1 day, no core edits | 260,261 |
| HLMCP-263 | Task | Tag framework template `v1.0` + sample department example | 2 | Tagged repo; example dept builds & deploys | 262 |

**E7 subtotal: 15 SP**

#### Epic E8 — Testing & Acceptance (S2)

| Key | Type | Title | SP | Acceptance Criteria | Dep |
|-----|------|-------|----|---------------------|-----|
| HLMCP-270 | Story | End-to-end scripted Q&A test across all 3 clients | 5 | Same question set passes on Claude/ChatGPT/Gemini | 240,241 |
| HLMCP-271 | Story | Load test (k6/Locust) + autoscaling validation on Cloud Run | 3 | Meets latency/throughput target; scales under load | 253 |
| HLMCP-272 | Task | UAT with Home-Loan SMEs + sign-off | 3 | SMEs approve answer quality; issues logged/closed | 270 |

**E8 (S2) subtotal: 11 SP**

#### Epic E9 — Production Deployment & Handover

| Key | Type | Title | SP | Acceptance Criteria | Dep |
|-----|------|-------|----|---------------------|-----|
| HLMCP-280 | Story | Promote to production env (Terraform prod, blue/green) | 5 | Prod Cloud Run live, health checks pass, rollback tested | 254,261 |
| HLMCP-281 | Task | Runbook, on-call, SLOs, alerting handover to ops | 3 | Runbook published; alerts route to on-call | 280,151 |
| HLMCP-282 | Task | Final demo + project closeout docs | 2 | Recorded demo (3 clients); README/onboarding finalized | 280,272 |

**E9 subtotal: 10 SP**

> **Sprint 2 total ≈ 81 SP across full team.** Critical path: E6 security + E5 clients → E8 E2E → E9 production. E7 framework runs in parallel once E2/E3 stabilize.

---

## 12. Capacity & Velocity Notes
- Sprint totals (~86 and ~81 SP) assume the **whole team** of ~6 working in parallel for 14 days, not a single engineer. Per-engineer load ≈ 14–18 SP/sprint.
- If your real team is smaller, **descope** in this order: HLMCP-123 (fees split), HLMCP-241/Gemini → move to a Sprint 3, HLMCP-271 (load) → minimal smoke load test.
- Reorder freely; the **dependency column** is what must be respected, not the listed order.

## 13. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| MCP testing blocked in VS Code | Med | Use MCP Inspector + CI clients + native LLM apps (Section 8) — no VS Code dependency |
| LLM hallucinates loan advice | High | RAG-only answers + citations + refusal guardrails + SME UAT |
| EMI/eligibility math wrong | High | Validate to the paisa vs reference sheet; 100% unit-test gate |
| PII / data leakage | High | Phase-1 excludes account data; redaction; audit; InfoSec gate |
| One client (e.g. ChatGPT/Gemini) connector changes | Med | Server is client-agnostic; only connector config changes |
| Vendor cost overrun (Vertex AI) | Med | Budget alerts; caching; right-size embeddings/index |
| Scope creep from other departments | Med | Framework template + clear onboarding boundary (config/data only) |

## 14. RACI (summary)

| Activity | Responsible | Accountable | Consulted | Informed |
|----------|-------------|-------------|-----------|----------|
| Architecture | Tech Lead | Eng Manager | InfoSec, Cloud | Product |
| MCP server & tools | Backend Engineers | Tech Lead | ML Eng | QA |
| RAG pipeline | ML Engineer | Tech Lead | Home-Loan SMEs | QA |
| GCP infra & CI/CD | DevOps Engineer | Tech Lead | InfoSec | All |
| Security/compliance | InfoSec | CISO/Compliance | Tech Lead | Sponsor |
| UAT & content sign-off | Home-Loan SMEs | Product/BA | QA | Sponsor |

## 15. Definition of Done (per story)
- Code reviewed + merged; unit/integration tests pass in CI.
- Deployed to the target env via pipeline (no manual steps).
- Logging/metrics in place; no secrets in code.
- AC verified (via Inspector/CI/LLM client, not VS Code).
- Docs/runbook updated.

---

## Appendix A — Jira Import Checklist
1. Create project `HLMCP` (Scrum), 2 sprints of 14 working days.
2. Create 9 Epics (Section 10).
3. Create stories/tasks (Section 11) under their epics with SP + AC + dependency links (`blocks`/`is blocked by`).
4. Assign Sprint 1 / Sprint 2 as labelled.
5. Set Sprint goals (Section 11 headers).
6. Add components: `mcp-core`, `home-loan-tools`, `rag`, `gcp-infra`, `security`, `clients`.

## Appendix B — Key Reference Tools/Configs to Produce
- `claude_desktop_config.json` snippet (remote MCP connector).
- OpenAI Agents SDK `MCPServerStreamableHttp` snippet.
- Gemini Gen AI SDK MCP-session snippet.
- `departments/home_loan/config.yaml` example.
- MCP Inspector run command + saved test collection.
