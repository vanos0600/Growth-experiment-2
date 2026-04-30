# Growth Decision Engine — Architecture

**Problem solved:** cut time-to-launch for growth experiments from weeks to hours by automating the two slowest steps — data analysis and content creation — while preventing the team from repeating expensive mistakes.

**Core design principle:** the intelligence lives in the prompts and the knowledge base, not in the code. A non-technical growth marketer can ship a better system by editing a markdown file. Engineers are not a bottleneck.

---

## 1. High-level architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         GROWTH DECISION ENGINE                        │
└──────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐                          ┌──────────────────┐
    │  MESSY       │                          │  CONTENT         │
    │  EXPERIMENT  │                          │  BRIEF           │
    │  DATA        │                          │  (segment +      │
    │  (paste)     │                          │   surface +      │
    └──────┬───────┘                          │   goal)          │
           │                                  └────────┬─────────┘
           ▼                                           ▼
    ┌──────────────┐                          ┌──────────────────┐
    │  ANALYST     │                          │  CONTENT         │
    │  AGENT       │                          │  AGENT           │
    │  (Gemini 2.5)│                          │  (Gemini 2.5)    │
    └──────┬───────┘                          └────────┬─────────┘
           │                                           │
           │    ┌──────────────────────────────┐      │
           └───▶│       RAG LAYER              │◀─────┘
                │  ──────────────────          │
                │  · Gemini embeddings         │
                │  · In-memory vector index    │
                │  · (prod: Vertex Vector DB)  │
                └──────────────┬───────────────┘
                               │ retrieves
                               ▼
                ┌──────────────────────────────┐
                │     KNOWLEDGE BASE           │
                │  ──────────────────          │
                │  · Conversion heuristics     │
                │  · Segmentation playbook     │
                │  · Past experiments log      │
                │  · Brand voice guidelines    │
                │  (editable markdown files)   │
                └──────────────────────────────┘

           ┌───────────────────────────────┐
           ▼                               ▼
    ┌──────────────┐              ┌────────────────┐
    │  REVENUE     │              │  LAUNCH-READY  │
    │  SIMULATOR   │              │  VARIANTS      │
    │  (pure math) │              │  + checklist   │
    │              │              │  + success KPI │
    │  12-mo $$    │              └────────────────┘
    │  LTV         │
    │  refund drag │
    │  churn drag  │
    └──────┬───────┘
           ▼
    ┌──────────────┐
    │  MANAGEMENT  │
    │  BRIEF       │
    └──────────────┘
```

---

## 2. Why RAG (and why it matters for conversion)

A plain LLM call "what should we do with this experiment?" gives generic advice. RAG lets the system reason from *our company's accumulated lessons*:

- **Past experiments** → "we tried fear framing on renewals; it cost us $38k/month. Don't repeat."
- **Segmentation playbook** → "power users convert on benefit copy at 4x baseline. Don't send them fear."
- **Brand voice** → "in DACH, urgency tactics depress conversion. Localize, don't translate."

This is how the system actually drives conversion: it stops the team from making the *same mistake twice* and it generates copy that is already aligned with what works.

The knowledge base is markdown. Growth marketers edit it directly. New learning goes in as soon as an experiment ends. The system gets smarter every week without an engineering ticket.

---

## 3. Module responsibilities

| Module | Lines | Job |
|--------|------:|-----|
| `core/rag_retriever.py` | ~140 | Chunk → embed → retrieve top-k with cosine similarity. Swappable for Vertex Vector Search. |
| `core/agents.py` | ~120 | Two thin Gemini wrappers: `AnalystAgent` and `ContentAgent`. Handles retrieval + JSON parsing. |
| `core/revenue_simulator.py` | ~70 | Pure Python math: 12-month revenue projection with refund & churn drag, cohort LTV. No ML. |
| `prompts/prompts.py` | ~90 | All Gemini prompts. The biggest quality lever in the system. |
| `knowledge_base/*.md` | ~250 | The company's accumulated lessons. Editable by non-engineers. |
| `main.py` | ~180 | CLI demo entrypoint. |

**Total: ~850 lines.** Small enough to read end-to-end in 20 minutes. Big enough to do real work.

---

## 4. Time-to-market impact (the headline number)

Current state (manual):

| Step | Time | Cost |
|------|------|------|
| Analyst pulls & cleans data | 2–4 h | analyst time |
| Analyst writes recommendation | 1–2 h | analyst time |
| Copywriter drafts 4 variants | 4–8 h | copywriter time |
| PM writes brief for leadership | 1 h | PM time |
| Review cycles | 1–2 days | calendar drag |
| **Total per experiment** | **~3 days** | 4 humans |

With this system:

| Step | Time | Cost |
|------|------|------|
| Paste data into analyst agent | 30 s | — |
| Review output, edit if needed | 10 min | analyst |
| Run content agent with brief | 30 s | — |
| Review variants, pick 2–3 to ship | 15 min | copywriter |
| Export management brief | 5 s | — |
| **Total per experiment** | **~30 min** | 2 humans, light review |

**~6x speedup.** At ~50 experiments/year for a Growth team, that's ~120 person-days returned to the team annually.

---

## 5. Scalability path (MVP → production)

The MVP runs on a single laptop. Nothing in the architecture prevents scaling:

| Concern | MVP | Production |
|---------|-----|-----------|
| Vector index | in-memory numpy | Vertex AI Vector Search / Pinecone |
| KB storage | markdown files | Git-backed CMS (Notion sync / Contentful) |
| Agent orchestration | sequential Python calls | LangGraph / Vertex AI Agent Builder |
| Observability | stdout | structured logs → BigQuery + Looker |
| Evals | manual review | golden set + Gemini-as-judge pipeline |
| Secrets | env var | Secret Manager / Vault |
| Deployment | local | Cloud Run + Cloud Scheduler |
| User access | CLI | React front-end (already built separately) |
| Cost control | per-call | batched embeddings + response caching |

The MVP keeps interfaces clean enough that each of these swaps is an isolated change. `rag_retriever.py` exposes `.retrieve(query, k)` — whether the index is numpy or Pinecone is invisible to the agents.

---

## 6. Cost profile

Back-of-envelope for Gemini 2.5 Flash (at ~$0.075 per 1M input tokens, ~$0.30 per 1M output tokens as of 2026):

- **Analyst run:** ~3k input (context + data), ~1k output → ~$0.0005
- **Content run:** ~2k input, ~1.5k output → ~$0.0006
- **Embedding a new KB doc** (~2k tokens): ~$0.00003

**Per experiment analyzed + new variants generated: ~$0.001.**

At 50 experiments/year: ~$0.05 in API costs. The system pays for itself in the first five minutes of saved analyst time.

---

## 7. Guardrails and what this system deliberately does NOT do

- **It does not decide on its own.** Every recommendation is reviewed by a human before rollout. The system's value is compressing the prep time, not replacing judgment.
- **It does not personalize copy per user.** That's a separate system (user-facing, real-time, with consent and privacy review). This system personalizes at the *segment* level — which is where the real conversion lift lives anyway.
- **It does not fire fear-framed copy.** The knowledge base and prompts both push against it. If a team member asks for fear framing, the system will flag the tradeoff before generating.
- **It does not claim statistical significance.** It flags when tests are underpowered and recommends significance checks as a next action.
- **It does not learn from user data without explicit logging.** Every decision saved to the Playbook is an explicit action, reviewable and reversible.

---

## 8. The 1-hour MVP vs. the full roadmap

**What exists now (MVP, buildable in ~1 hour as the case requires):**
- `main.py` demo
- RAG over markdown KB
- Analyst + Content agents
- Revenue simulator
- Management brief output

**What's next (post-internship roadmap):**
1. **Week 1–2:** wrap in a web UI (Streamlit first, then FastAPI + React)
2. **Week 3–4:** connect to the actual experiment data warehouse (Snowflake/BigQuery) — no more pasting
3. **Week 5–6:** auto-ingest new experiments into the KB after they end — close the loop
4. **Week 7–8:** add a Gemini-as-judge eval harness to regression-test the agents when prompts change
5. **Quarter 2:** real-time personalization layer that uses the generated variants as inputs

Every step is a clear win for the business and a clean extension of the MVP — no rewrites required.
