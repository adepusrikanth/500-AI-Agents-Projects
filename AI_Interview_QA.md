# AI Engineering Interview Questions & Answers

Comprehensive interview preparation covering **AI Agents & Orchestration**, **Production AI Systems**, and **Advanced AI Engineering**.

---

## Table of Contents

- [Section 3: AI Agents & Orchestration](#section-3-ai-agents--orchestration)
  - [LangGraph](#langgraph)
  - [AI Agents](#ai-agents)
  - [Orchestration](#orchestration)
  - [Memory Systems](#memory-systems)
  - [Multi-Agent Workflows](#multi-agent-workflows)
- [Section 4: Production AI Systems](#section-4-production-ai-systems)
  - [Production AI Deployment](#production-ai-deployment)
  - [Observability (Langfuse, etc.)](#observability-langfuse-etc)
  - [Inference Pipelines](#inference-pipelines)
  - [Scalable AI Infrastructure](#scalable-ai-infrastructure)
- [Section 5: Advanced AI Engineering](#section-5-advanced-ai-engineering)
  - [Fine-Tuning LLMs](#fine-tuning-llms)
  - [Custom Model Training](#custom-model-training)
  - [Advanced RAG (Hybrid Search)](#advanced-rag-hybrid-search)
  - [LLMOps / MLOps](#llmops--mlops)
  - [Scalable, Reliable AI Architecture](#scalable-reliable-ai-architecture)

---

## Section 3: AI Agents & Orchestration

### LangGraph

**Q1: What is LangGraph, and how does it differ from LangChain?**

**A:** LangGraph is a library built on top of LangChain for creating stateful, multi-step agent workflows as directed graphs. While LangChain provides chains (linear sequences of calls) and basic agents, LangGraph models workflows as nodes (functions) and edges (transitions), giving fine-grained control over execution flow, branching, looping, and state management. Key differences:

- **LangChain** uses chains and sequential pipelines; LangGraph uses a graph-based execution model with cycles and conditional branching.
- **LangGraph** maintains a persistent state object that passes through nodes, enabling complex multi-turn interactions.
- **LangGraph** natively supports human-in-the-loop patterns, checkpointing, and resumable workflows.
- LangGraph gives explicit control over the agent's decision-making loop rather than relying on the LLM to decide when to stop.

---

**Q2: Explain the core concepts of LangGraph: nodes, edges, and state.**

**A:**

- **State:** A shared data structure (typically a TypedDict or Pydantic model) that persists across the entire graph execution. Every node reads from and writes to this state. State can include messages, intermediate results, tool outputs, and control flags.
- **Nodes:** Functions that perform a unit of work — calling an LLM, executing a tool, transforming data, or making a decision. Each node receives the current state, performs its logic, and returns a partial state update that gets merged back.
- **Edges:** Define transitions between nodes. There are two types:
  - *Normal edges*: unconditional transitions from one node to another.
  - *Conditional edges*: use a routing function that inspects the current state to decide which node to transition to next, enabling branching and looping.

---

**Q3: How does LangGraph handle checkpointing and state persistence?**

**A:** LangGraph supports checkpointing through its `Checkpointer` interface. After each node execution, the full graph state is serialized and persisted. This enables:

- **Resumability:** A workflow can be paused and resumed from the last checkpoint (e.g., after a server restart or human review).
- **Human-in-the-loop:** The graph pauses at designated breakpoints, waits for human input, and resumes.
- **Time travel:** You can inspect or revert to any prior state in the execution history.
- **Fault tolerance:** If a node fails, the graph can be re-executed from the last successful checkpoint.

Built-in checkpointer implementations include `MemorySaver` (in-memory), `SqliteSaver`, and `PostgresSaver` for production use.

---

**Q4: How would you implement a ReAct agent loop in LangGraph?**

**A:** A ReAct (Reasoning + Acting) loop in LangGraph involves:

1. **Define the state** with a `messages` list (using `add_messages` reducer for appending).
2. **Create an "agent" node** that calls the LLM with the current messages and available tools. The LLM either returns a final answer or a tool call.
3. **Create a "tools" node** that executes the requested tool and appends the result back to the messages.
4. **Add a conditional edge** from the agent node: if the LLM response contains tool calls, route to the tools node; otherwise, route to `END`.
5. **Add a normal edge** from the tools node back to the agent node, creating the loop.

```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode

graph = StateGraph(AgentState)
graph.add_node("agent", call_model)
graph.add_node("tools", ToolNode(tools))
graph.set_entry_point("agent")
graph.add_conditional_edges("agent", should_continue, {"continue": "tools", "end": END})
graph.add_edge("tools", "agent")
app = graph.compile()
```

---

**Q5: What are the advantages of using LangGraph's `create_react_agent` prebuilt function versus building a custom graph?**

**A:** `create_react_agent` provides a quick, production-ready ReAct agent with sensible defaults — it handles the agent loop, tool execution, and termination logic automatically. Advantages include rapid prototyping and less boilerplate.

However, building a custom graph is preferred when you need:
- Custom routing logic (e.g., routing to different tools based on intent classification).
- Multiple sub-graphs or hierarchical agent teams.
- Complex state beyond simple message lists.
- Specific error-handling or retry strategies per node.
- Human-in-the-loop at precise points in the workflow.
- Parallel node execution or map-reduce patterns.

---

### AI Agents

**Q6: What is an AI agent, and how does it differ from a simple LLM call?**

**A:** An AI agent is an autonomous system that uses an LLM as its reasoning engine to plan, make decisions, and take actions to accomplish goals. Unlike a single LLM call (which is stateless, one-shot input/output), an agent:

- **Perceives** its environment (user input, tool outputs, external data).
- **Reasons** about what action to take next.
- **Acts** by calling tools, APIs, or code execution environments.
- **Observes** the results and iterates until the goal is achieved.
- Maintains **state and memory** across interaction steps.

The key distinction is the *action loop* — an agent decides autonomously when and how to use tools, whereas a basic LLM call just generates text.

---

**Q7: Describe the ReAct pattern for AI agents.**

**A:** ReAct (Reasoning + Acting) interleaves chain-of-thought reasoning with tool actions:

1. **Thought:** The LLM reasons about the current situation and what needs to be done.
2. **Action:** Based on the thought, it selects and invokes a tool with specific inputs.
3. **Observation:** The tool's output is returned and appended to the context.
4. **Repeat:** The LLM reasons again given the new observation, deciding whether to take another action or produce a final answer.

This cycle continues until the agent determines it has enough information to answer. ReAct improves reliability over pure chain-of-thought by grounding reasoning in real tool outputs, and improves over pure action-taking by adding explicit reasoning traces.

---

**Q8: What are tool-calling agents, and how does function calling work with LLMs?**

**A:** Tool-calling agents leverage the LLM's ability to generate structured tool invocations rather than free-form text. The workflow:

1. **Tool definitions** are provided to the LLM as part of the prompt or API schema (name, description, parameters with types).
2. The LLM analyzes the user query and decides whether a tool is needed.
3. If so, it outputs a structured JSON object specifying the tool name and arguments.
4. The orchestration layer intercepts this, executes the actual function, and feeds the result back.
5. The LLM incorporates the result to generate the next step or final response.

Modern APIs (OpenAI, Anthropic, Google) support native function/tool calling, which is more reliable than parsing tool calls from free-form text. Agents can call multiple tools in parallel when the API supports it.

---

**Q9: What are the common failure modes of AI agents, and how do you mitigate them?**

**A:**

| Failure Mode | Mitigation |
|---|---|
| **Infinite loops** | Set max iteration limits; add loop-detection logic |
| **Tool misuse** | Write clear, unambiguous tool descriptions; validate inputs before execution |
| **Hallucinated tool calls** | Use structured function calling APIs; schema validation |
| **Context window overflow** | Summarize long histories; use sliding windows; implement memory compression |
| **Error cascading** | Add error handling per tool; return structured error messages the LLM can reason about |
| **Over-planning** | Limit reasoning steps; use plan-then-execute patterns with fixed plan sizes |
| **Security risks** | Sandbox code execution; validate URLs; restrict filesystem access; implement guardrails |

---

**Q10: Explain the difference between proactive and reactive AI agents.**

**A:**

- **Reactive agents** respond to user input or external triggers. They wait for a stimulus, process it, and return a result. Most chatbot-style agents are reactive.
- **Proactive agents** can initiate actions autonomously based on goals, schedules, or environmental changes. Examples include agents that monitor data streams and trigger alerts, or agents that periodically check for updates and act on them.

In practice, many production agents combine both: they react to user requests but proactively execute multi-step plans, retrieve information without being asked, and suggest next actions.

---

### Orchestration

**Q11: What is agent orchestration, and why is it needed?**

**A:** Agent orchestration is the coordination layer that manages how one or more AI agents execute tasks. It handles:

- **Execution flow:** Sequencing steps, managing parallel execution, handling branching logic.
- **State management:** Maintaining context across steps and agent calls.
- **Tool routing:** Directing tool calls to the correct implementations.
- **Error handling:** Retrying failed steps, implementing fallbacks, managing timeouts.
- **Resource management:** Rate limiting LLM calls, managing API quotas, controlling costs.

Orchestration is needed because real-world AI tasks involve multiple steps, tools, and decision points that cannot be handled by a single LLM call. Frameworks like LangGraph, CrewAI, and AutoGen provide orchestration primitives.

---

**Q12: Compare orchestration approaches: sequential chains, DAGs, and cyclic graphs.**

**A:**

| Approach | Structure | Use Case | Example |
|---|---|---|---|
| **Sequential chains** | Linear A → B → C | Simple pipelines with fixed steps | Summarize → Translate → Format |
| **DAGs (Directed Acyclic Graphs)** | Branching/merging, no cycles | Parallel tasks with dependencies | Research + Analyze → Combine → Report |
| **Cyclic graphs** | Loops allowed | Iterative refinement, agent loops | ReAct loop, self-correction, retry logic |

- Sequential chains are simplest but inflexible.
- DAGs allow parallelism and are great for data pipelines but cannot iterate.
- Cyclic graphs (as in LangGraph) are the most powerful, supporting agent loops, self-correction, and dynamic control flow — but require careful termination conditions to avoid infinite loops.

---

**Q13: How do you handle error recovery and retries in an orchestration framework?**

**A:** Strategies include:

1. **Node-level retries:** Wrap individual nodes with retry logic (exponential backoff) for transient failures (API timeouts, rate limits).
2. **Fallback nodes:** If a primary tool/model fails, route to an alternative (e.g., fall back from GPT-4 to GPT-3.5).
3. **Checkpoint-and-resume:** Persist state at each step so a failed workflow can resume from the last successful node.
4. **Error-as-state:** Instead of crashing, capture errors in the state and let the LLM reason about recovery (e.g., "The API returned a 404. Try a different search query.").
5. **Circuit breakers:** After N consecutive failures, stop retrying and escalate to human review.
6. **Timeout guards:** Set maximum execution time per node and globally to prevent runaway processes.

---

**Q14: What is the "supervisor" pattern in multi-agent orchestration?**

**A:** The supervisor pattern uses a dedicated "manager" agent that delegates tasks to specialized worker agents:

1. The **supervisor** receives a task, analyzes it, and determines which worker agent(s) should handle it.
2. It dispatches sub-tasks to the appropriate workers (e.g., a researcher agent, a coder agent, a reviewer agent).
3. Workers execute their tasks and report results back to the supervisor.
4. The supervisor evaluates the results, decides if more work is needed, and either delegates further or produces the final output.

This pattern is implemented in LangGraph's multi-agent tutorials and in frameworks like CrewAI. It provides clear separation of concerns, specialization, and centralized control flow.

---

### Memory Systems

**Q15: What are the different types of memory in AI agent systems?**

**A:**

- **Short-term memory (Working memory):** The current conversation context or messages within a single session. Stored in the LLM's context window. Limited by token capacity.
- **Long-term memory:** Persistent storage of facts, user preferences, and past interactions across sessions. Typically implemented via vector databases, key-value stores, or knowledge graphs.
- **Episodic memory:** Records of specific past interactions or events that can be retrieved contextually. Like "remembering" a previous conversation about a specific topic.
- **Semantic memory:** General knowledge and facts about the world, stored as embeddings in a vector store for retrieval.
- **Procedural memory:** Learned procedures and skills, often encoded as prompt templates, few-shot examples, or fine-tuned model weights.

---

**Q16: How do you implement long-term memory for an AI agent?**

**A:** Common approaches:

1. **Vector store memory:** Store conversation summaries or key facts as embeddings in a vector database (Pinecone, Weaviate, Chroma). On each interaction, retrieve relevant past context via semantic search and inject it into the prompt.
2. **Structured memory:** Store user preferences and facts in a structured database (SQL, Redis). Query specific fields when needed.
3. **Knowledge graphs:** Represent entities and relationships extracted from conversations. Use graph queries to retrieve connected information.
4. **Summary memory:** Periodically summarize conversations using an LLM and store the summaries. Older details are compressed into summaries while recent ones remain in full.
5. **Hybrid approaches:** Combine vector search for semantic recall with structured storage for specific facts (e.g., user name, preferences).

Key considerations: memory relevance scoring, forgetting mechanisms (TTL, importance-based pruning), and privacy/data retention policies.

---

**Q17: Explain the concept of a memory buffer with summarization. How does it prevent context window overflow?**

**A:** A memory buffer with summarization maintains a sliding window of recent messages at full fidelity while compressing older messages into summaries:

1. Keep the last N messages (or K tokens) verbatim in the buffer.
2. When the buffer exceeds capacity, take the oldest messages and generate a summary using the LLM.
3. Replace the old messages with the condensed summary.
4. The prompt is composed of: system prompt + summary of older context + recent full messages.

This prevents context window overflow by ensuring the total token count stays within limits while preserving important historical context in compressed form. The trade-off is potential loss of specific details from older interactions.

---

**Q18: How would you implement memory with a vector database for an agent?**

**A:**

1. **Ingestion:** After each conversation turn (or at session end), extract key information, embed it using an embedding model (e.g., `text-embedding-3-small`), and upsert it into the vector database with metadata (timestamp, user_id, topic).
2. **Retrieval:** Before the agent processes a new query, generate an embedding for the query, perform a similarity search against the vector store, and retrieve the top-K most relevant memory entries.
3. **Injection:** Format the retrieved memories and prepend them to the system prompt or conversation context: `"Relevant context from previous interactions: ..."`
4. **Decay/Pruning:** Implement time-based decay (reduce relevance scores for old memories) or importance-based pruning (remove low-access-count memories periodically).

```python
# Pseudocode
memories = vector_store.similarity_search(query, k=5, filter={"user_id": user_id})
context = "\n".join([m.page_content for m in memories])
response = llm.invoke(f"Context: {context}\n\nUser: {query}")
```

---

### Multi-Agent Workflows

**Q19: What are multi-agent workflows, and when should you use them over a single agent?**

**A:** Multi-agent workflows involve multiple specialized agents collaborating to complete complex tasks. Use them when:

- The task requires **diverse expertise** that cannot be captured in a single prompt (e.g., research + code + review).
- You need **separation of concerns** — each agent has focused instructions and tools.
- The workflow benefits from **parallel execution** — multiple agents work on sub-tasks simultaneously.
- You want **checks and balances** — one agent generates, another critiques/validates.
- The task is too complex for a single agent's context window or reasoning capability.

Avoid multi-agent systems for simple tasks — the added complexity of inter-agent communication and coordination introduces latency, cost, and potential failure points.

---

**Q20: Describe the common multi-agent architectures.**

**A:**

1. **Supervisor/Manager:** A central agent delegates to workers and aggregates results. Provides centralized control but creates a bottleneck.
2. **Peer-to-peer (Swarm):** Agents communicate directly with each other via handoffs. More flexible but harder to debug. Used in OpenAI's Swarm pattern.
3. **Hierarchical:** Multiple levels of supervisors managing teams of workers. Suitable for very complex tasks with natural sub-task decomposition.
4. **Pipeline/Sequential:** Agents execute in a fixed sequence, each passing its output to the next. Simple but inflexible (e.g., Writer → Editor → Fact-checker).
5. **Debate/Adversarial:** Multiple agents argue different positions, and a judge agent synthesizes the best answer. Improves reasoning quality.
6. **Map-Reduce:** A coordinator fans out sub-tasks to multiple agents (map), then another agent aggregates results (reduce). Great for parallelizable tasks.

---

**Q21: How does CrewAI implement multi-agent workflows?**

**A:** CrewAI organizes multi-agent workflows around three core concepts:

- **Agents:** Autonomous units with a role, goal, backstory, and assigned tools. Each agent has a specific persona and expertise.
- **Tasks:** Units of work assigned to agents, with descriptions, expected outputs, and optional context from other tasks.
- **Crew:** The orchestration layer that assembles agents and tasks, defining the process type:
  - *Sequential:* Tasks execute in order.
  - *Hierarchical:* A manager agent delegates tasks to workers.

CrewAI handles agent communication, task delegation, and result aggregation. It supports tool integration, memory, and callbacks for monitoring.

```python
from crewai import Agent, Task, Crew, Process

researcher = Agent(role="Researcher", goal="Find accurate info", tools=[search_tool])
writer = Agent(role="Writer", goal="Write compelling content")

research_task = Task(description="Research topic X", agent=researcher)
write_task = Task(description="Write article on X", agent=writer, context=[research_task])

crew = Crew(agents=[researcher, writer], tasks=[research_task, write_task], process=Process.sequential)
result = crew.kickoff()
```

---

**Q22: How do agents communicate in a multi-agent system, and what are the challenges?**

**A:** Communication methods:

- **Shared state:** All agents read/write to a common state object (LangGraph approach). Simple but can cause conflicts.
- **Message passing:** Agents send structured messages to each other (AutoGen approach with chat histories).
- **Tool-based handoff:** One agent calls another as a tool, passing input and receiving output.
- **Blackboard pattern:** Agents post intermediate results to a shared "blackboard" and react to updates.

Challenges:
- **Context drift:** As messages pass between agents, the original intent can get distorted.
- **Information loss:** Summarizing inter-agent messages can lose critical details.
- **Coordination overhead:** More agents mean more communication, increasing latency and cost.
- **Deadlocks:** Circular dependencies between agents waiting on each other.
- **Inconsistent outputs:** Different agents may produce contradictory results requiring resolution logic.

---

---

## Section 4: Production AI Systems

### Production AI Deployment

**Q23: What are the key considerations when deploying an AI agent to production?**

**A:**

1. **Reliability:** Implement retries, fallbacks, circuit breakers, and graceful degradation.
2. **Latency:** Optimize prompt lengths, use streaming responses, cache frequent queries, choose appropriate model sizes.
3. **Cost management:** Track token usage per request, implement rate limiting, use cheaper models for simpler tasks (model routing).
4. **Security:** Sanitize inputs to prevent prompt injection, sandbox code execution, restrict tool access, implement authentication and authorization.
5. **Scalability:** Use async processing, queue-based architectures, and horizontal scaling for concurrent requests.
6. **Monitoring:** Log all LLM calls, tool invocations, and agent decisions. Track latency, error rates, and costs.
7. **Testing:** Unit test individual tools, integration test agent workflows, evaluate output quality with LLM-as-judge or human review.
8. **Versioning:** Version prompts, tool definitions, and model configurations. Enable rollbacks.

---

**Q24: How do you handle prompt injection attacks in production AI systems?**

**A:** Defense strategies:

1. **Input sanitization:** Strip or escape potentially malicious instructions from user input.
2. **Prompt structure:** Use clear delimiters (e.g., XML tags) to separate system instructions from user input. Instruct the model to ignore instructions within user content.
3. **Output validation:** Parse and validate LLM outputs before executing any actions (especially tool calls, code execution, database queries).
4. **Least privilege:** Give agents only the minimum tools and permissions needed. Never expose admin-level operations.
5. **Guardrails:** Use secondary LLM calls or rule-based systems to classify inputs as safe/unsafe before processing.
6. **Sandboxing:** Execute any generated code in isolated environments with resource limits.
7. **Human-in-the-loop:** Require human approval for high-stakes actions (e.g., sending emails, modifying data).
8. **Monitoring:** Log all interactions and set up alerts for anomalous patterns.

---

**Q25: Explain the concept of model routing and when to use it.**

**A:** Model routing directs requests to different LLM models based on task complexity, cost constraints, or latency requirements:

- **Simple queries** (FAQ, classification) → smaller/cheaper model (e.g., GPT-3.5, Haiku).
- **Complex reasoning** (multi-step analysis, code generation) → larger model (e.g., GPT-4, Opus).
- **Specialized tasks** (code, math) → domain-specific models.

Implementation approaches:
1. **Classifier-based:** A lightweight classifier analyzes the query and routes to the appropriate model.
2. **Cascading:** Start with a cheap model; if confidence is low, escalate to a more capable one.
3. **Rule-based:** Route by task type, user tier, or request metadata.

Benefits: 50-80% cost reduction while maintaining quality on hard queries. Reduced average latency since most requests go to faster models.

---

**Q26: How do you implement rate limiting and cost controls for LLM-based systems?**

**A:**

1. **Token budgets:** Set per-user and per-request token limits. Track cumulative usage and enforce daily/monthly caps.
2. **Request rate limiting:** Use token bucket or sliding window algorithms to limit API calls per time window.
3. **Tiered access:** Different user tiers get different rate limits and model access.
4. **Cost tracking:** Log token counts (input + output) for every LLM call. Multiply by per-token pricing. Aggregate by user, feature, and model.
5. **Alerts:** Set up alerts when spend exceeds thresholds (e.g., 80% of daily budget).
6. **Caching:** Cache responses for identical or semantically similar queries to avoid redundant LLM calls.
7. **Prompt optimization:** Minimize prompt length through compression, removing redundant instructions, and using few-shot examples efficiently.
8. **Circuit breakers:** Automatically disable expensive features when cost thresholds are exceeded.

---

### Observability (Langfuse, etc.)

**Q27: What is observability in the context of LLM applications, and why is it critical?**

**A:** Observability for LLM applications means having comprehensive visibility into every aspect of your AI system's behavior:

- **Traces:** End-to-end tracking of a request through the entire agent pipeline (LLM calls, tool invocations, retrieval steps).
- **Spans:** Individual operations within a trace (a single LLM call, a vector search, a tool execution) with timing and metadata.
- **Metrics:** Latency, token usage, cost, error rates, cache hit rates, and quality scores.
- **Logs:** Detailed records of prompts, completions, tool inputs/outputs, and decision points.

It is critical because LLM applications are non-deterministic — the same input can produce different outputs. Without observability, you cannot debug issues, optimize performance, track costs, or evaluate quality systematically.

---

**Q28: What is Langfuse, and how does it integrate with LLM applications?**

**A:** Langfuse is an open-source LLM observability and analytics platform. It provides:

- **Tracing:** Automatic capture of LLM calls, tool usage, and retrieval operations as nested traces/spans.
- **Prompt management:** Version and manage prompts with A/B testing capabilities.
- **Evaluation:** Score traces using model-based evaluation, human annotation, or custom metrics.
- **Analytics:** Dashboards for cost, latency, token usage, and quality metrics over time.
- **Dataset management:** Create evaluation datasets from production traces.

Integration methods:
- **Native SDKs:** Python/JS decorators (`@observe`) that automatically capture function calls.
- **Framework integrations:** Direct integrations with LangChain, LlamaIndex, OpenAI SDK, and others.
- **API:** REST API for custom integrations.

```python
from langfuse.decorators import observe, langfuse_context

@observe()
def my_agent(query: str):
    result = llm.invoke(query)
    langfuse_context.update_current_observation(metadata={"model": "gpt-4"})
    return result
```

---

**Q29: Compare Langfuse with other LLM observability tools (LangSmith, Phoenix, Helicone).**

**A:**

| Feature | Langfuse | LangSmith | Phoenix (Arize) | Helicone |
|---|---|---|---|---|
| **Open source** | Yes (self-hostable) | No (SaaS) | Yes | Yes |
| **Tracing** | Full nested traces | Full nested traces | Full traces | Request-level |
| **Framework support** | LangChain, LlamaIndex, OpenAI, etc. | Primarily LangChain | LlamaIndex, OpenAI, LangChain | Any (proxy-based) |
| **Evaluation** | LLM-as-judge, human, custom | LLM-as-judge, human, datasets | Embedding drift, retrieval metrics | Basic |
| **Prompt management** | Yes | Yes (Hub) | No | No |
| **Cost tracking** | Yes | Yes | Yes | Yes (primary focus) |
| **Self-hosting** | Docker, managed cloud | No | Yes | Yes |
| **Best for** | General LLM observability | LangChain-heavy stacks | ML/embedding analysis | Simple cost/latency proxy |

---

**Q30: How do you set up evaluation pipelines using observability data?**

**A:**

1. **Collect production traces:** Capture representative samples of real user interactions including inputs, outputs, and intermediate steps.
2. **Define evaluation criteria:** Determine what "good" looks like — relevance, accuracy, helpfulness, safety, format compliance.
3. **Implement scoring:**
   - *LLM-as-judge:* Use a separate LLM to score outputs against criteria (e.g., "Rate this response's relevance from 1-5").
   - *Heuristic checks:* Automated rules (response length, format validation, keyword presence).
   - *Human annotation:* Sample traces for human review with annotation queues.
4. **Build datasets:** Export interesting/edge-case traces into evaluation datasets for regression testing.
5. **Automate:** Run evaluations on every deployment. Compare scores across prompt versions, model versions, and code changes.
6. **Alert on regressions:** Set thresholds and trigger alerts when quality metrics drop below acceptable levels.

---

### Inference Pipelines

**Q31: What is an inference pipeline, and what are its typical stages?**

**A:** An inference pipeline is the end-to-end processing chain that transforms a user request into a final response. Typical stages:

1. **Input preprocessing:** Sanitize input, classify intent, extract entities, apply guardrails.
2. **Context enrichment:** Retrieve relevant documents (RAG), load user profile/history, fetch real-time data.
3. **Prompt construction:** Assemble the system prompt, retrieved context, conversation history, and user query into the final prompt.
4. **LLM inference:** Send the prompt to the model and receive the response (with streaming if applicable).
5. **Output processing:** Parse structured outputs, validate format, extract tool calls, apply safety filters.
6. **Action execution:** Execute any tool calls, API requests, or database operations.
7. **Response formatting:** Format the final response for the client (markdown, JSON, streaming chunks).
8. **Logging and feedback:** Log the full trace, record metrics, and capture user feedback.

---

**Q32: How do you optimize inference latency in production?**

**A:**

1. **Streaming:** Stream tokens to the client as they are generated rather than waiting for the full response.
2. **Caching:** Implement semantic caching — if a similar query was recently answered, return the cached result. Use embedding similarity with a threshold.
3. **Prompt compression:** Remove unnecessary context, use shorter system prompts, compress retrieved documents.
4. **Model selection:** Use smaller, faster models for simple tasks (model routing).
5. **Parallel processing:** Execute independent pipeline stages concurrently (e.g., retrieval and user profile lookup in parallel).
6. **Batch processing:** For non-real-time workloads, batch multiple requests to improve throughput.
7. **Infrastructure:**
   - Use GPU-optimized serving (vLLM, TGI, TensorRT-LLM) for self-hosted models.
   - Place inference endpoints in the same region as your application.
   - Use connection pooling for API calls.
8. **KV cache optimization:** For self-hosted models, implement KV cache reuse across requests with shared prefixes.

---

**Q33: Explain the concept of semantic caching for LLM applications.**

**A:** Semantic caching stores LLM responses indexed by the semantic meaning of the query rather than exact string matching:

1. **On request:** Generate an embedding of the user's query.
2. **Cache lookup:** Search the cache for embeddings within a similarity threshold (e.g., cosine similarity > 0.95).
3. **Cache hit:** If a sufficiently similar query exists, return the cached response (with optional TTL check).
4. **Cache miss:** Call the LLM, store the response with the query embedding, and return the result.

Benefits:
- Dramatically reduces latency for repeated/similar queries.
- Reduces API costs by avoiding redundant LLM calls.
- Improves consistency — similar questions get consistent answers.

Challenges:
- Threshold tuning — too low gives incorrect cache hits; too high gives few hits.
- Cache invalidation when underlying data changes.
- Not suitable for queries that require real-time data or personalized responses.

Tools: GPTCache, Redis with vector search, custom implementations with any vector database.

---

**Q34: How do you implement A/B testing for LLM-based features?**

**A:**

1. **Traffic splitting:** Assign users to experiment groups (control vs. variant) based on user ID hashing for consistent assignment.
2. **Variants:** Different prompts, models, temperatures, retrieval strategies, or system architectures.
3. **Metrics:** Define primary metrics (task completion rate, user satisfaction) and guardrail metrics (latency, cost, safety violations).
4. **Logging:** Log the experiment variant with every trace for proper attribution.
5. **Evaluation:** Use both automated metrics (LLM-as-judge scores, BLEU/ROUGE for generation tasks) and user-behavioral metrics (click-through rate, thumbs up/down, session length).
6. **Statistical analysis:** Use appropriate tests — given the high variance of LLM outputs, you typically need larger sample sizes than traditional A/B tests.
7. **Gradual rollout:** Start with a small percentage (5-10%) on the variant, monitor for regressions, and gradually increase.

---

### Scalable AI Infrastructure

**Q35: How do you architect an AI system for horizontal scalability?**

**A:**

1. **Stateless application layer:** Design your inference service to be stateless — all state lives in external stores (Redis, databases, vector stores). This allows adding more instances without session affinity.
2. **Message queues:** Use queues (RabbitMQ, Kafka, SQS) to decouple request ingestion from processing. This enables async processing and load leveling.
3. **Auto-scaling:** Scale inference workers based on queue depth, CPU/GPU utilization, or request latency. Use Kubernetes HPA or cloud auto-scaling groups.
4. **Load balancing:** Distribute requests across instances with load balancers. Use health checks to route away from unhealthy instances.
5. **Caching layers:** Add Redis/Memcached for prompt templates, embedding caches, and semantic response caches.
6. **Database scaling:** Use read replicas for vector databases under heavy retrieval load. Shard by user or tenant for multi-tenant systems.
7. **CDN/Edge:** Cache static assets and pre-computed responses at the edge for reduced latency.

---

**Q36: Compare self-hosted vs. API-based LLM inference for production systems.**

**A:**

| Factor | Self-Hosted (vLLM, TGI, Ollama) | API-Based (OpenAI, Anthropic) |
|---|---|---|
| **Cost at scale** | Lower marginal cost at high volume | Pay-per-token, higher at scale |
| **Latency control** | Full control, co-located | Depends on provider, network hops |
| **Data privacy** | Data stays in your infra | Data sent to third party |
| **Model selection** | Open-source models only | Access to frontier models |
| **Ops burden** | High (GPU management, updates, monitoring) | Minimal |
| **Scaling** | Manual GPU provisioning | Automatic, near-infinite |
| **Uptime** | Your responsibility | Provider SLA (usually 99.9%) |
| **Customization** | Full (fine-tuning, quantization) | Limited (fine-tuning APIs only) |

**Recommendation:** Use APIs for prototyping, low-to-medium volume, and when frontier model quality is critical. Self-host for high volume, strict data privacy requirements, or when you need specialized/fine-tuned open-source models.

---

**Q37: What is vLLM, and why is it used for production LLM serving?**

**A:** vLLM is a high-throughput, memory-efficient LLM serving engine. Key features:

- **PagedAttention:** A novel attention algorithm that manages the KV cache in non-contiguous memory blocks (like virtual memory paging), reducing memory waste by up to 90% compared to naive implementations.
- **Continuous batching:** Dynamically batches incoming requests and processes them together, maximizing GPU utilization.
- **Tensor parallelism:** Distributes model weights across multiple GPUs for serving models larger than a single GPU's memory.
- **Quantization support:** Serves quantized models (GPTQ, AWQ, FP8) for reduced memory footprint and faster inference.
- **OpenAI-compatible API:** Drop-in replacement for OpenAI's API, making migration seamless.
- **Speculative decoding:** Uses a smaller draft model to generate candidate tokens, verified by the main model, for faster generation.

vLLM typically achieves 2-4x higher throughput than HuggingFace Transformers for serving.

---

**Q38: How do you handle GPU resource management for AI workloads?**

**A:**

1. **Right-sizing:** Match GPU type to workload — A100/H100 for large model serving, T4/L4 for smaller models and inference, A100 for training.
2. **Multi-model serving:** Run multiple smaller models on a single GPU using frameworks like Triton Inference Server or vLLM's multi-model support.
3. **Quantization:** Use 4-bit or 8-bit quantization to fit larger models on smaller GPUs without significant quality loss.
4. **Scheduling:** Use Kubernetes with GPU-aware schedulers to pack workloads efficiently. Set resource requests and limits.
5. **Spot/preemptible instances:** Use spot instances for batch inference and fine-tuning jobs (with checkpointing) to reduce costs by 60-80%.
6. **Monitoring:** Track GPU utilization, memory usage, and inference throughput. Alert on underutilization (wasted cost) or saturation (performance issues).
7. **Autoscaling:** Scale GPU nodes based on inference queue depth or latency targets. Include cool-down periods since GPU instances take time to start.

---

---

## Section 5: Advanced AI Engineering

### Fine-Tuning LLMs

**Q39: When should you fine-tune an LLM vs. using prompt engineering or RAG?**

**A:**

| Approach | When to Use |
|---|---|
| **Prompt engineering** | General tasks, rapid iteration, when the base model already has the knowledge, low data availability |
| **RAG** | When the model needs access to external/updated knowledge, domain-specific documents, or proprietary data |
| **Fine-tuning** | When you need to change the model's behavior/style, teach new formats, improve on a specific task with consistent patterns, reduce prompt length, or when prompt engineering has hit its ceiling |

Fine-tune when:
- You have 100+ high-quality examples of desired input-output pairs.
- You need consistent output format/style that prompting cannot reliably achieve.
- You want to reduce latency by eliminating lengthy few-shot examples from prompts.
- You need domain-specific terminology or reasoning patterns.
- You want to distill a larger model's capabilities into a smaller, cheaper model.

Do not fine-tune when:
- Your data changes frequently (use RAG instead).
- You have fewer than 50-100 examples.
- Prompt engineering achieves acceptable results.

---

**Q40: Explain the differences between full fine-tuning, LoRA, and QLoRA.**

**A:**

| Method | Parameters Trained | Memory Requirement | Training Speed | Use Case |
|---|---|---|---|---|
| **Full fine-tuning** | All model parameters | Very high (4x model size) | Slow | Maximum quality, unlimited compute |
| **LoRA (Low-Rank Adaptation)** | Small low-rank matrices added to attention layers | Moderate (model + ~1-10% extra) | Fast | Good quality with limited GPU memory |
| **QLoRA** | Same as LoRA but base model is 4-bit quantized | Low (4-bit model + LoRA adapters) | Moderate | Fine-tuning large models on consumer GPUs |

**LoRA** works by freezing the original model weights and injecting trainable low-rank decomposition matrices into each attention layer. Instead of updating a weight matrix W (d x d), it learns two smaller matrices A (d x r) and B (r x d) where r << d, and the update is W + BA.

**QLoRA** extends LoRA by quantizing the base model to 4-bit precision (NormalFloat4), using double quantization for further compression, and paging optimizer states to CPU when GPU memory is exhausted. This enables fine-tuning a 65B parameter model on a single 48GB GPU.

---

**Q41: How do you prepare a dataset for fine-tuning an LLM?**

**A:**

1. **Data collection:** Gather examples of desired input-output pairs from production logs, human annotations, or synthetic generation (using a stronger model).
2. **Format:** Structure data in the model's expected format:
   ```json
   {"messages": [
     {"role": "system", "content": "You are a helpful assistant..."},
     {"role": "user", "content": "..."},
     {"role": "assistant", "content": "..."}
   ]}
   ```
3. **Quality filtering:**
   - Remove duplicates and near-duplicates.
   - Filter out low-quality or incorrect examples.
   - Ensure diverse coverage of edge cases and topics.
4. **Balanced representation:** Ensure the dataset represents all expected use cases proportionally.
5. **Train/validation split:** Typically 80/20 or 90/10. Ensure no data leakage between splits.
6. **Data augmentation:** Paraphrase inputs, vary formatting, add edge cases.
7. **Size guidelines:** Start with 50-100 high-quality examples for basic style changes, 500-1000 for more complex behaviors, 10K+ for significant capability changes.

---

**Q42: What are the key hyperparameters to tune when fine-tuning, and how do they affect results?**

**A:**

| Hyperparameter | Typical Range | Effect |
|---|---|---|
| **Learning rate** | 1e-5 to 5e-5 (full), 1e-4 to 3e-4 (LoRA) | Too high: catastrophic forgetting, instability. Too low: underfitting, slow convergence |
| **Epochs** | 1-5 | More epochs risk overfitting on small datasets. Monitor validation loss. |
| **Batch size** | 4-32 (effective, with gradient accumulation) | Larger batches give more stable gradients but require more memory |
| **LoRA rank (r)** | 8-64 | Higher rank = more expressiveness but more parameters. 16-32 is usually sufficient |
| **LoRA alpha** | 16-64 (typically 2x rank) | Scaling factor for LoRA updates. Higher alpha = stronger adaptation |
| **Weight decay** | 0.01-0.1 | Regularization to prevent overfitting |
| **Warmup ratio** | 0.03-0.1 | Gradually increase learning rate to stabilize early training |
| **Max sequence length** | Task-dependent | Truncation affects long examples; padding wastes compute |

Monitor training loss, validation loss, and specific evaluation metrics. Stop training when validation loss starts increasing (early stopping).

---

### Custom Model Training

**Q43: When would you train a custom model from scratch vs. fine-tuning a pre-trained model?**

**A:** Train from scratch (extremely rare) when:
- You need a model for a completely novel domain with no relevant pre-trained models (e.g., a new programming language, alien DNA sequences).
- Data privacy requirements prevent using any pre-trained model (the pre-training data is unknown/untrusted).
- You need a very specific architecture not available in existing models.
- You have enormous amounts of domain-specific data (billions of tokens) and compute budget.

Fine-tune a pre-trained model (almost always preferred) when:
- The target domain shares linguistic/structural patterns with natural language.
- You have limited data (thousands to millions of examples).
- You want to leverage the general knowledge and reasoning capabilities the model already has.
- You have limited compute budget.

In practice, 99%+ of real-world use cases are served by fine-tuning or prompt engineering on pre-trained models.

---

**Q44: Explain the concept of knowledge distillation and how it applies to LLMs.**

**A:** Knowledge distillation transfers knowledge from a large "teacher" model to a smaller "student" model:

1. **Generate training data:** Run the teacher model (e.g., GPT-4) on a large set of prompts to generate high-quality responses.
2. **Train the student:** Fine-tune a smaller model (e.g., Llama-7B) on the teacher's outputs.
3. **Optional soft-label distillation:** If you have access to the teacher's logits (probability distributions over tokens), train the student to match those distributions, not just the argmax tokens. This preserves more nuanced knowledge.

Applications:
- **Cost reduction:** Replace expensive API calls with a cheap self-hosted model that has learned the teacher's behavior for your specific use case.
- **Latency reduction:** Smaller models inference faster.
- **Specialization:** The student learns only what's relevant to your domain, potentially outperforming the teacher on narrow tasks.

Limitations: Student quality is bounded by teacher quality. The approach may not capture the teacher's full reasoning capability on novel queries.

---

**Q45: What are the common evaluation metrics for custom-trained language models?**

**A:**

| Category | Metrics | Description |
|---|---|---|
| **General quality** | Perplexity | How well the model predicts the next token. Lower is better. |
| **Generation quality** | BLEU, ROUGE, METEOR | Compare generated text against reference texts. Useful for translation, summarization. |
| **Semantic similarity** | BERTScore, embedding cosine similarity | Measure meaning preservation beyond surface-level word matching. |
| **Task-specific** | Accuracy, F1, Exact Match | For classification, extraction, Q&A tasks. |
| **LLM-as-judge** | GPT-4 scoring | Use a stronger model to rate outputs on criteria (relevance, accuracy, helpfulness). |
| **Human evaluation** | Elo rating, preference ranking | Gold standard but expensive and slow. |
| **Safety** | Toxicity score, refusal rate | Ensure the model doesn't generate harmful content. |
| **Efficiency** | Tokens/second, latency, memory usage | Practical deployment metrics. |

Always evaluate on a held-out test set that was not seen during training. Use multiple metrics — no single metric captures overall quality.

---

### Advanced RAG (Hybrid Search)

**Q46: What is Hybrid Search in RAG, and why is it better than pure vector search?**

**A:** Hybrid search combines **dense retrieval** (vector/semantic search) with **sparse retrieval** (keyword/BM25 search) to get the best of both:

- **Dense retrieval:** Uses embedding vectors to find semantically similar documents. Excels at understanding meaning, paraphrases, and conceptual matches. Weak at exact keyword matching and rare terms.
- **Sparse retrieval (BM25):** Uses term frequency and inverse document frequency for keyword matching. Excels at exact matches, proper nouns, technical terms, and rare words. Weak at understanding synonyms and paraphrases.
- **Hybrid:** Runs both searches and combines results using reciprocal rank fusion (RRF) or weighted scoring.

Example where hybrid wins: Query "Python GIL workaround" — dense retrieval finds documents about concurrency in Python; BM25 finds documents specifically mentioning "GIL." The hybrid approach surfaces documents that discuss both.

---

**Q47: Explain Reciprocal Rank Fusion (RRF) and how it's used in hybrid search.**

**A:** RRF is a method for combining ranked results from multiple retrieval systems:

```
RRF_score(doc) = Σ 1 / (k + rank_i(doc))
```

Where `k` is a constant (typically 60), and `rank_i(doc)` is the document's rank in the i-th retrieval system.

For each document that appears in any result list:
1. Find its rank in each retrieval system (if absent, assign a high rank like infinity).
2. Compute the RRF score by summing the reciprocal of (k + rank) across all systems.
3. Sort all documents by their combined RRF score.

Benefits:
- Score-agnostic — works even when different systems use incomparable scoring scales.
- Robust — naturally handles documents missing from one system's results.
- No training required — purely rank-based, no learned weights needed.
- Consistently outperforms either retrieval system alone.

---

**Q48: What are advanced chunking strategies for RAG systems?**

**A:**

1. **Fixed-size chunking:** Split by token count (e.g., 512 tokens) with overlap (e.g., 50 tokens). Simple but can break mid-sentence/concept.
2. **Recursive character splitting:** Split by paragraphs, then sentences, then words, respecting natural boundaries. Better semantic coherence.
3. **Semantic chunking:** Use embedding similarity between consecutive sentences. Start a new chunk when similarity drops below a threshold (indicating a topic change).
4. **Document-structure-aware:** Use headings, sections, and document structure to define chunk boundaries. Preserves logical units.
5. **Parent-child chunking:** Index small chunks for precise retrieval but return the larger parent chunk for context. Combines retrieval precision with context completeness.
6. **Agentic chunking:** Use an LLM to determine optimal chunk boundaries based on content understanding.
7. **Late chunking:** Encode the full document first, then chunk the embeddings while preserving full-document context in each chunk's representation.

Best practice: experiment with multiple strategies and evaluate retrieval quality on your specific data and queries.

---

**Q49: Describe a multi-stage RAG pipeline with re-ranking.**

**A:**

1. **Initial retrieval (Recall-focused):** Retrieve a large candidate set (e.g., top 50-100 documents) using hybrid search (vector + BM25). Prioritize recall over precision.
2. **Re-ranking (Precision-focused):** Use a cross-encoder re-ranker (e.g., Cohere Rerank, bge-reranker, ColBERT) to re-score candidate documents against the query. Cross-encoders jointly encode the query-document pair, producing more accurate relevance scores than bi-encoder retrieval.
3. **Filtering:** Apply metadata filters, deduplication, and relevance threshold cutoffs. Keep top 3-5 most relevant documents.
4. **Context compression:** Optionally extract only the most relevant passages/sentences from each document rather than including the full chunk.
5. **Prompt assembly:** Combine the refined context with the query and system prompt.
6. **Generation with citation:** The LLM generates a response grounded in the retrieved context, with inline citations.

This multi-stage approach achieves significantly better answer quality than single-stage retrieval because the re-ranker acts as a precision filter on top of a high-recall first stage.

---

**Q50: How do you evaluate RAG system quality?**

**A:**

**Retrieval metrics:**
- **Recall@K:** What fraction of relevant documents are in the top K results?
- **Precision@K:** What fraction of the top K results are relevant?
- **MRR (Mean Reciprocal Rank):** Average of 1/rank of the first relevant result.
- **NDCG:** Measures ranking quality considering the position of relevant documents.

**Generation metrics:**
- **Faithfulness:** Does the answer only contain information from the retrieved context (no hallucination)?
- **Relevance:** Does the answer address the user's question?
- **Completeness:** Does the answer cover all relevant aspects from the context?

**End-to-end metrics:**
- **Answer correctness:** Compared against ground-truth answers.
- **Citation accuracy:** Are sources correctly attributed?

**Evaluation frameworks:** RAGAS, TruLens, DeepEval provide automated RAG evaluation using LLM-as-judge for faithfulness, relevance, and context quality.

---

**Q51: What are the common failure modes in RAG systems, and how do you address them?**

**A:**

| Failure Mode | Cause | Solution |
|---|---|---|
| **Irrelevant retrieval** | Poor embeddings, wrong chunk size | Better embedding models, optimize chunking, add re-ranking |
| **Missed retrieval** | Query-document vocabulary mismatch | Hybrid search, query expansion, HyDE (Hypothetical Document Embeddings) |
| **Context overflow** | Too many retrieved chunks | Re-ranking + top-K filtering, context compression |
| **Hallucination despite context** | Model ignores context or confabulates | Stronger prompting ("Only use provided context"), smaller context windows, faithfulness checks |
| **Outdated information** | Stale index | Incremental indexing, metadata-based freshness filtering |
| **Lost in the middle** | Model pays less attention to middle chunks | Place most relevant chunks at the beginning and end of the context |
| **Multi-hop failure** | Answer requires combining info from multiple documents | Iterative retrieval, graph-based RAG, agent-based RAG loops |

---

### LLMOps / MLOps

**Q52: What is LLMOps, and how does it differ from traditional MLOps?**

**A:**

| Aspect | Traditional MLOps | LLMOps |
|---|---|---|
| **Model artifacts** | Trained model weights, feature pipelines | Prompts, fine-tuned adapters, RAG indices, model configs |
| **Versioning** | Model versions, dataset versions | Prompt versions, context templates, retrieval configs |
| **Evaluation** | Accuracy, F1, AUC on test sets | LLM-as-judge, human eval, domain-specific rubrics |
| **Deployment** | Model serving (TF Serving, Triton) | LLM serving (vLLM, TGI) + orchestration layer |
| **Monitoring** | Data drift, prediction drift | Prompt injection, hallucination rates, cost per query |
| **Data pipeline** | Feature engineering, ETL | Document ingestion, chunking, embedding, indexing |
| **Iteration cycle** | Retrain model (hours/days) | Update prompt (minutes), re-index documents, fine-tune |
| **Cost model** | Compute for training + serving | Per-token API costs + compute for serving |

LLMOps adds unique challenges: non-deterministic outputs, prompt management as code, retrieval pipeline optimization, and multi-model orchestration.

---

**Q53: How do you implement prompt versioning and management in production?**

**A:**

1. **Prompts as code:** Store prompts in version control alongside application code. Use template files with variable interpolation.
2. **Prompt registry:** Use a dedicated prompt management system (Langfuse, LangSmith Hub, PromptLayer) that versions prompts independently of code deployments.
3. **Parameterized prompts:** Separate static template from dynamic variables. Version the template, inject variables at runtime.
4. **A/B testing:** Deploy multiple prompt versions simultaneously, route traffic, and compare metrics.
5. **Rollback capability:** Instantly revert to a previous prompt version without code deployment.
6. **Audit trail:** Log which prompt version was used for every request for debugging and compliance.
7. **Testing pipeline:** Run prompt changes against an evaluation dataset before deploying to production. Gate deployment on quality thresholds.

```
prompts/
  v1/
    system_prompt.txt
    few_shot_examples.json
  v2/
    system_prompt.txt
    few_shot_examples.json
  config.yaml  # active version mapping
```

---

**Q54: Describe a CI/CD pipeline for an LLM-powered application.**

**A:**

1. **Code commit:** Developer pushes changes (prompts, tools, retrieval configs, application code).
2. **Lint and static checks:** Validate prompt templates, check for common issues (missing variables, format errors).
3. **Unit tests:** Test individual tools, parsers, and utility functions.
4. **Integration tests:** Test agent workflows end-to-end with mocked LLM responses (deterministic).
5. **Evaluation suite:** Run the application against a curated evaluation dataset with real LLM calls. Score outputs using automated metrics (LLM-as-judge, heuristics).
6. **Quality gate:** Compare evaluation scores against baseline. Fail if scores regress beyond thresholds.
7. **Cost estimation:** Estimate the per-request cost impact of changes (new prompts may be longer).
8. **Staging deployment:** Deploy to a staging environment. Run smoke tests with production-like traffic.
9. **Canary deployment:** Route a small percentage of production traffic to the new version. Monitor metrics.
10. **Full rollout:** If canary metrics are healthy, roll out to 100% of traffic.
11. **Post-deployment monitoring:** Track quality metrics, error rates, latency, and costs. Auto-rollback if anomalies are detected.

---

**Q55: How do you handle model versioning and migration in production?**

**A:**

1. **Version configuration:** Store model identifiers (model name, version, provider) in configuration, not hardcoded.
   ```yaml
   models:
     primary: gpt-4-turbo-2024-04-09
     fallback: gpt-3.5-turbo-0125
     embedding: text-embedding-3-small
   ```
2. **Shadow testing:** Run new model versions in shadow mode (process real traffic but don't serve responses to users) to compare outputs.
3. **Gradual migration:** Use feature flags to route a percentage of traffic to the new model. Increase gradually while monitoring quality.
4. **Evaluation before migration:** Run the full evaluation suite against the new model. Compare against the current production model.
5. **Prompt adaptation:** Model upgrades often require prompt adjustments. Test prompt compatibility with the new model.
6. **Rollback plan:** Maintain the ability to instantly switch back to the previous model version.
7. **Deprecation handling:** Monitor provider deprecation timelines and plan migrations proactively.

---

### Scalable, Reliable AI Architecture

**Q56: How do you design a fault-tolerant LLM application architecture?**

**A:**

1. **Provider failover:** Configure multiple LLM providers (OpenAI + Anthropic + self-hosted). Automatically route to backup when the primary fails.
   ```python
   try:
       response = openai_client.chat(messages)
   except (RateLimitError, APIError):
       response = anthropic_client.chat(messages)
   ```
2. **Circuit breakers:** After N consecutive failures to a provider, stop sending requests for a cooldown period. Prevents cascading failures.
3. **Request queuing:** Buffer requests in a durable queue (Redis, SQS). Workers pull and process at a sustainable rate. Prevents overload.
4. **Graceful degradation:** If the full agent pipeline fails, fall back to simpler responses (cached answers, template responses, or "I'm unable to help right now").
5. **Timeout management:** Set aggressive timeouts on LLM calls (30-60s). Return partial results or cached responses on timeout.
6. **Idempotency:** Design tool executions to be idempotent so retries don't cause duplicate actions.
7. **Health checks:** Implement deep health checks that verify LLM connectivity, vector store availability, and tool accessibility.
8. **Data redundancy:** Replicate vector stores and databases across availability zones.

---

**Q57: Explain the architecture of a production RAG system at scale.**

**A:**

```
User Request
    │
    ▼
[API Gateway / Load Balancer]
    │
    ▼
[Application Server (Stateless)]
    ├── Query Understanding (intent classification, query rewriting)
    ├── Retrieval Service
    │     ├── Vector Store (Pinecone/Weaviate/Qdrant) ── Dense retrieval
    │     ├── Search Engine (Elasticsearch/OpenSearch) ── Sparse retrieval (BM25)
    │     └── Reciprocal Rank Fusion ── Combine results
    ├── Re-ranking Service (Cohere Rerank / cross-encoder)
    ├── Context Assembly
    ├── LLM Service
    │     ├── Primary: OpenAI API
    │     └── Fallback: Self-hosted (vLLM)
    ├── Response Cache (Redis)
    └── Observability (Langfuse/LangSmith)
    │
    ▼
[Response to User]

Background Services:
├── Document Ingestion Pipeline (chunking, embedding, indexing)
├── Index Refresh (incremental updates, re-embedding)
├── Evaluation Pipeline (automated quality checks)
└── Cost & Usage Analytics
```

Key scaling patterns:
- Separate retrieval and generation into independent services that scale independently.
- Use read replicas for vector stores under heavy query load.
- Implement caching at multiple levels (query cache, embedding cache, response cache).
- Use async processing for non-real-time workloads (batch embedding, re-indexing).

---

**Q58: How do you implement guardrails for production AI systems?**

**A:** Guardrails protect against unsafe, off-topic, or low-quality AI outputs:

**Input guardrails:**
- **Topic classification:** Reject or redirect off-topic queries.
- **PII detection:** Detect and mask personal information before sending to LLMs.
- **Prompt injection detection:** Classify inputs as potential injection attempts.
- **Rate limiting:** Prevent abuse through request throttling.

**Output guardrails:**
- **Content safety filters:** Check outputs for toxicity, bias, or harmful content (using classifier models or APIs like OpenAI Moderation).
- **Factuality checks:** Verify outputs against retrieved context (faithfulness scoring).
- **Format validation:** Ensure structured outputs match expected schemas.
- **PII leakage prevention:** Scan outputs for inadvertent exposure of sensitive data.
- **Hallucination detection:** Compare generated claims against source documents.

**Implementation frameworks:** NeMo Guardrails (NVIDIA), Guardrails AI, custom rule-based pipelines, or LLM-as-judge validators.

---

**Q59: How do you handle multi-tenancy in an AI platform?**

**A:**

1. **Data isolation:** Each tenant's documents, embeddings, and conversation history are stored in separate namespaces, collections, or schemas.
   - Vector stores: Use collection-per-tenant or metadata filtering.
   - Databases: Row-level security or separate schemas.
2. **Model configuration:** Each tenant may have different model preferences, system prompts, tools, and fine-tuned adapters.
3. **Rate limiting:** Per-tenant rate limits and token budgets to prevent noisy-neighbor problems.
4. **Cost attribution:** Track and attribute LLM costs per tenant for billing.
5. **Access control:** Tenant-specific API keys, RBAC for tools and data sources.
6. **Customization:** Tenant-specific prompt templates, knowledge bases, and agent configurations stored in a tenant configuration service.
7. **Scaling:** Design for uneven load distribution — some tenants will generate 100x more traffic than others. Use queue-based processing with per-tenant fairness.

---

**Q60: What strategies do you use for testing LLM applications at scale?**

**A:**

1. **Unit tests:** Test deterministic components (tools, parsers, formatters) with traditional unit tests.
2. **Snapshot tests:** Record LLM responses and verify that future responses are semantically similar (not exact match) using embedding similarity.
3. **Evaluation datasets:** Curate datasets of (input, expected_output) pairs. Run the full pipeline and score using automated metrics.
4. **LLM-as-judge:** Use a strong model to evaluate outputs on multiple criteria (relevance, accuracy, completeness, safety). Average scores across the dataset.
5. **Regression testing:** On every change, run the evaluation suite and compare against baseline scores. Alert on regressions.
6. **Adversarial testing:** Red-team the system with prompt injection attempts, edge cases, and out-of-distribution queries.
7. **Load testing:** Simulate production traffic patterns to verify latency, throughput, and error rates under load.
8. **Integration tests with mocks:** For CI speed, mock LLM calls with recorded responses. Run full integration tests with real LLMs on a schedule (nightly).
9. **Canary analysis:** Compare metrics between canary and production after each deployment.

---

*This document provides a comprehensive foundation for interview preparation across AI agents, production systems, and advanced engineering topics. Answers should be adapted based on the specific role level (junior/senior/staff) and the company's technology stack.*
