# How to Develop the Orchestrator (Step by Step)

> The **orchestrator** is the brain that sits between the UI and everything else.
> Its job: take the user's query → ask the LLM which tool to use → run that tool via
> the **MCP client** → feed the JSON back → get a final answer → return it to the UI.
>
> This guide builds it from scratch, in small steps, with runnable Python.
> Keep it simple — it's basically **one loop**.

---

## 0. What the orchestrator must do (its responsibilities)

1. **Connect to the MCP server** and discover the available tools.
2. **Hold the prompts** (system prompt + formatting behavior).
3. **Call the LLM** with the user query + tool list.
4. **Run the tool** the LLM asks for (through the MCP client).
5. **Loop** until the LLM returns a plain-text answer (handles chaining).
6. **Return the final answer** (a string) to the UI.

That's it. Everything below is just filling in these six steps.

---

## 1. Prerequisites (install + run locally)

```bash
# Python deps (use uv or pip)
pip install mcp ollama httpx pydantic streamlit

# Local LLM runtime
#   1. install Ollama from https://ollama.com (runs locally, no cloud)
#   2. pull a model that supports tool-calling:
ollama pull qwen2.5          # or llama3.1 ; both do tool-calling well
```

You also need:
- The **MCP server** running (the `mcp_server.py` with `search_products` + `check_affordability`).
- The two **TypeScript APIs** running on `localhost:4001` and `localhost:4002`
  (or mock them while developing).

---

## 2. Pick how the orchestrator talks to the LLM + tools

You have two realistic local setups. Both work; pick one.

| Approach | What you write | When to choose |
|---|---|---|
| **A. DIY loop** (Ollama client + MCP client) | ~60 lines, full control, easy to understand | Learning, max transparency — **used in this guide** |
| **B. Framework** (`pydantic-ai` or LangGraph + `langchain-mcp-adapters`) | Less code, handles the loop for you | When the app grows or you want batteries included |

We'll build **Approach A** so you understand every moving part, then point to B at the end.

---

## 3. Step-by-step build (DIY loop)

### Step 3.1 — Connect the MCP client and list tools

The orchestrator launches/connects to the MCP server and asks "what tools do you have?".
Those tool definitions are converted into the format the LLM expects.

```python
# orchestrator.py
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

# Tell the client how to start our MCP server (stdio = local subprocess)
server = StdioServerParameters(command="python", args=["mcp_server.py"])

async def list_tools(session: ClientSession):
    resp = await session.list_tools()
    # Convert MCP tool defs -> OpenAI/Ollama "tools" schema
    tools = []
    for t in resp.tools:
        tools.append({
            "type": "function",
            "function": {
                "name": t.name,
                "description": t.description,
                "parameters": t.inputSchema,   # MCP already gives JSON Schema
            },
        })
    return tools
```

> **Key point:** the LLM never talks to the API or the MCP server directly. The
> orchestrator is the only thing that does. The LLM just *names* a tool and *gives args*.

### Step 3.2 — Write the system prompt (the orchestrator's behavior)

Keep it short and specific. This is where you tell the model how to behave.

```python
SYSTEM_PROMPT = """You are a shopping assistant.
You have two tools: search_products (find products/prices) and
check_affordability (check budget / EMI options).

Rules:
- Use a tool whenever the request needs live product or affordability data.
- If a required value is missing (e.g. income for affordability), ask one short
  follow-up question instead of guessing.
- When you answer, only state facts that appear in the tool's result.
- Keep answers short and friendly."""
```

### Step 3.3 — The core loop (pick tool → run tool → repeat → answer)

This is the heart of the orchestrator. The same chat call both **chooses the tool**
and later **writes the final answer**; we loop until it stops asking for tools.

```python
import json
import ollama

async def handle_query(user_query: str) -> str:
    async with stdio_client(server) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = await list_tools(session)

            messages = [
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": user_query},
            ]

            MAX_STEPS = 4           # stop runaway loops / chaining limit
            for _ in range(MAX_STEPS):
                resp = ollama.chat(model="qwen2.5", messages=messages, tools=tools)
                msg = resp["message"]

                # No tool requested -> this is the final answer for the UI
                if not msg.get("tool_calls"):
                    return msg["content"]

                # Otherwise, run each requested tool via MCP and feed results back
                messages.append(msg)
                for call in msg["tool_calls"]:
                    name = call["function"]["name"]
                    args = call["function"]["arguments"]
                    if isinstance(args, str):
                        args = json.loads(args)

                    result = await session.call_tool(name, args)   # MCP -> TS API
                    result_json = result.content[0].text           # raw JSON string

                    messages.append({
                        "role": "tool",
                        "name": name,
                        "content": result_json,
                    })

            return "Sorry, I couldn't complete that in a few steps. Please rephrase."
```

What just happened, in words:
1. We give the LLM the query + tool list.
2. If it returns **no tool call** → that text *is* the answer → return to UI.
3. If it returns a **tool call** → we run it through the MCP client → get JSON →
   append the JSON as a `tool` message → loop again so the LLM can use it.
4. On the next loop the LLM either calls another tool (chaining) or writes the final answer.

### Step 3.4 — How the answer gets generated

Two valid styles (you can mix per tool):

- **Let the LLM write it** (default above): after the tool result is fed back, the LLM's
  next reply is the natural-language answer. Good for product lists.
- **Template it yourself** for numbers (so math is always exact): instead of looping back
  to the LLM, format the JSON directly:

```python
def format_affordability(j: dict) -> str:
    opt = next(o for o in j["options"] if o["type"] == j["recommended_option"])
    verdict = "Yes" if j["affordable"] else "No"
    return f"{verdict} — your {opt['months']}-month EMI is ${opt['monthly']}/mo."
```

Use templating when correctness of numbers matters; use the LLM when you want a natural,
flexible summary.

### Step 3.5 — Connect it to the UI

The UI just calls `handle_query` and prints the string. Example with Streamlit:

```python
# app.py
import asyncio, streamlit as st
from orchestrator import handle_query

st.title("Shopping Assistant")

if prompt := st.chat_input("Ask about products or affordability..."):
    st.chat_message("user").write(prompt)
    answer = asyncio.run(handle_query(prompt))
    st.chat_message("assistant").write(answer)
```

```bash
streamlit run app.py        # opens locally in the browser
```

That's the full loop: **UI → orchestrator → LLM → MCP tool → TS API → JSON → answer → UI.**

---

## 4. Full file layout

```
mcp_product_affordability/
├── mcp_server.py      # defines the 2 tools (calls the TS APIs)
├── orchestrator.py    # the brain: MCP client + LLM loop  (this guide)
├── app.py             # Streamlit UI
└── .env               # ports, model name, etc. (optional)
```

Run order while developing:
1. Start the TS APIs (or mocks) on :4001 / :4002.
2. `ollama serve` is running and the model is pulled.
3. `streamlit run app.py` — the orchestrator auto-launches `mcp_server.py` over stdio.

---

## 5. Common things you'll tweak

- **Model choice**: if routing is wrong, try `llama3.1` vs `qwen2.5`, and improve tool
  descriptions in `mcp_server.py` (the LLM routes off those).
- **Temperature**: keep it low (`options={"temperature": 0}` in `ollama.chat`) so tool
  choice is consistent.
- **Missing arguments**: the system prompt already tells the LLM to ask a follow-up;
  that follow-up question is just a normal no-tool reply that goes to the UI.
- **Chaining depth**: raise/lower `MAX_STEPS`.
- **Showing your work**: in the UI, optionally print the raw tool JSON behind a toggle.

---

## 6. Approach B (if you'd rather not write the loop)

A framework gives you the loop, MCP wiring, and typed outputs for free:

- **`pydantic-ai`**: define the model + attach the MCP server; it handles tool-calling and
  returns typed results. Least code.
- **LangGraph + `langchain-mcp-adapters`**: load MCP tools as LangChain tools and use a
  prebuilt ReAct agent; best when you want explicit multi-step graphs/state.

Sketch with `pydantic-ai`:

```python
from pydantic_ai import Agent
from pydantic_ai.mcp import MCPServerStdio

server = MCPServerStdio("python", args=["mcp_server.py"])
agent = Agent("ollama:qwen2.5", mcp_servers=[server], system_prompt=SYSTEM_PROMPT)

async def handle_query(q: str) -> str:
    async with agent.run_mcp_servers():
        result = await agent.run(q)
        return result.output
```

Same flow, fewer lines — choose this once the app grows.

---

## 7. Recap (orchestrator in 5 lines)

1. Connect to the MCP server, get the tool list.
2. Send system prompt + user query + tools to the local LLM.
3. If the LLM asks for a tool → run it via MCP → feed the JSON back → loop.
4. When the LLM replies with plain text → that's the answer.
5. Return the answer string to the UI.
