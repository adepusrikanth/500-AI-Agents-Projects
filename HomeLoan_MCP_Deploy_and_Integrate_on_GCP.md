# Deploying Your MCP Tools on GCP and Integrating with Gemini / ChatGPT / Claude

> **You already know how to:** build MCP tools, write an API per tool, and run a **FastMCP** server with **uvicorn**.
> **This guide answers:** *Once the server is running, how do I deploy it on GCP and actually connect it to Gemini, ChatGPT, and Claude — given that (a) I can't enable MCP servers in VS Code, and (b) the ChatGPT/Gemini consumer apps are blocked on company laptops?*

---

## 1. The One Idea That Resolves Your Constraints

There are **two ways** an LLM can use an MCP server:

| Pattern | Who is the MCP client? | Where does it run? | Needs the consumer app? |
|---------|------------------------|--------------------|--------------------------|
| **A. Consumer-app connector** | Claude Desktop / ChatGPT app / Gemini app | On the **user's laptop** | ✅ Yes — **blocked for you** |
| **B. Your own orchestrator service** ✅ | **Your code** (a small "agent" service) | On **GCP (Cloud Run)** | ❌ **No** |

Because the consumer apps are blocked on company laptops, **Pattern A is off the table.** You go with **Pattern B**:

> You build a small **Orchestrator / Agent service** on GCP. *That service* is the MCP client. It calls the LLM through its **API/SDK** (server-to-server) and calls your **MCP server** for the tools. Company users just open an internal web page (or hit an internal API) — they never need the blocked consumer apps, and the model calls leave from **GCP**, not from the laptop.

```
   Company laptop (browser only)
            │  HTTPS (behind Identity-Aware Proxy)
            ▼
   ┌─────────────────────────────┐        API call (server→server)
   │  Orchestrator / Agent svc   │ ───────────────────────────────► LLM API
   │  (Cloud Run)                │   Gemini = Vertex AI (in-GCP)
   │  - is the MCP CLIENT        │   ChatGPT = OpenAI API
   │  - calls the LLM API        │   Claude  = Anthropic API
   └──────────────┬──────────────┘
                  │ MCP (streamable HTTP + auth)
                  ▼
   ┌─────────────────────────────┐
   │  Your FastMCP server        │  (Cloud Run)  ← the tools you already built
   │  calculate_emi, eligibility │
   │  get_rates, search_kb ...   │
   └─────────────────────────────┘
```

**Key takeaways:**
- The MCP server and the orchestrator are **both Cloud Run services**. They talk over the network, not through VS Code.
- **Gemini via Vertex AI is fully inside GCP** — no external access needed, so the laptop block is irrelevant.
- **ChatGPT (OpenAI) and Claude (Anthropic)** are reached by a **server-side API call from Cloud Run** with egress allowed. The laptop block does not matter because the laptop never calls them — your GCP service does.
- **VS Code is never used.** You test with the MCP Inspector, `curl`, and a tiny Python client.

---

## 2. Deploying the FastMCP Server on GCP (Cloud Run)

### 2.1 Make the server speak remote MCP (streamable HTTP)
For cloud clients to reach it, run the **streamable-HTTP** transport (not stdio). With the MCP Python SDK / FastMCP:

```python
# server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("homeloan-mcp")

@mcp.tool()
def calculate_emi(principal: float, annual_rate: float, months: int) -> dict:
    r = annual_rate / 12 / 100
    emi = principal * r * (1 + r) ** months / ((1 + r) ** months - 1) if r else principal / months
    return {"emi": round(emi, 2), "total_payment": round(emi * months, 2)}

# ... your other tools ...

# Expose an ASGI app so uvicorn/Cloud Run can serve it
app = mcp.streamable_http_app()
```

Run locally exactly as you do today:
```bash
uvicorn server:app --host 0.0.0.0 --port 8080
```

> Cloud Run injects the port via `$PORT` (default 8080). Bind to `0.0.0.0:$PORT`.

### 2.2 Containerize

```dockerfile
# Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV PORT=8080
CMD ["sh", "-c", "uvicorn server:app --host 0.0.0.0 --port ${PORT}"]
```

`requirements.txt` (example): `mcp`, `uvicorn`, plus your tool deps.

### 2.3 Build & deploy to Cloud Run

```bash
# one-time: enable services
gcloud services enable run.googleapis.com artifactregistry.googleapis.com \
    cloudbuild.googleapis.com aiplatform.googleapis.com secretmanager.googleapis.com

# create an Artifact Registry repo (one-time)
gcloud artifacts repositories create mcp --repository-format=docker --location=asia-south1

# build the image with Cloud Build and push
gcloud builds submit --tag asia-south1-docker.pkg.dev/$PROJECT/mcp/homeloan-mcp:1.0

# deploy to Cloud Run (start private: no unauthenticated access)
gcloud run deploy homeloan-mcp \
  --image asia-south1-docker.pkg.dev/$PROJECT/mcp/homeloan-mcp:1.0 \
  --region asia-south1 \
  --no-allow-unauthenticated \
  --min-instances 0 --max-instances 10 \
  --set-secrets "MCP_API_KEY=mcp-api-key:latest"
```

You now have an HTTPS URL like `https://homeloan-mcp-xxxx.a.run.app/mcp`.

### 2.4 Secure the MCP endpoint
- **Service-to-service (recommended):** keep `--no-allow-unauthenticated` and let *only* the orchestrator's service account call it (Cloud Run IAM invoker + ID token). Nothing on the public internet can hit your tools.
- **Or token auth:** require a bearer/API key (from **Secret Manager**) in an `Authorization` header, validated by middleware in `server.py`.
- Put **Cloud Armor** + HTTPS LB in front if you expose it more broadly.

---

## 3. Deploying the Orchestrator (the MCP client + LLM caller)

This is the new piece that makes integration work without the consumer apps. It's a small FastAPI/Flask service on Cloud Run that:
1. Receives a user question (from your internal web page or API).
2. Connects to the MCP server (as an MCP client) to discover/call tools.
3. Calls the chosen LLM via its API, letting the model decide which tool to call.
4. Returns the grounded answer.

Deploy it the same way as Section 2 (Dockerfile → Cloud Build → Cloud Run). Give its **service account** the `roles/run.invoker` permission on the MCP server, and (for Gemini) `roles/aiplatform.user`.

### 3.1 Connecting the orchestrator to the MCP server (the MCP client side)

```python
# mcp_client.py  (illustrative)
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

MCP_URL = "https://homeloan-mcp-xxxx.a.run.app/mcp"

async def open_session(headers):
    # `headers` carries the Cloud Run ID token or API key
    async with streamablehttp_client(MCP_URL, headers=headers) as (r, w, _):
        async with ClientSession(r, w) as session:
            await session.initialize()
            tools = await session.list_tools()   # discover your tools
            return session, tools
```

To get the ID token for a private Cloud Run target, the orchestrator uses Google auth libraries (metadata server) — no static key needed.

---

## 4. Integrating Each LLM (server-side, no consumer app)

> All three calls happen **from the orchestrator on Cloud Run**, so company-laptop blocks don't apply. You only need the relevant API access/egress from GCP.

### 4.1 Gemini — via **Vertex AI** (best fit: stays inside GCP)
Gemini through Vertex AI runs in your GCP project, authenticated by IAM — **nothing leaves GCP, nothing is blocked.** The Google Gen AI SDK can take an MCP session directly and do automatic tool-calling:

```python
# gemini_orchestrator.py (illustrative)
from google import genai
from google.genai import types

client = genai.Client(vertexai=True, project=PROJECT, location="asia-south1")

async def ask_gemini(question, mcp_session):
    resp = await client.aio.models.generate_content(
        model="gemini-2.5-flash",
        contents=question,
        config=types.GenerateContentConfig(
            temperature=0,
            tools=[mcp_session],   # SDK auto-discovers + calls your MCP tools
        ),
    )
    return resp.text
```

If you prefer manual control: list MCP tools → convert them to Vertex **function declarations** → on each `functionCall` from Gemini, invoke `session.call_tool(...)` → feed the result back. Both work.

### 4.2 ChatGPT (OpenAI) — via the **OpenAI API / Agents SDK**
The laptop can't open chat.openai.com, but your **Cloud Run service can call the OpenAI API** (allow egress; store the key in Secret Manager). The Agents SDK speaks MCP directly:

```python
# openai_orchestrator.py (illustrative)
from agents import Agent, Runner
from agents.mcp import MCPServerStreamableHttp

async def ask_chatgpt(question):
    async with MCPServerStreamableHttp(
        params={"url": "https://homeloan-mcp-xxxx.a.run.app/mcp",
                "headers": {"Authorization": f"Bearer {MCP_API_KEY}"}}
    ) as mcp_server:
        agent = Agent(
            name="HomeLoanAssistant",
            instructions="Answer only from the home-loan tools/knowledge. "
                         "If unknown, tell the customer to contact the branch.",
            mcp_servers=[mcp_server],
        )
        result = await Runner.run(agent, question)
        return result.final_output
```

> If direct OpenAI egress isn't allowed by InfoSec, route it through **Vertex AI Model Garden** or an approved **AI Gateway** — same code pattern, different base URL/credentials.

### 4.3 Claude (Anthropic) — via the **Anthropic API**
Same idea: server-side API call from Cloud Run. Anthropic's Messages API supports a **remote MCP connector**, so you can point Claude straight at your Cloud Run MCP URL:

```python
# claude_orchestrator.py (illustrative)
import anthropic
client = anthropic.Anthropic(api_key=ANTHROPIC_API_KEY)  # key from Secret Manager

def ask_claude(question):
    resp = client.beta.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        messages=[{"role": "user", "content": question}],
        mcp_servers=[{
            "type": "url",
            "url": "https://homeloan-mcp-xxxx.a.run.app/mcp",
            "name": "homeloan",
            "authorization_token": MCP_API_KEY,
        }],
        betas=["mcp-client-2025-04-04"],
    )
    return resp.content
```

> Alternatively, the orchestrator can be the MCP client (like the OpenAI example) and pass tools to Claude manually. Use whichever your InfoSec approves.

---

## 5. Giving Company Users Access (without the blocked apps)

Build a thin **internal chat UI** (or just an internal REST endpoint) in front of the orchestrator and protect it with **Identity-Aware Proxy (IAP)**:

```bash
# expose orchestrator only to authenticated company users via IAP
gcloud run deploy homeloan-assistant \
  --image .../homeloan-assistant:1.0 --region asia-south1 \
  --no-allow-unauthenticated
# then enable IAP on the backend / load balancer and grant access to your AD group
```

Now a banker opens an **internal URL in their browser**, logs in with their company Google identity (IAP), types a question, and gets a grounded answer — the LLM and MCP calls all happen inside GCP. No ChatGPT/Gemini app needed on the laptop.

---

## 6. Testing Without VS Code

| Method | Command / how | Use |
|--------|----------------|-----|
| **MCP Inspector** | `npx @modelcontextprotocol/inspector` then point it at your URL + auth header | Interactively list & call every tool, inspect schemas |
| **curl** | `curl -H "Authorization: Bearer $KEY" https://.../mcp` (MCP handshake) | Smoke-check reachability/auth |
| **Python MCP client** | the `mcp_client.py` snippet above in a script | Automated CI test: list tools, call each, assert output |
| **Orchestrator endpoint** | `curl -X POST https://assistant/.../chat -d '{"q":"EMI for 30L at 8.5% 240m"}'` | End-to-end test through the real LLM path |

Wire methods 3–4 into **Cloud Build** so every push runs them — no IDE involved.

---

## 7. Deploy Checklist (end to end)

1. ☐ `server.py` runs streamable-HTTP via uvicorn locally; tools verified in MCP Inspector.
2. ☐ Dockerfile builds; image pushed to Artifact Registry.
3. ☐ MCP server deployed to Cloud Run, **private** (`--no-allow-unauthenticated`).
4. ☐ Secrets (MCP key, OpenAI/Anthropic keys) in **Secret Manager**.
5. ☐ Orchestrator service deployed; its service account has `run.invoker` on the MCP server + `aiplatform.user` for Vertex AI.
6. ☐ Gemini path works via Vertex AI (in-GCP).
7. ☐ OpenAI/Anthropic egress approved by InfoSec **or** routed via an approved gateway; keys stored; calls succeed from Cloud Run.
8. ☐ Internal UI/endpoint protected by **IAP**; company AD group granted access.
9. ☐ CI (Cloud Build) runs MCP client tests + calculator unit tests on every push.
10. ☐ Logging, audit of tool calls, and alerts enabled.

---

## 8. FAQ for Your Two Blockers

**Q: I can't enable MCP servers in VS Code — is that a problem?**
No. VS Code is only one possible MCP *client*. Your client is the Cloud Run orchestrator. Test with the MCP Inspector, `curl`, or a Python client.

**Q: ChatGPT and Gemini apps are blocked on company laptops.**
That blocks **Pattern A** (consumer-app connectors) only. In **Pattern B**, the laptop never talks to those services — your **GCP orchestrator** does, via APIs. Gemini via **Vertex AI** is entirely inside GCP. For OpenAI/Anthropic, your Cloud Run service makes the API call (with InfoSec-approved egress or an approved gateway). Users only open an internal IAP-protected page.

**Q: Do I need all three LLMs?**
No. Given your environment, **Gemini via Vertex AI is the path of least resistance** (in-GCP, no external egress, no extra approvals). Add OpenAI/Anthropic later if/when egress is approved — the orchestrator pattern stays the same; only the model-call function changes.
