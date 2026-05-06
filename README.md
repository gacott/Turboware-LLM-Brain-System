# Brain — Autonomous Coding Memory Architecture

Special thanks to Guerin Green (https://www.facebook.com/guerin.green) and his NovCog Brain https://novcog.dev/ for the inspiration.

## Overview
This is the system we are using at Turboware for Everything. All employees, no matter
their job description, development, management, social media, etc, are using this system
for AI-based access and work. It applies to and works flawlessly for all of their jobs.
It's a true one-size-fits-all system.

Using this system allows us to GREATLY manage and reduce AI spend as a company. No more
$2k a week bills from Anthropic. Also, the results are outstanding! Good code comes faster
than it ever has, without a lot of iteration. The system balances cost and model, and as it is
right now, it is working perfectly. I can say this because the logs are filled with nothing 
but "success" messages, no "fails".

While we do have it running on a server, it could also run on a VPS, and because we are using
a Cloudflare tunnel, the system doesn't need a public IP address.

Many of us still use Claude (myself) included, as the primary agent/harness. But it's been tested
extensively with Roo code/Cline as well. It should work perfectly with opencode as well, although
I have not tested it

The brain is a continuously learning memory system that intercepts every coding model
request flowing through the LiteLLM gateway. It builds five kinds of memory —
code chunks, architectural summaries, problem/solution experiences, curated
skills, and agent personas — then retrieves relevant context on every subsequent
request. No per-client setup. Every model sees everything the team has learned.

```
                         ┌──────────────────────────────┐
                         │       LiteLLM Gateway        │
                         │     (smart-router proxy)     │
                         └──────────┬───────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
   claude-use wrapper         Smart-Router               Memory Pipeline
   (4 backends)          (complexity_router)         (5 loops, PostgreSQL)
        │
        ├── DeepSeek V4-Pro    (direct API)
        ├── DeepSeek V4-Flash  (direct API)
        ├── Kimi K2.6          (Moonshot API)
        └── LiteLLM Gateway    (smart-router, all models)
```

## Why This Exists

Before Brain, every coding session started cold. The model knew nothing about
our codebase, our conventions, or the problems we'd already solved. We repeated
the same debugging, re-explained the same architecture.

Brain fixes this by making the gateway — the single choke point every model
request passes through — the learning substrate. Every interaction becomes
training data. Every solved problem becomes a retrievable context. The system
gets smarter with every request, automatically.

With the persona system, Brain now also answers *who* should respond — injecting
an agent identity (domain expert, communication style, critical rules) that
shapes the model's behavior for the task at hand, not just what code it should
see.

## Models

The `claude-use` wrapper (`~/.local/bin/claude-use`) routes Claude Code through
four backends:

| Backend | Model | Provider | Role |
|---------|-------|----------|------|
| DeepSeek V4-Pro | deepseek-v4-pro | DeepSeek direct API | Explicit "use pro" override only — multi-turn deep reasoning |
| DeepSeek V4-Flash | deepseek-v4-flash | DeepSeek direct API | Default for all coding tiers (SIMPLE, MEDIUM, REASONING) — both models have thinking mode, Flash handles 90%+ at $0.14/$0.28 |
| Kimi K2.6 | kimi-k2.6 | Moonshot API | Explicit "use kimi" override only — creative/UI/websites |
| LiteLLM Gateway | smart-router | Local proxy | Tiered dispatch via complexity_router, all Brain context + learning |

### Smart-Router Tier Dispatch

The gateway's complexity_router classifies every request by scoring seven
dimensions (tokenCount, codePresence, reasoningMarkers, technicalTerms,
simpleIndicators, multiStepPatterns, questionComplexity) against configurable
thresholds:

| Tier | Model | Trigger |
|------|-------|---------|
| SIMPLE | deepseek-v4-flash | Score < 0.10 |
| MEDIUM | deepseek-v4-flash | Score 0.10–0.22 |
| COMPLEX | grok-4.3 | Score 0.22–0.42 |
| REASONING | deepseek-v4-pro | Score > 0.42 |

The `complex_reasoning` threshold was raised from 0.28 to 0.42 because a
single `reasoningMarkers` hit (weight 0.30) was routing everyday coding
to Pro. Now Pro requires both reasoning language AND strong technical
complexity — only genuinely hard multi-turn problems reach it.

### Why Kimi Is Explicit-Only

Creative keyword auto-detection was removed. Keywords like "frontend",
"animation", "modal", "prototype", "presentation", "polish", "beautiful"
appear constantly in normal coding conversations — routing them to Kimi
silently burned Moonshot quota on non-design tasks. Kimi is reserved for
explicit "use kimi to create a web page" so the user consciously opts into
Moonshot billing.

## The Five Loops

### 1. Retrieve Loop (pre-call)

Fires before every coding model request. Injects relevant memories into the
conversation as a system message.

**Pipeline (target: <250ms warm, <500ms budget):**

```
User query
  → Extract last user message + code blocks         5ms
  → Embed via qwen3-embedding (Ollama, 4096-dim)   170ms
  → Parallel: persona retrieval                      5ms
            + vector search (PGVector cosine)       10ms
            + full-text search (PostgreSQL ts_rank)   1ms
  → Merge, dedup by entry_id, recency boost           1ms
  → CrossEncoder rerank (ms-marco-MiniLM-L-6-v2)    50ms
  → Trim to top-5, edge expansion for neighbors      10ms
  → Append top-matching skills (independent search)   5ms
  → Format as system message (~1500 tokens)           5ms
                                                  ──────
                                                  ~262ms
```

**Graceful degradation:** If retrieval exceeds the 500ms timeout, the request
proceeds without context. The reranker is skipped on the first request after
LiteLLM restart (CrossEncoder loads from disk in ~5s cold). Errors in any stage
never block the user.

**Persona retrieval:** Runs in parallel with code/summary/experience search.
Uses domain-keyword-first matching: extracts domain-significant keywords from
the user query, matches against persona `domain_keywords` arrays via PostgreSQL
array overlap. Requires >=2 keyword overlap to trust the keyword path; falls
through to vector semantic search otherwise. Matched persona(s) are formatted
as an identity prefix and prepended to the system message — so the model adopts
the role of a domain expert before it sees any code context.

**Edge expansion:** After finding top results, follows `brain_edges`
relationships to pull related context — if we retrieve `brain_hook.py`, we also
get `brain_retriever.py` and `brain_formatter.py` because graph edges connect
them.

**Skill retrieval:** Skills are searched independently from code — they get
dedicated slots (configurable, default 2). This prevents denser code embeddings
from crowding skills out of the top-15 cutoff.

### 2. Capture Loop (post-call)

Fire-and-forget after every successful response. Decides if the output is worth
keeping, then stores it.

**Capture heuristics:**
- Code blocks with >=10 lines → stored as `brain_code` chunks
- Responses containing architectural insight markers ("design pattern", "best
  practice", "the reason is") → stored as `brain_summary`
- Also reads reasoning_content (thinking tokens) so reasoning-heavy models
  aren't excluded

**Access tracking:** Retrieved entries have `access_count` incremented and
`last_accessed_at` updated. This feeds recency boosts in future retrievals.

### 3. Learn Loop (post-call)

Detects problem-solving patterns. When a query contains markers like "error",
"bug", "fix", "why does", "how do I fix", and the response is substantive, the
interaction is stored as a `brain_experience` entry — problem, solution, and
outcome. These are retrievable when similar problems arise.

### 4. Skills Loop (batch ingestion)

Curated instructional modules from trusted repositories, ingested via
`brain_cli.py skill-ingest`. Each skill is a SKILL.md file with YAML frontmatter
(name, description) and a markdown body. Skills are embedded, stored in
`brain_skills`, and retrieved alongside code/summary/experience memories.

**Skill sources (24 skills from 6 repos):**

| Repository | Skills | Focus |
|-----------|--------|-------|
| mattpocock/skills | diagnose, improve-codebase-architecture, grill-with-docs, zoom-out | Engineering methodology |
| affaan-m/everything-claude-code | mcp-server-patterns, postgres-patterns, api-design, deployment-patterns, cost-aware-llm-pipeline, python-patterns, docker-patterns, security-review | System architecture |
| anthropics/skills | docx, pdf, pptx, xlsx | Document generation |
| obra/superpowers | finishing-a-development-branch, systematic-debugging, using-git-worktrees, subagent-driven-development, test-driven-development, brainstorming | Development workflow |
| yvgude/lean-ctx | lean-ctx | Context management |
| forrestchang/andrej-karpathy-skills | karpathy-guidelines | Behavioral guardrails — think first, simplicity, surgical edits, goal-driven |

### 5. Persona Loop (batch ingestion + runtime retrieval)

Agent personas from [msitarzewski/agency-agents](https://github.com/msitarzewski/agency-agents)
— 183 pre-written expert identities spanning 15 domains. Each persona is a
markdown file with YAML frontmatter (name, description, emoji, vibe) and
structured body sections (Identity & Memory, Critical Rules, Communication
Style, Core Capabilities, etc.).

**How personas differ from skills:** Skills describe *how* to do something
(methodology, patterns, checklists). Personas describe *who* is doing it —
identity, professional stance, rules, and communication style. A skill
says "follow this debugging checklist." A persona says "you are a senior SRE
who thinks in blast radius, error budgets, and root cause."

**Domain divisions (15):**

| Division | Count | Example Personas |
|----------|-------|-----------------|
| engineering | 28 | Senior Developer, Software Architect, Security Engineer, SRE |
| marketing | 28 | Content Creator, SEO Specialist, Social Media Strategist |
| specialized | 40 | Chief of Staff, Compliance Auditor, MCP Builder |
| game-development | 19 | Game Designer, Level Designer, Unity Architect |
| design | 8 | Brand Guardian, UX Architect, UI Designer |
| sales | 8 | Sales Engineer, Deal Strategist, Pipeline Analyst |
| paid-media | 7 | Paid Media Auditor, PPC Campaign Strategist |
| testing | 8 | Accessibility Auditor, API Tester, Performance Benchmarker |
| finance | 5 | Financial Analyst, Investment Researcher, Tax Strategist |
| academic | 5 | Anthropologist, Historian, Psychologist |
| product | 5 | Product Manager, Trend Researcher, Behavioral Nudge Engine |
| project-management | 6 | Senior Project Manager, Studio Producer, Jira Workflow Steward |
| support | 6 | Support Responder, Infrastructure Maintainer |
| spatial-computing | 6 | visionOS Spatial Engineer, XR Immersive Developer |
| strategy | 4 | Strategy playbooks and coordination |

Sub-domains for game engines (blender, godot, unity, unreal-engine,
roblox-studio) are treated as distinct divisions for finer-grained matching.

**Runtime retrieval:**
1. Extract domain-significant keywords from user query (filtering stop words)
2. Match against persona `domain_keywords` via PostgreSQL `&&` overlap
3. If best match has >=2 keyword overlap, use keyword results
4. Otherwise, fall through to vector semantic search
5. Format top 1-2 personas as system identity prefix (~600 token budget)
6. Prepend to conversation BEFORE code context (persona first, then code,
   then skills)

## Memory Tables

All in PostgreSQL with the pgvector extension.

### brain_code
Code chunks at function/class granularity.

| Column | Purpose |
|--------|---------|
| `repo_path` | Which codebase this belongs to |
| `file_path` | Source file relative to repo root |
| `language` | Python, TypeScript, Go, Rust, etc. |
| `symbol_name` | Function or class name |
| `symbol_type` | function, class, struct, interface, etc. |
| `content` | The actual source code |
| `summary` | One-line heuristic description |
| `embedding` | 4096-dim vector via qwen3-embedding |

### brain_summary
Architectural notes, design rationale, conventions.

| Column | Purpose |
|--------|---------|
| `title` | One-line topic |
| `content` | The insight or convention |
| `keywords` | Extracted for FTS matching |
| `embedding` | 4096-dim vector |

### brain_experience
Problems encountered and how they were solved.

| Column | Purpose |
|--------|---------|
| `context_type` | bug, error, architecture, performance, etc. |
| `problem` | What went wrong or what was needed |
| `solution` | What fixed it |
| `outcome` | Did it work? What was learned? |
| `embedding` | 4096-dim vector |

### brain_skills
Curated instructional modules — methodology, patterns, workflows.

| Column | Purpose |
|--------|---------|
| `name` | Unique skill identifier |
| `description` | One-line summary for semantic matching |
| `body` | Full markdown instruction set |
| `source_repo` | Origin repository |
| `source_url` | Raw URL fetched from |
| `embedding` | 4096-dim vector (name + description) |

### brain_personas
Agent identities for system prompt injection — domain expertise, professional
stance, and behavioral rules.

| Column | Purpose |
|--------|---------|
| `name` | Unique persona identifier ("Security Engineer") |
| `description` | One-line role summary |
| `division` | Domain category (engineering, marketing, product, etc.) |
| `identity` | Who-am-I section — role, personality, experience |
| `critical_rules` | Non-negotiable behavioral constraints |
| `communication_style` | Tone, framing, and interaction patterns |
| `full_body` | Complete source markdown |
| `domain_keywords` | TEXT[] — extracted keywords for overlap matching |
| `embedding` | 4096-dim vector (name + description + identity) |

Key design: `domain_keywords` uses PostgreSQL's GIN array index with the `&&`
(overlap) operator for sub-millisecond keyword matching. This is faster than
vector search and handles exact terminology matches that embeddings might miss.

### brain_edges
Directed relationships between memories — who calls whom, what depends on what.

| Column | Purpose |
|--------|---------|
| `source_entry_id` | Origin of the relationship |
| `target_entry_id` | Destination |
| `relation_type` | contains, calls, imports, rationale_for, etc. |
| `confidence` | 0.0-1.0, EXTRACTED vs INFERRED |

## Ingestion Paths

| Path | Source | Method | Granularity |
|------|--------|--------|-------------|
| Graphify (AST) | Local repositories | Tree-sitter parsing (25 languages) via graphify tool | Per-symbol: functions, classes, edges |
| Session Capture | Model responses (automatic) | Post-call heuristics: code blocks >=10 lines, architectural markers | Per-response: code, summaries, experiences |
| Skill Ingestion | GitHub SKILL.md repos (6) | YAML frontmatter parsing + embedding | Per-skill: methodology modules |
| Persona Ingestion | agency-agents repo (183 personas) | YAML frontmatter + section extraction + embedding | Per-persona: identity, rules, style |

### Path Details

### Path A: Graphify (AST-quality, bulk)

`brain_graphify.py` runs the [Graphify](https://github.com/safishamsi/graphify)
tool, which uses tree-sitter to parse 25 languages into a knowledge graph
(208 nodes, 316 edges for the gateway codebase). Brain then:

1. Extracts code at AST boundaries (function/class bodies, not regex)
2. Stores design rationale from docstrings as `brain_summary`
3. Stores `contains`, `calls`, and `rationale_for` edges in `brain_edges`

```
brain_cli.py graphify /path/to/repo
```

### Path B: Session Capture (continuous, automatic)

Every response from a coding model through the gateway is evaluated by
`brain_ingest.capture_if_valuable()`. Code blocks are chunked and stored.
Architectural insights are extracted and stored. This happens automatically —
no manual ingestion needed.

### Path C: Skill Ingestion (curated, batch)

Skills are ingested from GitHub repositories via `brain_cli.py skill-ingest`.
Each skill is a SKILL.md file fetched from a raw URL, parsed for YAML
frontmatter, embedded, and stored in `brain_skills`. Skills are versioned
— re-running the ingest updates existing skills in place.

```
brain_cli.py skill-ingest
```

### Path D: Persona Ingestion (curated, batch)

Personas are ingested from `msitarzewski/agency-agents` via `brain_cli.py
persona-ingest`. The ingester fetches the repo tree from GitHub's API, then
fetches, parses, embeds, and stores each `.md` agent file. Personas are
versioned like skills — re-running the ingest updates existing personas in
place via ON CONFLICT DO UPDATE.

```
brain_cli.py persona-ingest
```

## Search Architecture

Hybrid search combining four strategies:

| Strategy | Engine | Latency | Role |
|----------|--------|---------|------|
| Vector (semantic) | PGVector cosine distance, 4096-dim | ~10ms | Finds conceptually similar code/patterns |
| Full-text (keyword) | PostgreSQL `ts_rank` on `to_tsvector` | ~1ms | Catches exact symbol names and error messages |
| Persona domain-keyword | PostgreSQL `&&` array overlap with GIN index | <1ms | Exact terminology match for persona retrieval |
| CrossEncoder rerank | ms-marco-MiniLM-L-6-v2 (CPU) | ~50ms | Fine-grained relevance scoring on top 15 candidates |

**Vector search (semantic):** Query → embedding → cosine distance against all
tables in parallel → top-10 per table with table-specific weights
(brain_code 1.0, brain_summary 0.9, brain_experience 0.8). Skills are searched
independently with their own top-k to prevent denser code embeddings from
crowding them out.

**Full-text search (keyword):** Query → PostgreSQL `plainto_tsquery` with
`ts_rank` scoring → exact keyword matches. Weights lower than vector (0.6) but
catches exact symbol names and error messages that vectors miss.

**Persona domain-keyword search (array overlap):** Query → extract domain
keywords → PostgreSQL `&&` (array overlap) against `domain_keywords` column with
GIN index → ordered by overlap count. Sub-millisecond. Falls through to vector
search if best overlap < 2 keywords. Only searched for persona retrieval, not
code/summary/experience.

**Merge + rerank:** Results deduplicated by `entry_id`, vector results preferred
over FTS (higher weight). Top 15 candidates passed through a CrossEncoder
(ms-marco-MiniLM-L-6-v2) for fine-grained relevance scoring. Final top-5
injected as context. Skills appended in dedicated slots after reranking.
Persona identity prepended before everything.

## Infrastructure

| Component | Details |
|-----------|---------|
| LiteLLM Gateway | Team proxy with smart-router, all model traffic |
| PostgreSQL + PGVector | Brain memory tables (6) + LiteLLM metadata. GIN indexes on FTS, B-tree on recency, GIN on domain_keywords |
| qwen3-embedding (Ollama) | 4096-dim, CPU, keep-alive 24h. LRU cache (100 entries) on query embeddings |
| CrossEncoder reranker | ms-marco-MiniLM-L-6-v2, in-process. Background thread warm on startup |
| graphify (tree-sitter) | 25 language grammars. AST-quality chunking for bulk ingestion |
| Local models (llama.cpp) | Qwen3-Coder-Next (80B-A3B), Gemma-4-31B. Free GPU inference |

### Embedding

`brain_embedder.py` wraps Ollama's HTTP API for `qwen3-embedding:latest`
(4096-dim). Async via `httpx.AsyncClient` with persistent connections. LRU
cache (100 entries) on query embeddings — identical queries skip the Ollama
round-trip. Batch embedding for ingestion (up to 2000 chars per chunk).

### Database

PGVector extension on the same PostgreSQL instance the gateway uses. Connection
pooling via `asyncpg` (5-15 connections). Recursive CTE for graph traversal
(`get_neighbors` with `max_depth`).

## How It All Fits Together

A developer using Claude Code types: "I need to refactor the auth middleware
to support JWT rotation."

```
┌─ Client ─────────────────────────────────────────────────────────┐
│ "refactor auth middleware to support JWT rotation"               │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌─ Smart-Router ───────────────────────────────────────────────────┐
│ Classifies: COMPLEX (refactor, middleware, auth, rotation)       │
│ Routes to: grok-4.3                                              │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌─ Brain Retrieve ─────────────────────────────────────────────────┐
│ Persona: extracts keywords (refactor, auth, middleware, jwt...)  │
│   → keyword match: Security Engineer (engineering, overlap >=2)  │
│   → injected as identity prefix                                  │
│                                                                  │
│ Vector search finds: auth_middleware.py (brain_code),            │
│ JWT session patterns (brain_summary),                            │
│ "JWT refresh token race condition" (brain_experience)            │
│                                                                  │
│ Edge expansion pulls: token_store.py, session_manager.py         │
│                                                                  │
│ Skill search finds: security-review, api-design                  │
│                                                                  │
│ All injected as system message before the model sees the query.  │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌─ Model ──────────────────────────────────────────────────────────┐
│ Sees: [You are Security Engineer...] + [System Context] + query  │
│ - Adopts security engineering stance and methodology             │
│ - The actual auth middleware code                                │
│ - The JWT session patterns we already documented                 │
│ - The race condition bug we previously fixed                     │
│ - The security-review and api-design skill methodologies         │
│                                                                  │
│ Generates a solution informed by security expertise, our code,   │
│ our conventions, our past mistakes, and established patterns.    │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌─ Brain Capture ──────────────────────────────────────────────────┐
│ Response contains new code → stored as brain_code chunks         │
│ Architectural insight about JWT rotation → stored as summary     │
│ Problem/solution pattern → stored as experience                  │
│                                                                  │
│ Next developer to touch auth gets ALL of this context for free.  │
└──────────────────────────────────────────────────────────────────┘
```

The developer didn't invoke any skills. They didn't search the codebase. They
didn't read past PRs or bug reports. They just asked a question, and Brain
assembled everything — the expert identity, the code, the architecture, the
experience, and the methodology — and presented it to the model.

## Performance

**Steady-state retrieval:** ~260ms (embed 170ms + persona 5ms + search 11ms +
rerank 50ms + overhead). Well within the 500ms budget. Persona retrieval adds
<5ms (keyword match <1ms, vector fallback <5ms, format <1ms).

**Cold start:** First request after gateway restart pays ~5s for CrossEncoder
loading (background thread) and ~14s if the embedding model was evicted from
Ollama memory. Both gracefully degrade — the request proceeds without context
rather than blocking.

**Current state (2026-05-06):** 461 code chunks, 403 summaries, 0 experiences,
24 skills from 6 repos, 183 personas from 1 repo, 351 graph edges. Growing with
every coding request.

## Key Design Decisions

1. **Hook-based, not API-based.** Brain lives inside the gateway as a hook, not
   as a separate service. No network hop, no additional latency, no extra
   process to manage.

2. **PGVector, not LanceDB or Pinecone.** Same PostgreSQL instance the gateway
   already uses. Native FTS via `tsvector`. No additional infrastructure.

3. **Fire-and-forget post-call.** `asyncio.create_task()` launches capture and
   learning in the background. The user never waits for storage.

4. **Graceful degradation everywhere.** Any error in retrieval, capture, or
   learning is caught silently. Brain can fail completely and no request is
   blocked.

5. **AST chunking via Graphify, not regex.** Tree-sitter properly handles nested
   classes, closures, async functions, decorators across 25 languages. Regex
   was a pragmatic first step; Graphify replaced it.

6. **Edge-weighted retrieval.** Graph relationships (contains, calls, imports)
   complement vector similarity. Pure vector search finds "similar code"; edges
   find "connected code."

7. **Team-wide, not per-user.** Brain is a centralized gateway — everyone
   benefits from everyone else's sessions. No local state, no sync needed.

8. **Skills as independent retrieval target.** Skills aren't lumped into the
   unified vector search where they'd be crowded out by denser code embeddings.
   They get their own vector search, their own top-k, and dedicated slots in the
   formatted context.

9. **Tiered model routing with gated Pro.** Flash handles SIMPLE, MEDIUM, and
   REASONING (both models have thinking mode — chain-of-thought with
   reasoning_content). Grok handles COMPLEX (architecture, refactors). Pro is
   gated behind a high `complex_reasoning` threshold (0.42) so only requests
   with BOTH reasoning language AND strong technical complexity reach it.
   Explicit "use pro" bypasses the gate. Every tier gets brain context. Every
   tier learns.

10. **Explicit model overrides only — no automatic detection.** The
    router_classifier hook intercepts requests before the complexity router
    to handle natural-language model overrides ("use kimi to create a web
    page", "use pro"). Creative keyword auto-detection was removed: keywords
    like "frontend", "animation", "modal", "prototype", "polish" appeared
    constantly in normal coding conversations, silently burning Moonshot
    quota. Now Kimi and Pro are opt-in — the user consciously chooses them.

11. **Never cripple model capability.** When a model supports thinking/reasoning,
    tool calling, or vision, those capabilities must survive the round-trip
    through the gateway intact. If a translation path drops reasoning content,
    discards tool arguments, or strips thinking blocks, the fix goes in the
    translation layer — not by disabling the capability on the model side.

12. **Persona as identity prefix, not context block.** Personas aren't mixed into
    the code/search context. They're prepended as a system identity ("You are X,
    operating in the Y domain") before any code or skills. This shapes the
    model's professional stance BEFORE it reads the task — so it approaches the
    work as a domain expert, not as a generic assistant that happens to have
    seen relevant code.

13. **Domain-keyword-first with semantic fallback.** Persona matching uses
    PostgreSQL array overlap (`&&` operator with GIN index) on extracted domain
    keywords. This is sub-millisecond and handles exact terminology matches
    ("game level design" → Game Designer, Level Designer). When the keyword
    signal is weak (<2 overlap), the system falls through to vector semantic
    search. The threshold prevents false matches from coincidental single-word
    overlaps.

## Recreating This System

### Prerequisites

- **PostgreSQL 15+** with the [pgvector](https://github.com/pgvector/pgvector) extension
- **Python 3.11+** virtual environment
- **Ollama** with `qwen3-embedding:latest` pulled
- **LiteLLM** proxy installed and running
- **git** and **tree-sitter** grammars (for graphify AST chunking)

### 1. Database

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

The schema is self-managing — `brain_db.py` creates all six tables on first
start via `init_schema()`. Set the connection string:

```bash
export DATABASE_URL="postgresql://user:pass@localhost:5432/litellm"
```

### 2. Embedding Model

```bash
ollama pull qwen3-embedding:latest
# Keep the model warm — avoids ~14s cold-start on first retrieval:
export OLLAMA_KEEP_ALIVE=24h
```

### 3. Python Dependencies

```bash
pip install asyncpg httpx sentence-transformers pyyaml
```

sentence-transformers provides the CrossEncoder reranker
(`cross-encoder/ms-marco-MiniLM-L-6-v2`). It loads lazily in a background
thread — the first request skips reranking rather than blocking.

### 4. Brain Files

All brain modules live alongside the LiteLLM config. Copy these files into
your LiteLLM directory:

```
brain_config.py          # All settings via env vars
brain_db.py              # PGVector schema, CRUD, hybrid search
brain_embedder.py        # Ollama embedding API wrapper
brain_retriever.py       # Multi-stage retrieval pipeline
brain_formatter.py       # System message formatting
brain_hook.py            # LiteLLM CustomLogger (pre-call + post-call)
brain_ingest.py          # Session capture heuristics
brain_learn.py           # Problem/solution pattern detection
brain_graphify.py        # Tree-sitter AST ingestion wrapper
brain_skill_ingester.py  # SKILL.md fetcher/parser from GitHub
brain_persona_ingester.py # Agency agent persona fetcher/parser
brain_cli.py             # CLI: graphify, skill-ingest, persona-ingest, etc.
router_classifier.py     # Model override hook ("use kimi", "use pro")
cf_streaming_middleware.py # SSE heartbeat prevents Cloudflare 524 timeouts
```

### 5. LiteLLM Callback Wiring

In your `config.yaml`:

```yaml
litellm_settings:
  callbacks:
    - router_classifier.proxy_handler_instance
    - brain_hook.proxy_handler_instance
```

The `router_classifier` must run first — it handles explicit model overrides
("use kimi", "use pro") before the complexity router dispatches.

### 6. Smart-Router Configuration

Add to `config.yaml` under `model_list`:

```yaml
- model_name: smart-router
  litellm_params:
    model: auto_router/complexity_router
    complexity_router_config:
      tiers:
        SIMPLE: deepseek-v4-flash
        MEDIUM: deepseek-v4-flash
        COMPLEX: grok-4.3
        REASONING: deepseek-v4-pro
      dimension_weights:
        tokenCount: 0.10
        codePresence: 0.20
        reasoningMarkers: 0.30
        technicalTerms: 0.25
        simpleIndicators: 0.05
        multiStepPatterns: 0.03
        questionComplexity: 0.02
      tier_boundaries:
        simple_medium: 0.10
        medium_complex: 0.22
        complex_reasoning: 0.42   # Pro gated behind this
      technical_keywords:
        - "architecture"
        - "distributed"
        - "scalable"
        - "refactor"
        - "migration"
        # ... (see full list in config)
      reasoning_keywords:
        - "step by step"
        - "think through"
        - "prove"
        - "theorem"
        # ... (see full list in config)
    complexity_router_default_model: deepseek-v4-flash
```

The `complex_reasoning` boundary at 0.42 is critical: `reasoningMarkers` alone
(weight 0.30) cannot reach Pro. The request needs BOTH reasoning language AND
strong technical complexity.

### 7. Persona Ingestion

Fetches 183 agent personas from
[msitarzewski/agency-agents](https://github.com/msitarzewski/agency-agents):

```bash
python brain_cli.py persona-ingest
```

This clones the repo tree via the GitHub API, then fetches, parses, embeds,
and stores each `.md` agent file. Personas are versioned — re-running updates
existing ones in place.

Verify:
```bash
python brain_cli.py persona-list
```

### 8. Skill Ingestion

Fetches 24 SKILL.md files from 6 repositories:

| Repository | Skills |
|-----------|--------|
| [mattpocock/skills](https://github.com/mattpocock/skills) | diagnose, improve-codebase-architecture, grill-with-docs, zoom-out |
| [affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code) | mcp-server-patterns, postgres-patterns, api-design, deployment-patterns, cost-aware-llm-pipeline, python-patterns, docker-patterns, security-review |
| [anthropics/skills](https://github.com/anthropics/skills) | docx, pdf, pptx, xlsx |
| [obra/superpowers](https://github.com/obra/superpowers) | finishing-a-development-branch, systematic-debugging, using-git-worktrees, subagent-driven-development, test-driven-development, brainstorming |
| [yvgude/lean-ctx](https://github.com/yvgude/lean-ctx) | lean-ctx |
| [forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills) | karpathy-guidelines |

```bash
python brain_cli.py skill-ingest
```

Verify:
```bash
python brain_cli.py skill-list
```

### 9. Code Ingestion (Graphify)

Parses a repository into AST-quality chunks using tree-sitter (25 languages)
via the [Graphify](https://github.com/safishamsi/graphify) tool:

```bash
python brain_cli.py graphify /path/to/repo
```

This extracts functions, classes, methods at AST boundaries and stores
`contains`, `calls`, and `rationale_for` edges in `brain_edges`.

### 10. Verification

After setup, start LiteLLM and check the logs:

```bash
systemctl restart litellm
journalctl -u litellm -f | grep -E 'brain|router_classifier'
```

You should see the reranker loading in the background (first request skips
reranking), persona and context retrieval firing on each coding request, and
capture/learn running fire-and-forget after each response.

Cold-start timeline:
- Reranker loads in background thread (~5s) — requests proceed without reranking until ready
- Embedding model pulls from Ollama (~14s if evicted) — requests proceed without context
- PostgreSQL schema auto-creates on first brain hook initialization

### 11. Cloudflare Tunnel (cloudflared)

The gateway is exposed publicly through a Cloudflare Tunnel, which proxies
traffic from a public hostname to `localhost:4000` without opening firewall
ports. This also provides DDoS protection and free TLS termination.

**Install and authenticate:**

```bash
# Install cloudflared
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo gpg --dearmor -o /usr/share/keyrings/cloudflare-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/cloudflare-archive-keyring.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install cloudflared

# Authenticate (opens browser — or use a token from the Cloudflare dashboard)
cloudflared tunnel login
cloudflared tunnel create <tunnel-name>
```

**Configure** (`/etc/cloudflared/config.yml`):

```yaml
tunnel: <tunnel-id>
credentials-file: /etc/cloudflared/<tunnel-id>.json

ingress:
  - hostname: <your-public-hostname>
    service: http://localhost:4000
    originRequest:
      connectTimeout: 30s
      disableChunkedEncoding: false
  - service: http_status:404
```

**DNS:** Create a CNAME record in Cloudflare DNS pointing
`<your-public-hostname>` to `<tunnel-id>.cfargotunnel.com`.

```bash
sudo systemctl enable --now cloudflared
```

### 12. Streaming Heartbeat Middleware (CF 524 Fix)

DeepSeek V4-Pro and other reasoning models can spend >100s in their thinking
phase before emitting the first token. Cloudflare's proxy has a hard 100-second
read timeout — if no bytes flow, it terminates with a **524 error**.

The `cf_streaming_middleware.py` ASGI middleware fixes this by injecting SSE
heartbeat comments (`: heartbeat\n\n`) every 20 seconds during silent periods
in streaming responses. SSE parsers ignore comment lines (they start with `:`).

**Mount the middleware** in your LiteLLM startup script (before uvicorn):

```python
from cf_streaming_middleware import CFStreamingMiddleware
app.add_middleware(CFStreamingMiddleware)
```

It only intercepts POST requests to streaming paths (`/v1/chat/completions`,
`/v1/messages`). All other traffic passes through at wire speed with zero
overhead — one lock acquire/release per response chunk, no request body
inspection.
