# ai-articles

Self-contained LinkedIn article generators. Each article is a Python script that renders PlantUML diagrams and assembles a Word document ready for publishing.

## Articles

### [AI Agent Patterns](ai-agent-patterns/)

A comprehensive reference from graph fundamentals through production-grade multi-agent systems.

**Covers:**
- Part 0 — Foundations: directed graphs, DAGs, state machines, MapReduce, vector embeddings
- Part 1 — Eight agent patterns: REPL, ReAct, Plan-and-Execute, Reflection, Multi-Agent, Tree of Thoughts, Memory-Augmented, Event-Driven
- Part 2 — LangGraph in depth: state machine core, Human-in-the-Loop, Map-Reduce via Send API
- Part 3 — Framework comparison: LangGraph vs AutoGen vs CrewAI (control flow, topology, memory, HITL, determinism spectrum)
- Part 4 — LangGraph production persistence: checkpointing, resume after failure, time-travel / branching
- Part 5 — Real project analysis: ai-kb-quiz, ai-kd, ai-threat-state-graph, ai-procwatch-mcp, ai-asset-sweeper

28 PlantUML diagrams. Source: [`ai_agent_patterns.md`](ai-agent-patterns/ai_agent_patterns.md)

---

### [Stop Burning Your AI Budget](efficient-ai-usage/)

Cost and quality optimisation patterns for LLM applications.

**Covers:** multi-agent pipelines (model per step), prompt/token caching, RAG with vector DBs,
local model offloading, context window discipline, programmable tool calling, async batching,
and the full efficient AI stack.

5 PlantUML diagrams. Source: [`efficient_ai_usage.md`](efficient-ai-usage/efficient_ai_usage.md)

---

## Usage

```bash
pip install -r requirements.txt
```

```bash
# Generate either article
cd ai-agent-patterns  && python generate.py
cd efficient-ai-usage && python generate.py
```

Output (`.puml`, `.png`, `.docx`) is written to the `build/` subdirectory inside each article folder.

**Requires:** Java on `PATH` (for PlantUML) and `C:\tools\plantuml.jar`.

## Structure

```
ai-articles/
├── requirements.txt
├── ai-agent-patterns/
│   ├── generate.py           # article generator
│   ├── ai_agent_patterns.md  # content source
│   └── build/                # generated artifacts (gitignored)
└── efficient-ai-usage/
    ├── generate.py           # article generator
    └── build/                # generated artifacts (gitignored)
```
