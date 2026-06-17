# Production-Grade Agents with LangChain & LangSmith — An FDE Curriculum

A self-paced curriculum for going from "I know Python" to "I ship reliable,
observable, eval-backed LLM agents for clients" — the core job of a **Forward
Deployed Engineer (FDE)**.

> **What an FDE actually does:** sits with a customer, understands their messy
> real-world problem, and ships a working agent *into their environment* fast —
> then makes it reliable enough that they trust it. That means you optimize for
> three things this curriculum keeps coming back to: **speed to first demo**,
> **observability** (you can explain *why* it did that), and **evaluation** (you
> can prove it got better, not just different).

---

## How to use this

- **Time:** ~8–10 weeks part-time (10–12 hrs/week), or ~3 weeks full-time.
- **Format:** each phase has *Concepts → Build → Prove it*. Don't skip "Prove
  it" — that's the part that separates a demo from production.
- **Stack assumption:** Python 3.11+, `uv` or `poetry`, an Anthropic API key
  (Claude) and/or OpenAI key, and a free LangSmith account.
- **Capstone:** a multi-tool agent deployed behind an API, fully traced in
  LangSmith, with a regression eval suite that runs in CI.

A note on models: examples default to **Claude** (`claude-sonnet-4-6` for most
agent work, `claude-opus-4-8` when you need maximum reasoning, `claude-haiku-4-5`
for cheap/fast classification). LangChain is provider-agnostic — swapping to
`gpt-*` or open models is a one-line change, which is itself an FDE skill
(clients have provider constraints).

---

## Phase 0 — Foundations (before you touch LangChain)

You cannot debug an agent framework if you don't understand the primitives it
wraps. Spend 2–3 days here.

**Concepts**
- How an LLM API call actually works: messages, roles (system/user/assistant),
  tokens, temperature, max tokens, stop sequences.
- **Tool / function calling** at the raw API level — the model returns a
  structured request to call a function; *your code* runs it and feeds the
  result back. This loop *is* what an "agent" is.
- Structured output (JSON mode / schema-constrained output).
- Streaming responses.
- Cost & latency: tokens → dollars, why prompt caching matters.

**Build**
- Call the Claude API directly (the `anthropic` SDK) with no framework. Make a
  tool-calling loop *by hand*: define a `get_weather` tool, let the model ask
  for it, run it, return the result, loop until done.

**Prove it**
- You can explain, without notes, the full request/response cycle of one
  tool-calling turn — including where *your* code runs vs. where the model runs.

📚 Anthropic docs: Messages API, Tool use. (For Claude specifics — model IDs,
pricing, prompt caching, token counting — read the official API docs; don't
trust memorized numbers.)

---

## Phase 1 — LangChain core & LCEL

**Concepts**
- Why LangChain exists: a common interface over models, prompts, tools, parsers,
  and retrievers so you can swap components without rewrites.
- **Chat models** (`ChatAnthropic`, `ChatOpenAI`) and the unified message types.
- **Prompt templates** and partials.
- **LCEL** (LangChain Expression Language): the `|` pipe operator, `Runnable`s,
  `.invoke()` / `.stream()` / `.batch()` / async variants.
- **Output parsers** & `.with_structured_output(PydanticModel)` — the
  production-correct way to get typed data out.
- **Tool definitions** via the `@tool` decorator and Pydantic arg schemas.

**Build**
- A "support-ticket classifier" chain: input ticket text → structured output
  `{category, urgency, summary}` using `.with_structured_output()`.
- A small RAG chain (we go deeper in Phase 3): embed a handful of docs, retrieve,
  stuff into a prompt, answer with citations.

**Prove it**
- Swap the model from Claude to another provider by changing **one line**, and
  your chain still passes its tests. That decoupling is the whole point.

📚 LangChain docs: "Chat models", "LCEL", "Structured outputs", "Tools".

---

## Phase 2 — Agents with LangGraph (the real core)

This is where modern LangChain agent development lives. The old
`AgentExecutor` is legacy; **LangGraph** is the current, production-grade way to
build agents.

**Concepts**
- Why graphs beat linear chains: agents need loops, branches, retries, and
  human-in-the-loop — that's a state machine, not a pipeline.
- **State**: a typed dict that flows through nodes; reducers for accumulating
  messages.
- **Nodes & edges**, conditional edges, the **ReAct** pattern (reason → act →
  observe → repeat).
- `create_react_agent` prebuilt, then *building the same thing by hand* so you
  understand it.
- **Checkpointers / persistence**: how a graph remembers across turns (threads),
  and how that enables pause/resume.
- **Human-in-the-loop**: interrupting before a risky tool call for approval.
- **Memory**: short-term (thread state) vs. long-term (a store).
- **Multi-agent** patterns: supervisor, hand-offs, subgraphs — and when *not* to
  reach for them (most problems don't need a swarm).
- **Streaming** intermediate steps so a UI can show "thinking" and tool calls.

**Build**
- A research assistant agent: tools for web search, a calculator, and a
  retriever over the client's docs. Built with `create_react_agent` first, then
  re-implemented as an explicit `StateGraph`.
- Add a checkpointer so it holds a multi-turn conversation.
- Add a human-approval interrupt before any tool that "writes" (sends email,
  files a ticket).

**Prove it**
- Kill the process mid-conversation, restart, and resume the same thread from the
  checkpoint. Trigger the human-approval interrupt and both approve and reject a
  tool call.

📚 LangGraph docs: "Quickstart", "StateGraph", "Persistence", "Human-in-the-loop",
"Multi-agent".

---

## Phase 3 — Retrieval / RAG done properly

Most client agents are RAG agents. Naive RAG demos beautifully and fails in
production; this phase is about the failure modes.

**Concepts**
- Document loading, **chunking strategies** (and why fixed-size chunking is
  usually wrong), metadata.
- Embeddings & **vector stores** (start with Chroma/FAISS locally; know that
  pgvector / Pinecone / etc. exist for scale).
- Retrieval quality: similarity vs. MMR, **hybrid search** (keyword + vector),
  **re-ranking**, metadata filtering.
- **Agentic RAG**: let the agent decide *whether* and *what* to retrieve, rewrite
  queries, and do multi-hop retrieval — instead of always stuffing top-k.
- Citations & groundedness; reducing hallucination.

**Build**
- Turn the Phase 2 retriever into an agentic RAG tool with query rewriting and a
  re-ranking step. Return answers *with* source citations.

**Prove it**
- Build a 20–30 question eval set (next phase makes this rigorous) and show
  retrieval improvements move a measurable groundedness/correctness number — not
  just vibes.

📚 LangChain docs: "Retrieval", "Vector stores", "Retrievers". LangGraph:
"Agentic RAG" tutorial.

---

## Phase 4 — LangSmith: observability & tracing

This is the FDE superpower. When a client says "it gave a wrong answer
yesterday," you open the trace and *show them exactly what happened*.

**Concepts**
- **Tracing**: set `LANGSMITH_TRACING=true` + API key and every LangChain/
  LangGraph run is captured — full input/output of every step, latency, token
  counts, cost, errors.
- Reading a trace tree: which tool was called, with what args, what it returned,
  where the latency/cost went.
- **Projects** to separate environments (dev / staging / prod / per-client).
- Tracing **non-LangChain code** with the `@traceable` decorator and the
  `wrap_anthropic` / `wrap_openai` wrappers — so even raw SDK calls show up.
- **Metadata, tags, and run names** for filtering production traffic.
- **Feedback**: attaching thumbs-up/down or scores to runs from your app.
- Monitoring dashboards: latency p50/p99, error rate, cost over time.

**Build**
- Instrument the Phase 2/3 agent with full tracing. Add per-client `metadata`
  and tags. Wire a `/feedback` endpoint that logs user feedback onto the trace.

**Prove it**
- Given a "bad" run, find it in LangSmith by filtering on tags/feedback, open the
  trace, and write a one-paragraph root-cause: which step failed and why.

📚 LangSmith docs: "Tracing", "Annotate code (@traceable)", "Add metadata & tags".

---

## Phase 5 — Evaluation (the hardest, most valuable skill)

Anyone can build a demo. FDEs get trusted because they can **prove** an agent
works and **prove** a change made it better. This phase is what makes you
production-grade.

**Concepts**
- **Datasets** in LangSmith: curated input/output examples; building them from
  production traces (turn real failures into test cases).
- **Evaluators**:
  - *Heuristic / code*: exact match, JSON schema valid, contains-citation,
    latency under budget.
  - *LLM-as-judge*: correctness, groundedness, helpfulness, with a rubric — and
    knowing its limitations (judge bias, need for calibration).
  - *Pairwise*: is version B better than version A?
- **Offline eval**: run a dataset through your agent, score it, compare
  experiments side by side in the LangSmith UI.
- **Online eval**: sample production traffic and score it live.
- **Agent-specific eval**: trajectory evaluation (did it call the right tools in
  a sensible order?), not just final-answer eval.
- **Regression testing**: an eval suite that runs in CI and **blocks a merge** if
  quality drops below a threshold.

**Build**
- A dataset of 30–50 cases for your capstone agent. A mix of code evaluators and
  one LLM-as-judge. Run two prompt variants as separate experiments and compare.

**Prove it**
- Make a deliberate prompt regression and watch your CI eval catch it. This is
  the deliverable that wins client trust.

📚 LangSmith docs: "Evaluation", "How-to: evaluate an agent", "LLM-as-judge",
"Run evals in CI".

---

## Phase 6 — Prompt engineering & management

**Concepts**
- The **LangSmith Prompt Hub**: version prompts, pull them at runtime, collaborate
  with non-engineers (a PM can tweak a prompt without a deploy).
- Prompt versioning tied to eval experiments — so you know *which* prompt scored
  what.
- Practical techniques: clear role/task structure, few-shot examples, XML/section
  delimiters, tool-use instructions, controlling verbosity, reducing
  hallucination, prompt caching for cost.
- Playground: iterate on a prompt against real traced inputs.

**Build**
- Move your capstone's system prompt into the Prompt Hub, pull it by version at
  runtime, and tie each version to an eval run.

**Prove it**
- Change a prompt in the Hub (no code deploy), and your running agent picks up the
  new version. Show the eval score for each version.

📚 LangSmith docs: "Prompt Hub", "Playground". Anthropic: prompt engineering
guide.

---

## Phase 7 — Production hardening

The difference between "works on my laptop" and "I left it running at a client."

**Concepts**
- **Reliability**: retries with backoff, timeouts, fallback models
  (`.with_fallbacks()`), graceful degradation when a tool/API is down.
- **Guardrails**: input validation, output validation against schemas, PII
  handling, prompt-injection awareness (especially for RAG and tool use), max
  iteration/cost caps to prevent runaway loops.
- **Cost & latency control**: prompt caching, model routing (cheap model for easy
  cases, escalate to a stronger one), streaming for perceived latency,
  parallelizing tool calls.
- **Security**: never trust tool inputs/outputs blindly; sandbox code execution;
  least-privilege API keys; secrets management.
- **Rate limits & concurrency**: handling 429s, queuing, batching.

**Build**
- Add fallbacks, a global cost/iteration cap, output schema validation, and a
  basic prompt-injection check to your capstone.

**Prove it**
- Simulate a downstream tool outage and an LLM 429; the agent degrades gracefully
  instead of crashing, and the failure is visible in LangSmith.

📚 LangChain: "Fallbacks", "Rate limits". Industry guidance on LLM security
(e.g. OWASP Top 10 for LLM apps).

---

## Phase 8 — Deployment

**Concepts**
- **LangGraph Platform / LangGraph Server**: purpose-built deployment for
  LangGraph agents — gives you persistence, streaming, a task queue, and a
  built-in API + "Agent Chat" UI. Know what it offers before you hand-roll.
- **Self-hosting the hard way**: wrapping your graph in **FastAPI**, containerizing
  with Docker, managing state (Postgres checkpointer), and handling streaming
  (SSE/websockets). Clients often *require* this (their cloud, their VPC).
- Config & secrets per environment; separate LangSmith projects per env.
- CI/CD: tests + the Phase 5 eval gate before deploy.

**Build**
- Deploy the capstone two ways: (1) on LangGraph Platform, (2) as a Dockerized
  FastAPI service with a Postgres checkpointer. Compare the effort/tradeoffs.

**Prove it**
- Hit the deployed endpoint from a separate client, hold a multi-turn
  conversation across requests (state persists), and see every request traced in
  the prod LangSmith project.

📚 LangGraph docs: "Deployment", "LangGraph Platform". FastAPI docs.

---

## Capstone project

Build **one** of these end-to-end, hitting every phase above:

1. **Customer-support agent** — RAG over a knowledge base + tools to look up
   orders and (with human approval) issue refunds. Escalates to a human when
   unsure.
2. **Internal data analyst** — natural-language questions → SQL over a real DB,
   with guardrails (read-only, row caps), charts, and cited results.
3. **Research/briefing agent** — web search + document retrieval that produces a
   sourced brief, with a supervisor coordinating sub-agents.

**Definition of done (this is the FDE bar):**
- [ ] Built on LangGraph with persistence and human-in-the-loop on risky actions.
- [ ] Fully traced in LangSmith with per-client metadata and a feedback loop.
- [ ] A 30–50 case eval dataset with code + LLM-judge evaluators.
- [ ] An eval gate in CI that blocks regressions.
- [ ] Prompts versioned in the Prompt Hub.
- [ ] Fallbacks, cost/iteration caps, and output validation.
- [ ] Deployed behind an API (LangGraph Platform *or* Dockerized FastAPI).
- [ ] A 1-page runbook: how to debug a bad response using a LangSmith trace.

---

## The FDE skills around the code

Technical mastery is table stakes. These get you hired and kept:

- **Demo-driven development:** get *something* working in front of the client in
  days, then harden. A traced, eval-backed prototype beats a polished slide deck.
- **Scoping ruthlessly:** most "we need a multi-agent system" requests are one
  good RAG agent. Talk clients out of complexity.
- **Explaining failures:** open a LangSmith trace and narrate exactly what
  happened. This builds more trust than any feature.
- **Eval as a sales tool:** "here's the number, here's how it improved" closes
  deals and renewals.
- **Working in their environment:** their cloud, their data residency, their
  model provider, their security review. Flexibility is the job.
- **Knowing when not to use an LLM:** sometimes the right answer is a regex, a
  SQL query, or a button.

---

## Reference list

- **LangChain docs** — Chat models, LCEL, Tools, Structured output, Retrieval.
- **LangGraph docs** — Quickstart, StateGraph, Persistence, Human-in-the-loop,
  Multi-agent, Agentic RAG, Deployment.
- **LangSmith docs** — Tracing, `@traceable`, Datasets & Evaluation,
  LLM-as-judge, Prompt Hub, Monitoring, Eval in CI.
- **LangChain Academy** — free structured courses (esp. "Introduction to
  LangGraph").
- **Anthropic docs** — Messages API, Tool use, Prompt engineering, Prompt
  caching, Token counting. (Always check live docs for model IDs & pricing.)
- **DeepLearning.AI short courses** — several free LangChain/agents courses.
- **OWASP Top 10 for LLM Applications** — security baseline.

> Build something at every phase. Reading docs feels like progress; shipping a
> traced, evaluated agent *is* progress. The portfolio of capstones you build
> here is exactly what you'll show in FDE interviews.
