# How It Works: MCP Server, Orchestrator & Prompting (Simple Version)

> A focused, no-frills walkthrough of the **request → tool → response** flow.
> User types a query → an LLM decides which of our **two tools** to call →
> the tool hits a TypeScript API → JSON comes back → we turn it into a friendly
> answer → it shows up in the UI.
>
> (Guardrails and evals are intentionally **not** covered here — see `PLAN.md` for those.)

We have two APIs:
- **Products API** — find products (name, price, stock…).
- **Affordability API** — check if a user can afford something / EMI options.

We expose each API as **one MCP tool**. That's it: two tools.

---

## 1. The Big Picture (one diagram)

```
   USER types a query
        │
        ▼
   ┌─────────┐      ┌──────────────────────────────┐      ┌───────────────┐
   │   UI    │ ───► │      ORCHESTRATOR (Python)   │ ───► │   LLM (local) │
   │(Streamlit)     │  - holds the prompt          │ ◄─── │   via Ollama  │
   └─────────┘ ◄─── │  - sends query + tool list   │      └───────────────┘
        ▲           │  - runs the chosen tool      │
        │           │  - turns JSON into an answer │
        │           └───────────────┬──────────────┘
        │                           │ "call this tool with these args"
   final answer                     ▼
                          ┌────────────────────┐
                          │  MCP SERVER (Py)   │   two tools live here:
                          │  - search_products │
                          │  - check_afford…   │
                          └─────────┬──────────┘
                                    │ HTTP (localhost)
                       ┌────────────┴────────────┐
                       ▼                          ▼
              ┌────────────────┐        ┌────────────────────┐
              │ Products API   │        │ Affordability API  │
              │ (TypeScript)   │        │ (TypeScript)       │
              └────────────────┘        └────────────────────┘
```

Three pieces of code we write:
1. **MCP server** — defines the two tools (each calls one TS API).
2. **Orchestrator** — the brain: talks to the LLM, runs tools, builds the answer.
3. **UI** — a simple chat box.

---

## 2. The MCP Server (where the two tools live)

An MCP server is just a small program that **advertises a list of tools**. Each tool has:
- a **name** (e.g. `search_products`),
- a **description** ("use this when the user wants to find products…"),
- **typed parameters** (what arguments it needs),
- and the **code** that runs when it's called (here: call the TypeScript API).

```python
# mcp_server.py
from mcp.server.fastmcp import FastMCP
import httpx

mcp = FastMCP("commerce-tools")

@mcp.tool()
async def search_products(query: str, brand: str | None = None,
                          price_max: float | None = None, limit: int = 5) -> dict:
    """Find products in the catalog. Use when the user wants to search,
    browse, compare, or get the price of products."""
    body = {"query": query, "brand": brand, "price_max": price_max, "limit": limit}
    async with httpx.AsyncClient() as c:
        r = await c.post("http://localhost:4001/api/products/search", json=body)
        return r.json()          # <-- raw JSON from the TS API

@mcp.tool()
async def check_affordability(price: float, monthly_income: float,
                              monthly_expenses: float, tenure_months: int = 6) -> dict:
    """Check if the user can afford a product and show EMI options. Use when
    the user asks 'can I afford…', about budget, EMI, or monthly payments."""
    body = {"price": price, "tenure_months": tenure_months,
            "customer": {"monthly_income": monthly_income,
                         "monthly_expenses": monthly_expenses}}
    async with httpx.AsyncClient() as c:
        r = await c.post("http://localhost:4002/api/affordability/check", json=body)
        return r.json()          # <-- raw JSON from the TS API

if __name__ == "__main__":
    mcp.run()   # runs locally over "stdio" so the orchestrator can talk to it
```

**Key idea:** the tool's **name + description + parameter names** are what the LLM
reads to decide *whether* and *how* to use it. So write them clearly — this is the
first place "prompt engineering" shows up (see Section 5).

The tool returns the **raw JSON** from the TS API. It does **not** write the user-facing
sentence — that happens in the orchestrator.

---

## 3. How a Tool Gets Triggered (and the options)

The orchestrator hands the user's query **and the list of tools** to the LLM. The LLM
then "triggers" a tool. There are basically **three ways** this triggering can be decided:

### Option A — LLM native tool-calling (recommended, simplest)
Modern LLMs support "function/tool calling": you give the model the tool schemas, and
**the model itself** returns a structured message like:

```json
{ "tool": "search_products", "arguments": { "query": "headphones", "brand": "Sony", "price_max": 200 } }
```

The orchestrator sees that, runs the matching tool, and feeds the result back. **No
manual if/else.** This is what we use.

### Option B — Keyword / rule-based routing (no LLM for routing)
You write simple rules: if the query contains "afford / EMI / budget" → call
`check_affordability`; if it contains "find / show / price" → call `search_products`.
Fast and predictable, but brittle — fails on phrasing it didn't expect. Fine as a
fallback, not as the main approach.

### Option C — LLM "router" prompt that outputs a label
You ask the LLM a narrow question: *"Which tool fits: products, affordability, or none?
Reply with just the label."* Then the orchestrator calls that tool. This is a middle
ground — flexible like A, but you handle argument extraction yourself.

> **We go with Option A.** It handles varied phrasing *and* fills in the arguments in one step.

---

## 4. How We Decide WHICH Tool (varied user queries)

The user can phrase things many ways. The LLM matches the **meaning** of the query to the
**tool descriptions**. Examples:

| User says | Tool the LLM picks | Why |
|---|---|---|
| "Show me Sony headphones under $200" | `search_products` | wants to find products |
| "Got any cheap wireless earbuds?" | `search_products` | same intent, different words |
| "Can I afford a $198 item if I make 3000 and spend 2100?" | `check_affordability` | all numbers given |
| "What's the EMI on a 500 dollar phone?" | `check_affordability` | EMI/budget intent |
| "Can I afford the cheapest Sony headphones?" | `search_products` → then `check_affordability` | needs the price first, then the check |
| "Hi" / "what's the weather" | **no tool** | LLM just answers / says it can't help |

Two things make this reliable:
1. **Clear tool descriptions** ("use this when the user wants to…").
2. If a required argument is missing (e.g. affordability needs income), the orchestrator
   has the LLM **ask a follow-up question** instead of guessing.

For the chained case ("afford the cheapest Sony headphones"), the orchestrator simply
loops: LLM calls `search_products`, gets the price, then the LLM calls
`check_affordability` with that price. We cap the loop at a few steps so it can't run forever.

---

## 5. Where Prompt Engineering Comes In (3 places)

Prompting isn't one giant prompt — it appears in **three small, specific spots**:

1. **Tool descriptions (in the MCP server).** The "use this when…" docstrings are prompts
   the LLM reads to route correctly. Good descriptions = correct tool choice.

2. **The system prompt (in the orchestrator).** A short instruction that sets behavior:
   > "You are a shopping assistant. You can search products and check affordability.
   > Use a tool when the user's request needs live data. If you're missing required
   > info, ask a short follow-up question. Only state facts that appear in the tool's
   > result."

3. **The answer-formatting prompt (after the tool runs).** Once we have the JSON, we ask
   the LLM to turn it into a friendly reply:
   > "Here is the user's question and the JSON result from the tool. Write a clear,
   > short answer using only the data in the JSON. Don't invent prices or products."

That's the whole role of prompt engineering in this simple app: **route well, behave well,
summarize faithfully.**

---

## 6. How the Response Goes Back to the UI

This is the part you specifically asked about. After a tool is triggered and returns JSON,
here's the round trip:

```
LLM picks tool ──► Orchestrator runs MCP tool ──► MCP server calls TS API
                                                        │
                                                JSON result returns
                                                        │
                        Orchestrator gets the JSON ◄────┘
                                  │
                                  ▼
              Orchestrator builds the user-facing answer
              (one of two ways — see below)
                                  │
                                  ▼
                    Answer string returned to the UI
                                  │
                                  ▼
                     UI displays it in the chat box
```

**Two ways to turn JSON into the final answer:**

- **(a) LLM summarization** — give the JSON back to the LLM with the formatting prompt
  (Section 5.3) and let it write a natural sentence. Best for product lists:
  > *"I found 2 Sony options: WH-CH520 at $49.99 and WH-1000XM4 at $198 — both in stock."*

- **(b) Template formatting (no LLM)** — for numbers/affordability, just plug values into
  a fixed sentence so the math is always exact:
  > *"Yes — at $198 your 6-month EMI is $34.32/mo, which fits your budget."*

Either way, the orchestrator returns a **plain string** (or a small structured object) to
the UI, and the UI prints it. Optionally the UI also shows the **raw JSON** behind a
"details" toggle so the user can see where the answer came from.

### What the orchestrator loop looks like (pseudo-code)

```python
def handle(user_query):
    messages = [system_prompt, user_query]

    while True:
        reply = llm.chat(messages, tools=[search_products, check_affordability])

        if reply.wants_tool:                       # Option A: native tool-calling
            result_json = run_mcp_tool(reply.tool, reply.arguments)
            messages.append(tool_result(result_json))
            continue                               # let the LLM use the result / chain

        return reply.text                          # final natural-language answer → UI
```

So the flow is: **query → LLM (pick tool) → run tool → JSON → LLM (write answer) → UI.**
The same `llm.chat` call both *picks the tool* and later *writes the answer* — we just keep
looping until the LLM responds with plain text instead of a tool request.

---

## 7. Recap (the whole thing in 6 lines)

1. **MCP server** exposes two tools, each calling one TypeScript API and returning JSON.
2. **Orchestrator** sends the user query + tool list to a **local LLM** (Ollama).
3. The **LLM picks the right tool** (native tool-calling) and fills in the arguments.
4. The orchestrator **runs the tool**, the TS API returns **JSON**.
5. The orchestrator **turns the JSON into an answer** (LLM summary or a template).
6. The answer string goes back to the **UI** and is shown to the user.

That's the entire simple application.
