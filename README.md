# 🔭 Pydantic Logfire — Production LLM Observability

**Module 4 · Instructor: Divesh · Production-Grade LLM Engineering**

---

## What Is This Module About?

When you build an LLM app and it gives a wrong answer, is slow, or costs too much — how do you figure out *why*?

That's what **observability** solves. This module teaches you to use **Pydantic Logfire** to see exactly what's happening inside your LLM app — in real time, on a dashboard.

```
Without observability:   "My app is slow."  ← you have no idea why

With Logfire:            embed_query   →  12ms   ✅
                         retrieve_docs →   4ms   ✅
                         generate_answer → 834ms  ← HERE is your bottleneck
```

---

## The Big Picture

```mermaid
graph TD
    You(["👨‍💻 Your Python Code\n(notebook / app.py)"])

    subgraph LLMs["LLM APIs"]
        Groq["Groq\nllama-3.3-70b"]
        Gemini["Gemini\ngemini-2.5-flash-lite"]
    end

    subgraph Data["Vector Search"]
        JSON["documents.json\n(knowledge base)"]
        FAISS["FAISS\nin-memory index"]
    end

    LF["🔭 Pydantic Logfire\n(OpenTelemetry)"]
    DB[("Logfire Dashboard\nlogfire.pydantic.dev\n\nReal-time traces\nlatency · tokens · cost")]

    You -->|"API calls"| Groq
    You -->|"API calls"| Gemini
    You -->|"loads & embeds"| JSON --> FAISS
    You -.->|"every span + log event"| LF --> DB
    Groq -.->|"auto-instrumented"| LF
    Gemini -.->|"auto-instrumented"| LF

    style LF fill:#f0ad4e,color:#000,stroke:#e67e22
    style DB fill:#2E75B6,color:#fff
```

---

## Three Core Concepts (Learn These First)

### 1. Span — a timed block of code

A **span** wraps a piece of code and measures how long it takes.
It also stores any metadata you attach (model name, user id, token count, etc.).

```mermaid
sequenceDiagram
    participant Code as Your Code
    participant LF as Logfire

    Code->>LF: span starts → "embed_query" (t=0ms)
    Code->>Code: embedding = embed(query)
    Code->>LF: log event → dim=768, latency=12ms
    Code->>LF: span ends (t=12ms) ← duration recorded automatically
```

```python
with logfire.span("embed_query", model="gemini-embedding-2-preview"):
    embedding = embed(query)           # code runs here
    logfire.info("done", dim=768)      # log event inside the span
# span closes → duration stored automatically
```

### 2. Trace — a tree of spans

A **trace** is what you get when you nest spans inside each other.
In the Logfire dashboard, a trace appears as a **waterfall** — like a Gantt chart for your code.

```mermaid
graph LR
    subgraph Trace["One Trace — rag_pipeline (850ms total)"]
        P["rag_pipeline  ████████████████████  850ms"]
        E["  embed_query  ██  12ms"]
        R["  retrieve_docs  █  4ms"]
        G["  generate_answer  ████████████████  834ms  ← bottleneck"]
    end
    P --> E --> R --> G
    style G fill:#fdecea,color:#000
```

### 3. Attribute — searchable metadata

Every keyword argument you pass to `logfire.span()` or `logfire.info()` becomes a **searchable column** in the dashboard.

```python
logfire.info("docs_retrieved",
             num_docs=2,
             topics=["RAG", "Guardrails"],   # filter by topic
             user_id="alice",                # filter by user
             latency_ms=16.2)               # sort by latency
```

---

## Setup

```bash
# 1. Install
pip install -r requirements.txt

# 2. Fill in .env
LOGFIRE_TOKEN  = "pylf_v1_..."    # logfire.pydantic.dev → Settings → Token
GROQ_API_KEY   = "gsk_..."        # console.groq.com
GEMINI_API_KEY = "..."            # aistudio.google.com

# 3. Open your Logfire dashboard in a browser tab, then run the notebook
```

---

## 📓 The Notebook — 7 Experiments

The notebook (`pydantic_logfire.ipynb`) is split into 3 parts:

| Part | Experiments | What You Learn |
|------|-------------|----------------|
| Part 1 | 1–2 | How Logfire works — spans, logs, Pydantic models |
| Part 2 | 3–5 | Auto-instrumentation — trace any LLM with one line |
| Part 3 | 6–7 | RAG tracing + LangGraph agent with tool observability |

---

### Experiment 1 — Configure Logfire & Your First Span

**What you learn:** The three Logfire primitives — configure, log, span.

```mermaid
sequenceDiagram
    participant NB as Notebook Cell
    participant LF as Logfire SDK
    participant DB as Dashboard (browser)

    NB->>LF: logfire.configure(token=...)
    Note over LF: connected to your project

    NB->>LF: logfire.info("notebook_started", part="Part 1")
    LF-->>DB: event appears instantly ✅

    NB->>LF: with logfire.span("data_processing"):
    NB->>LF:     logfire.info("step", n=1)  t=0ms
    NB->>LF:     logfire.info("step", n=2)  t=300ms
    NB->>LF:     logfire.info("step", n=3)  t=500ms
    LF-->>DB: span closes → duration = 500ms shown in waterfall ✅
```

**Key insight:** `logfire.info()` is NOT `print()`. It creates a **structured record** with a timestamp and searchable fields. `logfire.span()` times a block automatically.

---

### Experiment 2 — Structured Logging with Pydantic Models

**What you learn:** Log a Pydantic model → every field becomes a searchable column.

```mermaid
graph LR
    subgraph Bad["❌  print() — useless in production"]
        B1["print(f'Request: {req}')"]
        B2["→ 'Request: user_id=alice, tokens=142, ...'"]
        B3["Not searchable. Not filterable. Lost in terminal."]
    end

    subgraph Good["✅  logfire.info() — production-ready"]
        G1["logfire.info('request', **req.model_dump())"]
        G2["→ user_id column:  alice"]
        G3["→ tokens column:   142"]
        G4["→ model column:    llama-3.3-70b"]
        G5["Query:  user_id = 'alice' AND tokens > 100"]
    end

    style Bad fill:#fdecea,color:#000
    style Good fill:#e8f5e9,color:#000
```

**Key insight:** When you log a Pydantic model, Logfire expands every field into a dedicated searchable column. This is how you do **per-user cost attribution** — filter by `user_id` and sum `output_tokens`.

---

### Experiment 3 — Auto-Instrument Groq (llama-3.3-70b)

**What you learn:** One line of code makes every Groq call traceable — zero extra work.

```mermaid
flowchart LR
    A["logfire.instrument_openai()\n— called once at startup —"]

    subgraph Before["Without instrumentation"]
        B1["llm.invoke(prompt)"] --> B2["Response ✓\n\nbut: no idea what\nmodel/tokens/latency"]
    end

    subgraph After["With instrumentation"]
        C1["llm.invoke(prompt)"] --> C2["Response ✓\n\nAND in dashboard:\nmodel = llama-3.3-70b\ninput_tokens = 24\noutput_tokens = 142\nlatency = 412ms\nfull prompt text\nfull response text"]
    end

    A -->|patches SDK| After

    style A fill:#f0ad4e,color:#000
    style After fill:#e8f5e9,color:#000
    style Before fill:#fdecea,color:#000
```

**Key insight:** Groq's API is **OpenAI-compatible** — it uses the same REST format as OpenAI. So `logfire.instrument_openai()` captures Groq calls automatically without any Groq-specific code.

---

### Experiment 4 — Auto-Instrument Gemini (gemini-2.5-flash-lite)

**What you learn:** The exact same setup works for Gemini — one dashboard, two models.

```mermaid
flowchart TD
    INST["logfire.instrument_openai()\n← called ONCE, instruments everything"]

    G["ChatOpenAI\nbase_url = groq endpoint\nmodel = llama-3.3-70b"]
    M["ChatOpenAI\nbase_url = gemini endpoint\nmodel = gemini-2.5-flash-lite"]

    DB[("Logfire Dashboard\n\nllama-3.3-70b  →  trace 1\ngemini-2.5-flash-lite  →  trace 2\n\nSame dashboard, side by side")]

    INST -->|"auto-traces"| G --> DB
    INST -->|"auto-traces"| M --> DB

    style INST fill:#f0ad4e,color:#000
    style DB fill:#2E75B6,color:#fff
```

**Key insight:** Both Groq and Gemini expose an **OpenAI-compatible endpoint**. Point `ChatOpenAI` at a different `base_url` and Logfire captures both automatically.

---

### Experiment 5 — Side-by-Side Model Comparison (A/B Test)

**What you learn:** Wrap two LLM calls in a parent span → see them as a waterfall → compare latency and tokens.

```mermaid
gantt
    title Trace Waterfall — model_comparison span
    dateFormat  X
    axisFormat  %Lms

    section Parent
    model_comparison (total)  :0, 1500

    section Groq
    groq_call                 :50, 800

    section Gemini
    gemini_call               :850, 1450
```

```mermaid
graph TD
    P["model_comparison\n⏱ total wall time"]
    G["groq_call\nmodel: llama-3.3-70b\n⏱ ~750ms\ntokens: auto-captured"]
    M["gemini_call\nmodel: gemini-2.5-flash-lite\n⏱ ~1200ms\ntokens: auto-captured"]

    P --> G
    P --> M

    style P fill:#D6E4F0,color:#000
    style G fill:#e8f5e9,color:#000
    style M fill:#e3f2fd,color:#000
```

**Key insight:** This is exactly how you do **model A/B testing in production**. Same query, same retrieval, different models. Dashboard tells you which is faster, cheaper, and whether answers differ.

---

### Experiment 6 — Traced RAG Pipeline

**What you learn:** Build a full RAG pipeline from a JSON knowledge base, and trace every stage.

#### How RAG Works

```mermaid
flowchart LR
    J[("documents.json\n6 topic docs")]
    Q(["User Question"])

    subgraph Startup["At startup — runs once"]
        J -->|"json.load()"| D["6 Document objects"]
        D -->|"Gemini API"| E["Embeddings\n768-dim vectors"]
        E -->|"FAISS.from_documents()"| V[("FAISS Index\nin memory")]
    end

    subgraph Query["Every query"]
        Q -->|"embed question"| QE["Question vector"]
        QE -->|"similarity search"| V
        V -->|"top 2 matches"| R["Retrieved Docs"]
        R -->|"format as context"| P["Prompt\n= context + question"]
        P -->|"Groq API"| A(["Answer ✅"])
    end
```

#### What Logfire Captures

```mermaid
graph TD
    SPAN["rag_pipeline  ⏱ total"]
    LOG1["📌 docs_retrieved\ntopics: RAG, Fine-tuning\nnum_docs: 2"]
    LLM["Chat Completion with llama-3.3-70b  ⏱ ~800ms\n🤖 AUTO-INSTRUMENTED\nmodel · input_tokens · output_tokens · prompt · response"]

    SPAN --> LOG1
    SPAN --> LLM

    style SPAN fill:#D6E4F0,color:#000
    style LLM fill:#fff3cd,color:#000
```

**Key insight:** You can see **which documents were retrieved** for any question — crucial for debugging wrong answers. "Why did the LLM say X?" → check `docs_retrieved` → the wrong document was retrieved.

---

### Experiment 7 — LangGraph ReAct Agent with Retriever Tool

**What you learn:** Build an agent that *decides* whether to search the knowledge base, using a plain Python function as a tool.

#### RAG Chain vs ReAct Agent

```mermaid
graph LR
    subgraph Chain["❌  Fixed RAG Chain (Exp 6)"]
        C1["Question"] --> C2["Always embed"] --> C3["Always retrieve"] --> C4["Always generate"]
        C5["'What is the capital of France?'\n↓ retrieves docs it doesn't need\n↓ wastes API calls + time"]
    end

    subgraph Agent["✅  ReAct Agent (Exp 7)"]
        A1["Question"] --> A2["LLM decides:\ndo I need to search?"]
        A2 -->|"YES — LLM topic"| A3["search_knowledge_base()"]
        A3 --> A4["Generate answer\nusing retrieved docs"]
        A2 -->|"NO — general knowledge"| A5["Answer directly\n(no retrieval needed)"]
    end

    style Chain fill:#fdecea,color:#000
    style Agent fill:#e8f5e9,color:#000
```

#### How the Tool Is Defined

```python
@tool
def search_knowledge_base(query: str) -> str:
    """Search for LLM topics: RAG, guardrails, observability, evals, fine-tuning."""
    docs = vectorstore.similarity_search(query, k=2)
    return "\n\n".join(f"[{d.metadata['topic']}] {d.page_content}" for d in docs)
```

No `create_retriever_tool`, no wrapper class — just a **plain Python function** with a docstring. The LLM reads the docstring to decide when to call it.

#### Trace Waterfall in Logfire

```mermaid
sequenceDiagram
    participant U as User Question
    participant A as ReAct Agent (Gemini)
    participant T as search_knowledge_base()
    participant LF as Logfire Dashboard

    U->>A: "What is LLM observability?"
    A->>LF: agent_run span starts
    A->>A: LLM thinks: "I should search"
    A->>T: calls search_knowledge_base()
    T->>LF: tool call span ← visible in waterfall
    T-->>A: returns 2 docs about Observability
    A->>A: LLM generates answer using docs
    A->>LF: agent_done (used_tool=True)

    Note over LF: Waterfall shows:\nagent_run\n  └─ Chat Completion [LLM]\n  └─ search_knowledge_base [TOOL]\n  └─ Chat Completion [LLM]
```

**Key insight:** The tool call span **only appears when retrieval actually happens**. For "What is the capital of France?" — no tool call span. This lets you audit your agent's reasoning from the dashboard.

---

## 🤖 The Streamlit App — Multi-Specialist AI Agent

This is a full **chat application** (`app.py`) powered by a **LangGraph multi-node agent**.

---

### Why Is It Called "Agentic"?

A regular chatbot sends every question to the same LLM with the same prompt. That's a **pipeline** — fixed, no decisions.

An **agent** is different:

```mermaid
graph LR
    subgraph Pipeline["❌  Regular Chatbot (fixed pipeline)"]
        P1["Question"] --> P2["Same prompt\nevery time"] --> P3["Same LLM\nevery time"] --> P4["Answer"]
        P5["'What is RAG?' and 'Latest GPT-5 news?'\nboth go through identical code"]
    end

    subgraph Agent["✅  This App (agentic)"]
        A1["Question"] --> A2["Planner\ndecides intent"]
        A2 -->|"explain"| A3["Explainer\nteacher prompt"]
        A2 -->|"analyze"| A4["Analyst\ncomparison prompt"]
        A2 -->|"create"| A5["Creator\nideas prompt"]
        A2 -->|"search"| A6["Web Searcher\nTavily + synthesis"]
        A3 & A4 & A5 & A6 --> A7["Formatter\npolishes output"]
    end

    style Pipeline fill:#fdecea,color:#000
    style Agent fill:#e8f5e9,color:#000
```

**What makes it agentic:**
- The **Planner decides** which path to take — the code doesn't hardcode which node runs
- Different questions get **different prompts, different tools, different reasoning styles**
- The graph can be extended with new specialist nodes without touching existing ones

---

### The Full Agent Flow

```mermaid
flowchart TD
    U(["💬 User types a question\nin the chat UI"])

    P["🧠 Planner Node\nGroq classifies the question\ninto one of 4 intents"]

    EX["🎓 Explainer\n'What is X?'\n'How does X work?'"]
    AN["🔍 Analyst\n'Compare X vs Y'\n'Pros and cons of X'"]
    CR["✨ Creator\n'Write me X'\n'Give me 5 ideas for X'"]
    WS["🌐 Web Searcher\n'Latest news on X'\n'Current state of X'"]

    FM["📝 Formatter Node\nGroq lightly polishes\nthe specialist's output"]

    OUT(["✅ Final Answer\nshown in chat\n+ Trace details expandable"])

    U --> P

    P -->|"intent = explain"| EX
    P -->|"intent = analyze"| AN
    P -->|"intent = create"| CR
    P -->|"intent = search"| WS

    EX --> FM
    AN --> FM
    CR --> FM
    WS --> FM
    FM --> OUT

    style P fill:#f0ad4e,color:#000
    style WS fill:#D6E4F0,color:#000
    style FM fill:#e8f5e9,color:#000
    style OUT fill:#2E75B6,color:#fff
```

---

### The Nodes — What Each One Does and Why It Exists

#### 🧠 Planner Node
**What it does:** Reads the question and outputs exactly one word: `explain`, `analyze`, `create`, or `search`.

**Why we built it:** Without a planner, you'd need to write `if/elif` chains in Python to route questions. The planner uses the LLM's understanding of language — it catches "elaborate on X" as `explain` and "draft X" as `create` without explicitly programming those synonyms.

```mermaid
flowchart LR
    Q["'Can you compare RAG\nand fine-tuning?'"] --> PL["Planner (Groq)\n\nSystem prompt:\n'Reply with ONE word:\nexplain / analyze / create / search'"]
    PL --> OUT["analyze"]
    OUT --> AN["→ Analyst node runs"]
```

**Trigger words:**

| Intent | Keywords it catches |
|--------|-------------------|
| `explain` | what is, how does, why, tell me about, describe, elaborate |
| `analyze` | compare, vs, pros and cons, evaluate, difference, tradeoffs |
| `create` | write, generate, brainstorm, create, ideas for, draft, give me |
| `search` | latest, current, news, today, recent, 2024, 2025, search for |

---

#### 🎓 Explainer Node
**What it does:** Answers "what is / how does" questions in teacher mode — simple language, analogies, bullet points.

**Why we built it:** A general LLM prompt gives generic answers. A specialist system prompt ("you are an expert teacher, use analogies, aim for 150-250 words") consistently produces cleaner educational output. Separating this from the analyst means you can tune each prompt independently without breaking the others.

```
Sample query  →  "What is the attention mechanism in transformers?"
Expected output  →  Bullet points with analogy, 150-250 words, beginner-friendly
```

---

#### 🔍 Analyst Node
**What it does:** Handles comparison and evaluation questions — structured markdown with headers, pros/cons tables, clear recommendations.

**Why we built it:** Comparisons need a different output format than explanations. If the planner routes "compare RAG vs fine-tuning" to the explainer, you get a paragraph. The analyst gives you a structured breakdown you can actually use to make decisions.

```
Sample query  →  "Compare RAG vs fine-tuning — pros, cons, when to use each"
Expected output  →  Markdown headers, comparison table, recommendation
```

---

#### ✨ Creator Node
**What it does:** Generates ideas, writes content, brainstorms — produces numbered lists of creative, specific, actionable items.

**Why we built it:** Creative tasks need a different mindset from factual tasks. The creator system prompt encourages novelty and specificity, while the explainer prompt encourages accuracy and clarity. Mixing them produces mediocre output for both.

```
Sample query  →  "Give me 5 ideas for an LLM security blog post"
Expected output  →  Numbered list, specific titles + 1-line description each
```

---

#### 🌐 Web Searcher Node
**What it does:** Calls the **Tavily search API** to fetch live web results, then uses Groq to synthesize them into a coherent answer with sources.

**Why we built it:** LLMs have a knowledge cutoff — they don't know about last week's OpenAI announcement. The web searcher is the only node that goes outside the model's training data. It has two internal sub-steps (search → synthesize) which both appear as child spans in Logfire.

```mermaid
flowchart LR
    Q["'Latest news on\nOpenAI models 2025?'"] --> TV["Tavily API\n(live web search)\n~900ms"]
    TV --> R["4 web results\nwith URLs"]
    R --> SY["Groq synthesizes\ninto clean answer\n~700ms"]
    SY --> ANS["Answer + sources"]
```

```
Sample query  →  "Latest developments in AI agents in 2025"
Expected output  →  Synthesized summary of recent articles, with source URLs
```

---

#### 📝 Formatter Node
**What it does:** Takes the specialist's output and lightly polishes it — fixes structure, improves readability — **without changing the content**.

**Why we built it:** Specialists are optimized for correctness and depth, not polish. A separate formatting pass means you can improve output quality without touching the specialist prompts. It also demonstrates the pattern of **chaining nodes** in LangGraph — each node has one responsibility.

---

### The 4 Routing Paths

| Intent | Triggered by | Sample query | Node path |
|--------|-------------|-------------|-----------|
| `explain` | what is, how does, why, describe | `"What is RAG?"` | planner → explainer → formatter |
| `analyze` | compare, vs, pros and cons | `"Compare RAG vs fine-tuning"` | planner → analyst → formatter |
| `create` | write, generate, brainstorm, ideas | `"Write 3 blog post ideas about LLM security"` | planner → creator → formatter |
| `search` | latest, current, news, 2025 | `"Latest news on OpenAI models 2025"` | planner → web_searcher → formatter |

---

### 🧪 Sample Queries to Run (Try All 4 Paths)

Copy-paste these into the chat to see every node in action:

```
# Hits EXPLAINER
What is the attention mechanism in transformers?
How does RAG reduce hallucinations?
Why do LLMs hallucinate?

# Hits ANALYST
Compare RAG vs fine-tuning — when should I use each?
Pros and cons of using an LLM gateway like Portkey
What are the tradeoffs between Groq and OpenAI for production?

# Hits CREATOR
Give me 5 ideas for an LLM security course
Brainstorm 3 names for an AI observability startup
Write a short LinkedIn post about why LLM observability matters

# Hits WEB SEARCHER
Latest news on OpenAI models 2025
What is the current state of AI agents in 2025?
Recent developments in LLM safety research
```

**After each query:** expand the "Trace details" section in the chat — it shows `intent`, `path`, and `model_used`. Then open Logfire to see the full waterfall.

---

### The Web Search Path (Deepest Waterfall)

```mermaid
sequenceDiagram
    participant U as User
    participant PL as Planner (Groq)
    participant WS as Web Searcher
    participant TV as Tavily API
    participant SY as Synthesizer (Groq)
    participant FM as Formatter (Groq)
    participant LF as Logfire

    U->>PL: "Latest news on GPT-5?"
    PL->>LF: planner span → intent=search
    PL->>WS: route to web_searcher
    WS->>TV: Tavily search(query)
    TV->>LF: tavily_search span (~900ms)
    TV-->>WS: 4 web results with URLs
    WS->>SY: Groq synthesizes results
    SY->>LF: synthesize_results span (~700ms)
    SY-->>FM: summary text
    FM->>LF: formatter span (~200ms)
    FM-->>U: polished final answer
```

```
Logfire waterfall for a search query:

agent_run                    ██████████████████████████  ~2.1s total
  planner                    ██                          ~280ms
  web_searcher               ██████████████████          ~1.6s
    tavily_search            ████████████                ~900ms  ← external API
    synthesize_results       ████████                    ~700ms
  formatter                  ████                        ~240ms
```

### What You See in the UI

```mermaid
graph LR
    subgraph Streamlit["Streamlit Chat UI"]
        Chat["💬 Chat messages\n(question + answer)"]
        Expand["▼ Trace details\n(expandable per message)"]
        Sidebar["Sidebar:\n• Session ID\n• Routing map\n• Example queries\n• Link to Logfire"]
    end

    subgraph TraceDetails["Trace details (JSON)"]
        Intent["intent: analyze"]
        Path["path: planner → analyst → formatter"]
        Model["model_used: groq/llama-3.3-70b → groq (formatter)"]
        Session["session_id: a3f7c2b1"]
    end

    Expand --> TraceDetails
```

### How the Agent Code Is Organized

```mermaid
graph TD
    APP["app.py\nStreamlit UI + session state"]

    subgraph Agent["agent/ package"]
        S["state.py\nAgentState TypedDict\n(question, intent, output, path…)"]
        T["tracing.py\nlogfire.configure()\nlogfire.instrument_openai()"]
        L["llms.py\ngroq_llm() factory"]
        N["nodes.py\nplanner · explainer · analyst\ncreator · web_searcher · formatter\n— each wrapped in logfire.span() —"]
        G["graph.py\nStateGraph wiring\nconditional routing logic"]
    end

    APP -->|"init()"| T
    APP -->|"build_graph()"| G
    G --> N
    N --> L
    N --> S

    style T fill:#f0ad4e,color:#000
    style N fill:#D6E4F0,color:#000
    style G fill:#e8f5e9,color:#000
```

**Each file has one job:**

| File | Does exactly one thing |
|------|----------------------|
| `state.py` | Defines the data shape that flows between nodes |
| `tracing.py` | Configures Logfire once at startup |
| `llms.py` | Creates LLM client objects |
| `nodes.py` | Business logic — each node is a function with a `logfire.span()` |
| `graph.py` | Wires the nodes together and defines routing rules |
| `app.py` | Handles the chat UI — calls the graph, displays results |

---

## 📁 File Structure

```
logifre observability/
│
├── pydantic_logfire.ipynb    ← 7 experiments (run top to bottom)
│
├── documents.json            ← knowledge base for RAG (6 LLM topic docs)
│
├── app.py                    ← Streamlit chat app (run: streamlit run app.py)
│
├── agent/
│   ├── state.py              ← AgentState — data flowing between nodes
│   ├── tracing.py            ← Logfire setup (called once)
│   ├── llms.py               ← Groq LLM factory
│   ├── nodes.py              ← planner, explainer, analyst, creator,
│   │                            web_searcher, formatter
│   └── graph.py              ← LangGraph StateGraph + routing
│
├── requirements.txt
└── .env                      ← LOGFIRE_TOKEN, GROQ_API_KEY, GEMINI_API_KEY
```

---

## 🚀 Running the App

```bash
# Run the notebook
# Open pydantic_logfire.ipynb in Jupyter and run cells top to bottom

# Run the Streamlit app
streamlit run app.py
```

**Test queries that hit all 4 routing paths:**

| Query | Route |
|-------|-------|
| `What is RAG?` | explain → 🎓 Explainer |
| `Compare RAG vs fine-tuning` | analyze → 🔍 Analyst |
| `Write 3 blog post ideas about LLM security` | create → ✨ Creator |
| `Latest news on OpenAI models 2025` | search → 🌐 Web Searcher |

---

## 📖 Key Concepts Quick Reference

| Term | Simple definition |
|------|------------------|
| **Span** | A named, timed block of code — like a stopwatch with labels |
| **Trace** | A tree of spans belonging to one request — visualized as a waterfall |
| **Attribute** | A key-value tag on a span (`user_id="alice"`) — searchable in dashboard |
| **Auto-instrumentation** | `logfire.instrument_openai()` — Logfire patches the SDK so every LLM call is traced automatically |
| **logfire.info()** | A structured log event — like `print()` but searchable and timestamped |
| **logfire.span()** | A timed context manager — measures everything inside it |
| **Token** | Basic unit of LLM input/output — roughly 0.75 words, billing is per token |
| **Embedding** | A list of numbers representing the meaning of text — similar texts have similar numbers |
| **FAISS** | A library that finds the most similar embeddings quickly (vector search) |
| **RAG** | Retrieve relevant docs → inject as context → generate grounded answer |
| **ReAct Agent** | An LLM that loops: Reason → Act (call tool) → Observe result → Reason again |
| **OpenTelemetry** | The open standard for distributed tracing — Logfire implements it |
| **Trace Waterfall** | The dashboard view where each span is a horizontal bar — width = duration |
