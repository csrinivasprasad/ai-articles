# Stop Burning Your AI Budget

> An Engineer's Guide to Efficient Premium Model Usage

The debate is everywhere in enterprise right now: are we getting productivity returns that
justify the token costs? Finance wants invoices justified. Engineering wants fewer restrictions
on model access. Leadership wants both.

I'm an enterprise software engineer who also runs personal AI projects on my own Claude and
Gemini subscriptions — a knowledge distillation pipeline, a KB quiz engine, and a
process-monitoring MCP server. That combination gives me a useful vantage point: I feel the
cost personally, I see the productivity question professionally, and I've had to find real
answers for both. Here's what I've learned.

---

## The Core Mental Model: Route by Difficulty

Stop treating your AI stack as a single model and start treating it as a **fleet with a
routing layer**. Premium models (Claude Opus, Gemini Ultra) are best for complex multi-step
reasoning, novel synthesis across sources, and high-stakes output where quality directly
affects users. Everything else is a candidate for a cheaper or local alternative.

---

## 1. Multi-Agent Pipelines — Match the Model to the Step

Most workflows aren't a single task — they're a sequence of steps with very different
complexity profiles. Assigning one premium model to the entire pipeline is like hiring a
specialist surgeon to also do your hospital's paperwork. The better approach: decompose your
workflow into steps, then assign the **cheapest model capable of each step**.

```plantuml
@startuml agent_pipeline
!theme plain
skinparam backgroundColor #FAFAFA
skinparam ArrowColor #444444

title Multi-Agent Pipeline — Model per Step

rectangle "Incoming Task" as IN
rectangle "Agent 1\nLocal Model\n(Phi-3 / Mistral 7B)\nClassify + Pre-process" as A1
rectangle "Agent 2\nEmbedding Model\n+ Vector DB\nRetrieve top-K chunks" as A2
rectangle "Agent 3\nMid-tier Model\n(Sonnet / Flash)\nIntermediate reasoning" as A3
rectangle "Agent 4\nPremium Model\n(Opus / Ultra)\nSynthesize & Generate" as A4
rectangle "Agent 5\nSmall Model\n(Haiku / Flash)\nFormat & Validate schema" as A5
rectangle "Structured Output" as OUT

IN --> A1
A1 --> A2
A2 --> A3
A3 --> A4 : escalate only\nif needed
A4 --> A5
A5 --> OUT

note right of A4
  Premium model fires
  only at this step
end note
@enduml
```

Example — a document Q&A skill:

```
Step 1: Parse & clean raw input          → local model (Phi-3, Mistral 7B)
Step 2: Classify intent / route query    → small cloud model (Haiku, Flash)
Step 3: Retrieve relevant chunks (RAG)   → embedding model only, no LLM
Step 4: Synthesize answer from chunks    → premium model (Claude Sonnet/Opus)
Step 5: Format & validate output schema  → small cloud model (Haiku, Flash)
```

Practical implementation tips:

- **Schema contracts:** Define a schema contract between agents — the output of each step is
  the structured input to the next.
- **Single responsibility:** Each agent should have a single responsibility. Agents that do
  too much become expensive and opaque.
- **Confidence scoring:** Build confidence scoring into classification steps — if the cheaper
  model isn't confident, escalate.
- **MCP integration:** With MCP support, wire different model-backed agents as distinct MCP
  servers, each domain-specialized.

---

## 2. Prompt / Token Caching — The Fastest Win

Claude's prompt caching lets you cache a prefix of your context (system prompt, documents,
examples) and pay approximately **10% of the input token cost** on repeated calls that share
that prefix.

```plantuml
@startuml prompt_caching
!theme plain
skinparam backgroundColor #FAFAFA
skinparam ArrowColor #444444
title Prompt Caching — Cost Per Query

rectangle "Without Caching" as WOC {
  rectangle "System Prompt\n800 tokens" as SP1
  rectangle "KB Chunk\n12,000 tokens" as KB1
  rectangle "User Question\n50 tokens" as UQ1
}

rectangle "With Caching" as WC {
  rectangle "System Prompt\n800 tokens\n[CACHED — 10% cost]" as SP2
  rectangle "KB Chunk\n12,000 tokens\n[CACHED — 10% cost]" as KB2
  rectangle "User Question\n50 tokens\n[billed normally]" as UQ2
}

note bottom of WOC
  Every query: 12,850 tokens billed
end note

note bottom of WC
  Every query after first: ~50 tokens + cache read fee
  Saving: >80% on input costs
end note
@enduml
```

Rules for getting cache hits:

- Put **stable content first** (system prompt, docs, few-shot examples).
- Put **variable content last** (the user's actual question).
- Use consistent `cache_control` breakpoints across requests in the same session.
- Cache TTL is **5 minutes** on Anthropic's API — keep call cadence inside that window or
  re-warm explicitly.

In one of my personal projects — a knowledge base quiz engine that repeatedly queries the same
document corpus — this single change reduced input token costs by **over 80%**.

---

## 3. Vector DBs — Ground the Model in What It Cannot Know

Before getting to the cost angle, it's worth understanding why a vector database is necessary
in the first place. Large language models reason from their latent space — knowledge compressed
into weights during training. That works well for general concepts, but it fails in predictable
ways:

- **Version-specific SDK and API references:** The model was trained on docs from months ago.
  It will confidently describe a method signature that no longer exists.
- **Deep domain knowledge:** Windows kernel internals, IRQL rules, minifilter callback
  contracts. Models have surface-level familiarity, not the precision required when the details
  actually matter.
- **Internal project context:** Your component design, your team's architectural decisions,
  your internal APIs. The model has zero knowledge of these by definition.

The fix is **RAG** — injecting authoritative, current context into the prompt so the model
reasons from your ground truth rather than its best guess.

```plantuml
@startuml rag_flow
!theme plain
skinparam backgroundColor #FAFAFA
skinparam ArrowColor #444444

title RAG — Ground the Model in What It Cannot Know

participant "User Query" as Q
participant "Embedding\nModel" as EM
participant "Vector DB" as VDB
participant "Re-ranker" as RR
participant "Premium\nModel" as LLM
participant "Response" as R

Q -> EM : embed query
EM -> VDB : similarity search
VDB --> RR : top-20 candidates
RR --> LLM : top-K chunks (3-5)\nonly relevant context
LLM --> R : grounded answer

note over VDB
  Stores: SDK docs, kernel
  internals, internal design
  docs, version-pinned refs
end note

note over LLM
  Reasons from injected
  context, not latent space
end note
@enduml
```

Sizing guidance:

- **Top-K = 3–5 chunks** is usually enough. More is rarely better and always more expensive.
- **Chunk size:** 512–1024 tokens per chunk gives retrieval precision without truncating context.
- **Re-rank before sending:** use a lightweight cross-encoder or BM25 hybrid.

> Key reframe: RAG isn't primarily a cost optimization — it's a **correctness optimization**
> that also happens to reduce cost. That's a much easier sell in an enterprise conversation
> where accuracy matters more than invoices.

---

## 4. Local Models — Free the Routine Work

Not every task needs a frontier model. Local models (Ollama + Llama 3, Mistral, Phi-3)
running on your own hardware handle a surprising amount of routine work.

| Task                                        | Use Local | Use Premium |
|---------------------------------------------|:---------:|:-----------:|
| Intent classification                       | ✓         |             |
| Entity extraction (structured schema)       | ✓         |             |
| Routing / triage                            | ✓         |             |
| Summarization of well-structured input      | ✓         |             |
| Complex reasoning across multiple documents |           | ✓           |
| Code generation with non-trivial logic      |           | ✓           |
| Novel synthesis, edge cases, ambiguity      |           | ✓           |

My personal knowledge distillation pipeline uses a local 7B model to pre-process and structure
raw input, then sends only the structured output to Claude for synthesis. Token input to Claude
drops **60–70%** because the local model did the cleanup first.

---

## 5. Context Window Discipline

Premium models have large context windows. That doesn't mean you should fill them. Cost scales
linearly with input tokens. Quality does not.

- **Compress before sending:** Summarize prior conversation turns rather than appending them
  verbatim.
- **Structured output contracts:** Ask the model to return JSON with a defined schema —
  smaller, parseable outputs, no rambling preambles.
- **Trim few-shot examples:** Three high-quality examples beat ten mediocre ones — and cost
  70% less.
- **Strip metadata:** Remove headers, footers, page numbers, and formatting artifacts from
  documents before sending.

---

## 6. Programmable Tool Calling — The Model as Orchestrator

Tool calling (function calling) is commonly seen as a way to connect models to external data.
Its bigger value is architectural: it lets the premium model act as an **orchestrator** that
decides what to invoke — and what not to invoke — rather than doing all reasoning inline.

Three efficiency patterns emerge from this:

- **Parallel tool calls reduce round trips:** The model can call `vector_search()` and
  `classify_intent()` in a single response. Instead of two sequential round trips — each
  paying full input token cost — you get one. For pipelines with multiple retrieval steps
  this compounds quickly.
- **Tools as cheap-service wrappers:** Define tools that wrap your local model, embedding
  service, or cache lookup. The premium model decides whether to invoke them based on the
  task. A high-confidence classification routes to `run_local_model()`; a complex synthesis
  stays in-model. The model becomes the router — no separate routing layer needed.
- **Schema enforcement via tool definitions:** Instead of prompt-engineering "return valid
  JSON with fields X, Y, Z" and then parsing brittle free text, define a tool with a strict
  JSON schema. The model is constrained to call it correctly. Output token count drops,
  parsing reliability goes to near-100%, and you eliminate a whole class of retry logic.

```plantuml
@startuml tool_calling
!theme plain
skinparam backgroundColor #FAFAFA
skinparam ArrowColor #444444

title Programmable Tool Calling — Model as Orchestrator

participant "Premium\nModel" as LLM
participant "Tool:\nvector_search()" as T1
participant "Tool:\nclassify_intent()" as T2
participant "Tool:\nrun_local_model()" as T3
participant "Tool:\nfetch_doc_chunk()" as T4
participant "Tool:\nformat_output()" as T5

LLM -> T1 : call (parallel)
LLM -> T2 : call (parallel)
T1 --> LLM : top-K chunks
T2 --> LLM : intent + confidence

alt confidence HIGH — routine task
  LLM -> T3 : delegate to local model
  T3 --> LLM : structured result
else confidence LOW — complex task
  LLM -> T4 : fetch additional context
  T4 --> LLM : doc chunk
  LLM -> LLM : reason over full context
end

LLM -> T5 : enforce output schema
T5 --> LLM : validated JSON
@enduml
```

Practical tips:

- Keep tool descriptions concise — they count against your input tokens on every call.
- Group tools logically: retrieval tools, transformation tools, output tools. Fewer tools in
  scope = less decision overhead for the model.
- Use `tool_choice: "required"` when you always need structured output — prevents the model
  from answering in prose instead.
- Log every tool invocation with latency and token counts. This is where you find the
  expensive surprises.

---

## 7. Async Batching — Don't Pay Rush Rates for Batch Work

Anthropic's Message Batches API gives you **50% off** input and output tokens for non-real-time
workloads with 24-hour turnaround. Nightly evaluations, bulk document processing, quiz
generation pipelines — none of these should hit the synchronous API. Same principle applies
across providers: identify workloads that can tolerate latency and route them to batch
endpoints.

---

## The Full Stack

Each layer below reduces what reaches the premium model. The premium model does what only it
can do — and nothing else.

```plantuml
@startuml full_stack
!theme plain
skinparam backgroundColor #FAFAFA
skinparam ArrowColor #444444

title The Full Efficient AI Stack

rectangle "Incoming Task" as T0
rectangle "Step 1: Classify + Pre-process\nLocal Model — free, fast" as T1
rectangle "Step 2: Vector DB Retrieval\nEmbedding model + top-K only" as T2
rectangle "Step 3: Cache Check\nIs prompt prefix warm?" as T3
rectangle "Step 4: Tool Calling Layer\nParallel calls: retrieve + classify" as T4
rectangle "Step 5: Premium Model\nSynthesize — only what requires it" as T5
rectangle "Step 6: Format + Validate\nTool schema enforcement" as T6
rectangle "Structured Output" as T7

T0 --> T1
T1 --> T2
T2 --> T3
T3 --> T4
T4 --> T5 : escalate if\ncomplexity high
T5 --> T6
T6 --> T7

note right of T3
  Cache hit: pay 10%
  Cache miss: re-warm prefix
end note

note right of T5
  Async Batching API:
  50% off for non-realtime
  workloads
end note
@enduml
```

---

## Answering the Enterprise Debate

When the cost-vs-productivity question comes up at work, the honest answer is: the question is
usually posed too broadly to be useful. It's not "is Claude worth it?" — it's "which steps in
our workflows justify the cost, and which are we routing there out of habit?"

Track these metrics per feature:

- **Input tokens per task** — primary cost driver
- **Cache hit rate** — target >70% for KB-heavy workloads
- **Local/mid-tier deflection rate** — % of steps resolved without a premium call
- **Premium model invocation rate** — how often does the expensive agent actually fire?
- **Output token ratio** — output/input; high ratios mean you're extracting value

---

## The Mindset Shift

Premium models are exceptional tools. Routing every step of every workflow through them is like
staffing a team entirely with principal engineers when juniors, mids, and seniors each have work
suited to their level.

The engineers extracting real value from frontier AI are the ones who build agent pipelines with
**deliberate model assignment** — so that when the premium model runs, it runs on exactly the
right input, at exactly the right step, at exactly the right time.

**Build the infrastructure. Make every premium token count.**
