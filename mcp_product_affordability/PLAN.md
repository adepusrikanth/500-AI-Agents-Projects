# Plan: LLM + MCP Assistant over Product & Affordability APIs

> **Goal**: A user types a natural-language query into a UI. An LLM understands the
> intent, picks the right tool, calls one of two existing **TypeScript** APIs
> (`Products` and `Affordability`) with a valid JSON payload, and returns a clean,
> grounded answer. The whole stack runs **locally — no cloud resources**, and the
> orchestration/agent code is written in **Python** using **MCP (Model Context Protocol)**.

This document is written from a senior-architect point of view. It covers:

1. The two APIs (with concrete JSON in/out contracts)
2. Why MCP, and the end-to-end architecture
3. The Python library stack (all local)
4. How intent → correct tool → API call → grounded answer works
5. Guardrails (input, routing, output, safety, cost/latency)
6. Evaluations (tool-selection, argument, end-to-end, regression, online)
7. Build phases / milestones
8. Risks and open questions

---

## 1. The Two APIs (TypeScript)

These two HTTP services already exist and are written in TypeScript (e.g. Express/Nest/Fastify).
We do **not** rewrite them. We wrap them as **MCP tools** so the LLM can call them.

### 1.1 Products API

Returns product catalog data filtered by the caller's criteria.

**Endpoint**: `POST http://localhost:4001/api/products/search`

**Input JSON (request body):**

```json
{
  "query": "wireless headphones",
  "category": "electronics",
  "price_min": 0,
  "price_max": 200,
  "brand": "Sony",
  "in_stock_only": true,
  "sort_by": "price_asc",
  "limit": 5
}
```

**Output JSON (response body):**

```json
{
  "count": 2,
  "currency": "USD",
  "results": [
    {
      "product_id": "P-10231",
      "name": "Sony WH-CH520 Wireless Headphones",
      "brand": "Sony",
      "category": "electronics",
      "price": 49.99,
      "in_stock": true,
      "rating": 4.4,
      "url": "https://store.local/p/P-10231"
    },
    {
      "product_id": "P-10987",
      "name": "Sony WH-1000XM4",
      "brand": "Sony",
      "category": "electronics",
      "price": 198.0,
      "in_stock": true,
      "rating": 4.8,
      "url": "https://store.local/p/P-10987"
    }
  ]
}
```

### 1.2 Affordability API

Given a product/price and the user's financial context, returns whether they can
afford it and which financing/EMI options apply.

**Endpoint**: `POST http://localhost:4002/api/affordability/check`

**Input JSON (request body):**

```json
{
  "product_id": "P-10987",
  "price": 198.0,
  "currency": "USD",
  "customer": {
    "monthly_income": 3000,
    "monthly_expenses": 2100,
    "existing_emi": 150,
    "credit_score": 720
  },
  "tenure_months": 6
}
```

**Output JSON (response body):**

```json
{
  "affordable": true,
  "max_affordable_price": 540.0,
  "recommended_option": "emi_6m",
  "options": [
    { "type": "full_payment", "monthly": 198.0, "months": 1, "interest_rate": 0.0 },
    { "type": "emi_3m", "monthly": 67.32, "months": 3, "interest_rate": 0.04 },
    { "type": "emi_6m", "monthly": 34.32, "months": 6, "interest_rate": 0.06 }
  ],
  "reasoning": "Disposable income (750/mo) comfortably covers the 6-month EMI of 34.32."
}
```

> These payloads are the **contract**. Both the MCP tool schemas (Section 4) and the
> eval datasets (Section 6) are derived directly from them. If the real APIs differ,
> update this section first — it is the single source of truth.

---

## 2. Why MCP + High-Level Architecture

### 2.1 Why MCP at all?

MCP (Model Context Protocol) is an open standard for exposing **tools/resources** to
LLM clients over a uniform interface (stdio or HTTP). It is the right fit here because:

- **Decoupling**: The TypeScript APIs stay as-is. We add a thin MCP server that
  presents them as typed tools. Tooling is reusable across any MCP-compatible client
  (our Python app, Claude Desktop, Cursor, etc.).
- **Typed tool schemas**: Each tool declares a JSON Schema for its arguments. The LLM
  gets exactly the contract it must satisfy, which makes routing + argument generation
  far more reliable than free-form prompting.
- **Separation of concerns**: The MCP server owns "how to call the API"; the agent
  owns "which tool and why". Guardrails and evals plug in cleanly at the boundary.
- **Local-first**: MCP runs perfectly over stdio or localhost HTTP — no cloud needed.

### 2.2 Component diagram

```
            ┌──────────────────────────────────────────────────────────┐
            │                        USER                               │
            │             "Can I afford Sony XM4 on EMI?"               │
            └───────────────────────────┬──────────────────────────────┘
                                         │  (HTTP / WebSocket)
                                         ▼
            ┌──────────────────────────────────────────────────────────┐
            │   UI  (Streamlit / Gradio / Chainlit — local web app)     │
            └───────────────────────────┬──────────────────────────────┘
                                         │  Python call
                                         ▼
            ┌──────────────────────────────────────────────────────────┐
            │           AGENT / ORCHESTRATOR  (Python)                  │
            │  - Input guardrails                                       │
            │  - LLM with tool-calling (intent → tool selection)        │
            │  - MCP client (talks to MCP server)                       │
            │  - Output guardrails + grounding                          │
            └───────────────┬──────────────────────────┬───────────────┘
                            │ MCP (stdio/HTTP)          │
                            ▼                           ▼
            ┌────────────────────────────┐  ┌────────────────────────────┐
            │  MCP SERVER (Python)       │  │   LOCAL LLM RUNTIME         │
            │  Tool: search_products     │  │   Ollama (e.g. llama3.1,    │
            │  Tool: check_affordability │  │   qwen2.5, mistral) — local │
            └─────────────┬──────────────┘  └────────────────────────────┘
                          │ HTTP (localhost)
            ┌─────────────┴──────────────┐
            ▼                            ▼
   ┌──────────────────┐        ┌──────────────────────┐
   │ Products API     │        │ Affordability API    │
   │ (TypeScript)     │        │ (TypeScript)         │
   │ :4001            │        │ :4002                │
   └──────────────────┘        └──────────────────────┘
```

### 2.3 Request lifecycle (happy path)

1. User submits a query in the UI.
2. **Input guardrails** run (length, PII, prompt-injection, off-topic).
3. The agent sends the query + tool schemas to the local LLM.
4. The LLM emits a **tool call** (`search_products` or `check_affordability`) with JSON args.
5. The agent validates args against the JSON Schema, then invokes the tool via the **MCP client**.
6. The MCP server calls the corresponding **TypeScript API** over localhost HTTP.
7. The API's JSON response flows back → MCP server → agent.
8. The LLM (or a deterministic formatter) turns the JSON into a grounded NL answer.
9. **Output guardrails** run (grounding/hallucination check, formatting, safety).
10. The UI renders the answer + optionally the raw tool result for transparency.

> A single query may chain tools: e.g. "Can I afford the cheapest Sony headphones?"
> → `search_products` (find product + price) → `check_affordability` (use that price).
> This is multi-step agentic behavior; keep a small max-step budget (Section 5).

---

## 3. Python Library Stack (all local)

| Concern | Recommended | Why / Notes |
|---|---|---|
| **MCP** | `mcp` (official Python SDK) | Build the MCP server + client. Use `FastMCP` for ergonomic tool definitions. |
| **Local LLM runtime** | **Ollama** (`ollama` Python client) | Runs `llama3.1:8b`, `qwen2.5:7b`, `mistral` etc. locally; OpenAI-compatible endpoint. No cloud. |
| **Agent / tool-calling glue** | Choose ONE: `langgraph` + `langchain-mcp-adapters`, **or** `pydantic-ai`, **or** plain `openai` SDK pointed at Ollama | `pydantic-ai` is lightweight and has first-class MCP + typed outputs. LangGraph is best if you want explicit multi-step graphs. |
| **Schema / validation** | `pydantic` v2 | Tool arg models, output models, guardrail contracts. Single source of truth → JSON Schema. |
| **HTTP client (MCP→TS API)** | `httpx` | Async, timeouts, retries to the TypeScript services. |
| **UI** | **Streamlit** (simplest) or `gradio` / `chainlit` | All run locally in the browser. Chainlit is chat-native; Streamlit is fastest to ship. |
| **Guardrails** | `pydantic` + lightweight rules; optionally `guardrails-ai` or `nemoguardrails` (local) | Start with deterministic rules; add a library only if needed. |
| **Evals** | `pytest` + `deepeval` (local) or `promptfoo` (CLI), `ragas` optional | Run offline against fixtures; assert tool choice + args + answer quality. |
| **Tracing/observability** | `langfuse` (self-hosted/local) or OpenTelemetry + console | Inspect tool calls, latency, token usage during dev + eval. |
| **Config / secrets** | `python-dotenv`, `pydantic-settings` | Local `.env` for ports, model name, thresholds. |
| **Packaging / deps** | `uv` (fast) or `poetry` | Reproducible local env. |

**Recommended minimal core** (to avoid over-engineering):
`mcp`, `pydantic`, `httpx`, `ollama`, `pydantic-ai` (agent + MCP client), `streamlit` (UI),
`pytest` + `deepeval` (evals). Add LangGraph only if multi-step orchestration grows complex.

---

## 4. Intent → Correct Tool → API → Answer

### 4.1 The two MCP tools

Defined in the Python MCP server with `FastMCP`. Each wraps one TS API call.

```python
# mcp_server.py  (illustrative)
from mcp.server.fastmcp import FastMCP
import httpx

mcp = FastMCP("commerce-tools")
PRODUCTS_URL = "http://localhost:4001/api/products/search"
AFFORD_URL   = "http://localhost:4002/api/affordability/check"

@mcp.tool()
async def search_products(
    query: str,
    category: str | None = None,
    price_min: float | None = None,
    price_max: float | None = None,
    brand: str | None = None,
    in_stock_only: bool = False,
    sort_by: str = "relevance",
    limit: int = 5,
) -> dict:
    """Search the product catalog. Use when the user asks to find, compare,
    or get details/prices of products."""
    payload = {k: v for k, v in locals().items() if v is not None}
    async with httpx.AsyncClient(timeout=10) as c:
        r = await c.post(PRODUCTS_URL, json=payload)
        r.raise_for_status()
        return r.json()

@mcp.tool()
async def check_affordability(
    price: float,
    monthly_income: float,
    monthly_expenses: float,
    existing_emi: float = 0.0,
    credit_score: int | None = None,
    tenure_months: int = 6,
    product_id: str | None = None,
    currency: str = "USD",
) -> dict:
    """Check whether the user can afford a product and which EMI/financing
    options apply. Use when the user asks about affordability, EMI, budget,
    or 'can I afford X'."""
    payload = {
        "product_id": product_id, "price": price, "currency": currency,
        "tenure_months": tenure_months,
        "customer": {
            "monthly_income": monthly_income,
            "monthly_expenses": monthly_expenses,
            "existing_emi": existing_emi,
            "credit_score": credit_score,
        },
    }
    async with httpx.AsyncClient(timeout=10) as c:
        r = await c.post(AFFORD_URL, json=payload)
        r.raise_for_status()
        return r.json()

if __name__ == "__main__":
    mcp.run()  # stdio transport for local use
```

Key design choices:
- **Docstrings are routing signal.** The LLM uses the tool name + description +
  parameter names to decide intent. Write them as crisp "use when…" instructions.
- **Flatten arguments** the LLM must produce (e.g. `monthly_income` rather than a nested
  `customer` object) — LLMs fill flat typed params more reliably. The tool re-nests
  before calling the TS API.
- **Defaults & optionals** reduce required slots the LLM must guess.

### 4.2 How routing works

The LLM does the routing via **native tool/function calling** — not regex/keyword
matching. Flow:

1. The agent passes both tool schemas to the model.
2. The model returns either a direct answer (no tool needed) or one tool call with args.
3. If required args are missing, the agent **asks a clarifying question** instead of guessing
   (e.g. affordability needs income/expenses — if absent, ask).
4. For chained intents, allow the model to call `search_products` first, feed the result
   back, then call `check_affordability`. Cap at **N steps** (e.g. 4).

Intent → tool mapping examples:

| User query | Tool(s) | Notes |
|---|---|---|
| "Show me Sony headphones under $200" | `search_products` | filters: brand, price_max |
| "Can I afford a $198 item if I earn 3000 and spend 2100?" | `check_affordability` | direct args present |
| "Can I afford the cheapest Sony headphones?" | `search_products` → `check_affordability` | chained; needs income → clarify if missing |
| "What's the weather?" | none | off-topic → refuse politely (guardrail) |

### 4.3 Answer generation

Two valid strategies (pick per tool):
- **Deterministic formatting** for affordability numbers (avoid LLM math errors): the
  agent templates the answer directly from JSON.
- **LLM summarization** for product lists, but **strictly grounded**: instruct "only use
  fields present in the tool result; never invent prices/availability." Then run the
  grounding guardrail (Section 5).

---

## 5. Guardrails (in detail)

Guardrails are layered: **input → routing → tool-call → output → system**.

### 5.1 Input guardrails (before the LLM)
- **Length / rate limits**: cap query length and per-session request rate.
- **PII handling**: financial inputs (income, credit score) are sensitive. Do not log raw
  values; redact in traces. Keep them in-memory only (local).
- **Prompt-injection / jailbreak detection**: scan for "ignore previous instructions",
  attempts to exfiltrate the system prompt, or to make the agent call internal URLs.
- **Topic/scope filter**: only product & affordability intents are in scope. Off-topic →
  polite refusal with a hint of supported capabilities.

### 5.2 Routing guardrails
- **Schema-constrained tool calls**: reject any tool call whose args fail Pydantic/JSON-Schema
  validation; re-prompt the model with the validation error (self-correction loop, max 2 retries).
- **Required-slot check**: if a mandatory field is missing (e.g. income for affordability),
  ask a clarifying question rather than hallucinating a value.
- **No-tool fallback**: if confidence is low or the query is ambiguous between tools, ask the
  user to disambiguate instead of guessing.
- **Step budget**: hard cap on tool-call iterations to prevent loops/runaway cost.

### 5.3 Tool-execution guardrails (MCP server)
- **Allowlist URLs**: the MCP server may only call the two known localhost endpoints — no
  arbitrary URLs (prevents SSRF if the LLM tries to supply one).
- **Timeouts + retries + circuit breaker**: bounded `httpx` timeouts; limited retries with
  backoff; fail fast with a clean error message if a TS API is down.
- **Value sanity bounds**: clamp/validate numeric ranges (e.g. `price >= 0`, `limit <= 50`,
  `tenure_months` in {1,3,6,12,…}) before hitting the API.
- **Idempotency / read-only**: both tools are reads; no destructive operations exposed.

### 5.4 Output guardrails (after the tool result)
- **Grounding / anti-hallucination**: verify that every concrete fact in the answer
  (price, product name, EMI amount, affordable yes/no) exists in the tool JSON. If not,
  drop it or regenerate. This is the single most important guardrail.
- **No fabricated tools/fields**: never reference products or numbers not returned.
- **Numeric integrity**: prefer deterministic templating for money math; if the LLM restates
  numbers, cross-check against JSON.
- **Tone & disclaimers**: affordability output should include a non-advice disclaimer
  ("informational estimate, not financial advice").
- **Format contract**: structured output (e.g. summary + bullet list + optional raw JSON toggle).

### 5.5 System guardrails
- **Determinism for evals**: low temperature (0–0.2) for tool selection.
- **Observability**: log tool chosen, args (redacted), latency, retries, final answer for every
  request to enable evals and debugging.
- **Graceful degradation**: if LLM or API fails, return a clear, actionable error, never a stack trace.

---

## 6. Evaluations (in detail)

Evals are split into **offline** (CI / pre-merge against fixtures) and **online**
(observed live behavior). All run locally.

### 6.1 What to measure (metrics)

| Layer | Metric | Definition |
|---|---|---|
| Tool selection | **Routing accuracy** | % of queries where the correct tool (or "no tool") is chosen |
| Tool selection | **Confusion matrix** | products vs affordability vs none — find systematic mis-routes |
| Arguments | **Argument exact/semantic match** | Are extracted params correct (brand, price_max, income…)? |
| Arguments | **Schema validity rate** | % tool calls that pass JSON-Schema first try |
| Chaining | **Plan correctness** | For multi-step queries, is the tool sequence right? |
| End-to-end | **Answer correctness** | Does the final answer match expected (grounded in API output)? |
| End-to-end | **Faithfulness / hallucination rate** | % of answers with facts not supported by tool JSON |
| Clarification | **Appropriate-clarification rate** | Asks when info missing; doesn't ask when info present |
| Guardrails | **Refusal accuracy** | Off-topic/unsafe correctly refused; in-scope not over-refused |
| Ops | **Latency p50/p95, tokens, retries** | Per query and per tool |
| Robustness | **Injection-resistance** | % of adversarial prompts that fail to subvert routing/output |

### 6.2 Datasets / fixtures

- **Golden set (50–150 queries)** labeled with: expected tool(s), expected args,
  expected answer (or key facts). Cover: pure product, pure affordability, chained,
  ambiguous, missing-slot, off-topic, adversarial/injection.
- **Mock the TS APIs** for deterministic evals: a local stub that returns fixed JSON for
  fixed inputs, so answer-correctness is reproducible and CI doesn't depend on the live services.
- **Paraphrase robustness set**: same intent worded many ways (typos, slang, multilingual if needed).

### 6.3 How to run them

- **Unit / component**: `pytest` for tool functions (arg → correct API payload), guardrail
  rules, formatters.
- **LLM-as-judge + assertions**: `deepeval` (local) for faithfulness, answer relevancy,
  correctness; or `promptfoo` YAML test suites for routing/regression with assertions
  (`is-json`, `contains`, `llm-rubric`).
- **Routing eval**: programmatic — call the agent, capture the chosen tool + args, compare
  to labels; compute accuracy + confusion matrix. No LLM judge needed (it's deterministic).
- **Regression gate in CI**: fail the build if routing accuracy or faithfulness drops below
  thresholds (e.g. routing ≥ 0.95, hallucination ≤ 0.02 on the golden set).

### 6.4 Online evals (post-deploy, still local)
- **Trace sampling**: log a sample of real sessions (redacted) and periodically score with the
  judge to detect drift.
- **Implicit signals**: clarification frequency, retry rate, fallbacks, user retries → proxy for quality.
- **Optional thumbs up/down** in the UI feeding a feedback table.

### 6.5 Eval-specific guardrail tests
Explicitly include adversarial cases: prompt injection in the query, attempts to make the
agent call non-allowlisted URLs, requests for financial "advice", and numeric edge cases
(zero income, negative price, huge price). Assert the system refuses/clamps correctly.

---

## 7. Build Phases / Milestones

1. **Contracts & stubs**: lock the JSON schemas in Section 1; build local mock servers for
   both TS APIs so Python work isn't blocked.
2. **MCP server**: implement `search_products` + `check_affordability` tools over `httpx`;
   verify with the MCP Inspector / a simple client.
3. **Agent core**: wire Ollama + `pydantic-ai` (or LangGraph) as MCP client; get single-tool
   routing working end-to-end.
4. **Chaining + clarification**: enable multi-step (search → affordability) and missing-slot
   clarification with the step budget.
5. **Guardrails**: add input/routing/output guardrails incrementally; each with a unit test.
6. **UI**: Streamlit/Chainlit chat with answer + "show tool result" transparency toggle.
7. **Evals**: build golden set + mocks; wire `pytest`/`deepeval`/`promptfoo`; set CI thresholds.
8. **Hardening**: tracing (Langfuse local), latency tuning, model selection, regression suite.
9. **Swap mocks for real TS APIs**: point the MCP server at the live localhost services; re-run evals.

---

## 8. Risks & Open Questions

- **Local LLM tool-calling quality**: smaller local models route less reliably. Mitigate with
  clear tool descriptions, flat args, schema-constrained decoding, and a self-correction retry.
  Validate model choice (e.g. `qwen2.5`, `llama3.1`) against the routing eval before committing.
- **Numeric reasoning**: never trust the LLM for EMI/affordability math — keep it deterministic.
- **Schema drift**: if the TS APIs change, the MCP tool schemas + eval fixtures must update in lockstep.
  Treat Section 1 as the contract of record.
- **Latency**: local models + chaining can be slow; cache product lookups within a session,
  stream tokens to the UI, keep the step budget tight.
- **PII/financial data**: keep everything in memory, redact logs, add the non-advice disclaimer.

---

### TL;DR

Wrap the two TypeScript APIs as **two MCP tools** in a small **Python MCP server**. Run a
**local LLM via Ollama**. A **Python agent** (`pydantic-ai`/LangGraph) acts as the MCP client,
uses the LLM's **native tool-calling to route intent** to the right tool, validates args with
**Pydantic**, calls the API, and returns a **grounded** answer through a **Streamlit** UI.
Wrap it in **layered guardrails** (input/routing/tool/output) and gate quality with **offline
evals** (routing accuracy, argument correctness, faithfulness) plus light **online** monitoring —
all running locally with no cloud resources.
