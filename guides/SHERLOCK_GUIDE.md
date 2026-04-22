# Sherlock — Complete Intern Guide

> **Who this is for:** New team members with no prior knowledge of knowledge graphs, AI pipelines, or observability systems. This document explains Sherlock from scratch — what it is, why it exists, how it works, and how to get started.

---

## Table of Contents

1. [What Is Sherlock? (Plain English)](#1-what-is-sherlock-plain-english)
2. [Why Does Sherlock Exist? The Problem It Solves](#2-why-does-sherlock-exist-the-problem-it-solves)
3. [How Sherlock Fits Into Testsigma's Main Product](#3-how-sherlock-fits-into-testsigmas-main-product)
4. [The Core Idea — A Knowledge Graph](#4-the-core-idea--a-knowledge-graph)
5. [How Sherlock Works — The Full Pipeline](#5-how-sherlock-works--the-full-pipeline)
6. [Versions: v2 vs v3 — What Changed and Why](#6-versions-v2-vs-v3--what-changed-and-why)
7. [Architecture: The Three-Layer System](#7-architecture-the-three-layer-system)
8. [The Planner Agent — Using Sherlock for Test Planning](#8-the-planner-agent--using-sherlock-for-test-planning)
9. [Technology Stack — What Each Tool Does](#9-technology-stack--what-each-tool-does)
10. [Directory Structure Walkthrough](#10-directory-structure-walkthrough)
11. [How to Get Started Locally](#11-how-to-get-started-locally)
12. [Key Concepts You Must Know](#12-key-concepts-you-must-know)
13. [Recent Changes and Latest Updates](#13-recent-changes-and-latest-updates)
14. [Glossary](#14-glossary)

---

## 1. What Is Sherlock? (Plain English)

**Sherlock is an AI system that watches screen recordings of users using a software application and builds a map of how the application works.**

Think of it exactly like the famous detective Sherlock Holmes — given only observations (user sessions recorded as video + browser events), it figures out the full picture of the application: what screens exist, how users navigate between them, what the critical user journeys are, and how different parts of the app are related.

The output is a **knowledge graph** — a structured, queryable database of:
- Every unique **screen** in the application
- Every **transition** between screens (what button was clicked, what happened)
- Every **user workflow** (the full journey to achieve a goal, e.g., "Complete a Purchase")
- Every **feature** (a logical grouping of screens, like "Checkout" or "User Management")

Once this knowledge exists, it can answer questions like:
- *"What screens exist in the checkout feature?"*
- *"What's the shortest path from Login to Order Confirmation?"*
- *"If we change the Payment page, what else might break?"*
- *"What should we test when we release this new feature?"*

---

## 2. Why Does Sherlock Exist? The Problem It Solves

### The Core Problem

When a QA team wants to test a software application, they face a fundamental challenge: **they don't have a complete map of the application**.

Documentation is often outdated. The actual application behaviour only exists in the live app. To write good tests, you need to know:
- What screens exist
- How to get from one screen to another
- What user journeys are "critical" (if they break, the product fails)
- What parts of the app are connected to each other

Without this knowledge:
- Tests miss important screens
- Regression testing is guesswork ("I hope I picked the right tests")
- When something changes, nobody knows what else might break

### The Traditional Approach (and Why It Fails)

QA teams traditionally build this knowledge manually — someone sits down and writes documentation, maps out user flows in spreadsheets, and decides which tests are critical. This:
- Takes weeks
- Goes out of date immediately
- Depends on people who may leave the company
- Misses edge cases humans don't think of

### Sherlock's Approach

Instead of manual documentation, Sherlock **learns from observation**. Every time a user records a session (even just using the app normally), Sherlock:
1. Watches the recording
2. Identifies what screens appeared
3. Understands what the user was trying to do
4. Adds this knowledge to the graph

The more recordings, the more complete the map. The system is **self-building** — it gets smarter over time without anyone manually writing documentation.

---

## 3. How Sherlock Fits Into Testsigma's Main Product

Sherlock is **the brain that makes Atto smarter**.

### Without Sherlock

Atto (the test generation agent) generates tests based on:
- User stories ("Add Apple Pay to checkout")
- Documentation
- Existing test cases

This is like asking someone to write a test for a house they've never visited — they can write something generic, but might miss the hidden staircase or the broken window.

### With Sherlock

Atto can now ask Sherlock:
- *"What screens exist in the Checkout feature?"*
- *"What user workflows already exist for Payment?"*
- *"What other features might be affected by a Payment change?"*
- *"What are the actual navigation steps to reach the Payment screen?"*

This transforms Atto from a generic test writer to one with **actual knowledge of your specific application**.

### The Integration

```
User Story: "Add Apple Pay as a payment option"
         │
         ▼
    Atto Agent
         │
         │  Queries Sherlock:
         │  ┌─────────────────────────────────────────┐
         │  │ find_similar_flows("Complete payment")   │
         │  │ get_feature_details("feat_checkout")     │
         │  │ analyze_impact("payment option change")  │
         │  │ get_critical_flows("feat_checkout")      │
         │  └─────────────────────────────────────────┘
         │
         │  Sherlock returns:
         │  ─ Checkout has 5 screens: Cart → Shipping → Payment → Review → Confirm
         │  ─ Credit Card and PayPal workflows already exist
         │  ─ Payment is connected to: Order History, Subscriptions, Refunds
         │  ─ Critical flows: Complete Purchase, Process Refund
         │
         ▼
    Generated Tests (much better):
    1. Add Apple Pay option (new)
    2. Complete purchase with Apple Pay (new)
    3. Verify Credit Card still works (regression — Apple Pay shouldn't break this)
    4. Apply coupon + Apple Pay combination (edge case discovered from existing flows)
    5. Verify order confirmation shows Apple Pay method (real screen path used)
```

Without Sherlock, Atto might only generate tests 1 and 2 and miss the regression tests entirely.

### What Sherlock Can Answer for Atto

| Question | Sherlock's Answer |
|----------|------------------|
| What screens exist in this feature? | All `PageState` nodes with `feature_id` |
| What workflows already exist? | `KnowledgeNode` list — avoid duplicating tests |
| What's the actual navigation path? | `ActionEdge` chain from source to destination |
| What else might break? | Related features, shared screens |
| What should the smoke test include? | Flows marked `is_critical_path = true` |
| What edge cases exist? | All observed variations of a workflow |

---

## 4. The Core Idea — A Knowledge Graph

A **knowledge graph** is a data structure that stores information as **nodes** (things) and **edges** (relationships between things). Sherlock's graph has three main types of nodes:

### Node Type 1: PageState (a Screen)

Represents one unique screen in the application.

```
PageState: "User Profile Edit Page"
├── screen_name: "User Profile Edit Page"
├── screen_purpose: "Allows users to edit their profile information"
├── key_elements: ["email field", "name field", "avatar upload", "save button"]
├── route_pattern: "/users/{id}/edit"
├── structural_signature: "a3f7b2c1..."  (fingerprint of HTML structure)
└── occurrence_count: 47               (seen 47 times across all recordings)
```

### Node Type 2: ActionEdge (a Transition)

Represents a user action that moves from one screen to another.

```
ActionEdge: User Profile → Settings Page
├── source_id: "state_profile_123"
├── target_id: "state_settings_456"
├── action_type: "click"
├── selector: "#save-settings-btn"
└── is_parametric: false
```

### Node Type 3: KnowledgeNode (a Workflow)

Represents a complete user goal — the full sequence of screens and actions to achieve something.

```
KnowledgeNode: "Complete a Purchase"
├── goal_text: "User completes a purchase with items in cart"
├── anchor_state_ids: ["state_cart", "state_shipping", "state_payment", "state_confirm"]
├── steps: [
│     "Navigate to cart",
│     "Proceed to checkout",
│     "Fill shipping address",
│     "Select payment method",
│     "Place order",
│     "See order confirmation"
│   ]
├── is_critical_path: true
├── is_smoke_test: true
└── occurrence_count: 312
```

### Node Type 4: Feature (a Logical Grouping)

Groups related screens and workflows together.

```
Feature: "Checkout"
├── name: "Checkout"
├── description: "All screens and flows related to the purchase process"
├── is_critical: true
├── state_count: 5
└── flow_count: 8
```

### How They Relate

```
Feature ──contains──▶ KnowledgeNode ──anchors──▶ PageState
    │                                                 │
    └──────────────spans──────────────────────────────┘

PageState ◀───────── ActionEdge ────────▶ PageState
```

---

## 5. How Sherlock Works — The Full Pipeline

When a screen recording is fed to Sherlock, it goes through a **7-step AI pipeline**. Each step passes its results to the next, so context accumulates throughout.

### Input Format

A recording consists of:
```
recording/
├── session.webm          ← Video of the user's screen
├── tab-events.json       ← Browser event log (clicks, typing, navigation)
└── pagesources/          ← HTML snapshots of pages visited
    ├── page_001.html
    └── page_002.html
```

### The 7 Steps

```
Recording (video + events)
        │
        ▼
┌─────────────────────────┐
│  Step 1: Load Recording  │  Parse events, validate video, measure duration
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  Step 2: Screen Scanner  │  Gemini watches the full video and identifies
│  Agent (Gemini vision)   │  every unique screen: name, purpose, key elements
└───────────┬─────────────┘
            │ "I see: Login page, Dashboard, Product List, Cart, Checkout..."
            ▼
┌────────────────────────────┐
│  Step 2b: Element          │  LLM-named element labels are merged into
│  Annotation Enrichment     │  the browser events (enriches click/fill records)
└───────────┬────────────────┘
            │
            ▼
┌─────────────────────────┐
│  Step 3: Segmentation    │  Identifies episode boundaries — splits the
│  Agent                   │  recording into separate tasks
└───────────┬─────────────┘
            │ "Events 0-45 = login task; Events 46-120 = product search; ..."
            ▼
┌─────────────────────────┐
│  Step 4: Goal Extraction │  HER Agent: For each episode segment, extracts
│  Agent (HER)             │  the user's goal and the outcome
└───────────┬─────────────┘
            │ "Episode 1 goal: 'Log in as admin user' → Outcome: Success"
            ▼
┌─────────────────────────────────────────────────┐
│  Step 5: Graph Merger Agent                      │
│  (Read-only access to existing graph + Qdrant)   │
│                                                  │
│  Tools available:                                │
│  • find_similar_flows(goal_text)                 │
│  • get_flow_anchors(flow_id)                     │
│  • find_states(route_pattern, name)              │
│  • get_state_details(state_id)                   │
│                                                  │
│  Output: MergePlan (structured JSON instructions)│
└───────────┬─────────────────────────────────────┘
            │ "Reuse existing 'login' state; Create new 'product search' state; ..."
            ▼
┌─────────────────────────┐
│  Step 6: Plan Executor   │  Reads MergePlan and actually writes to the graph:
│  (deterministic)         │  creates/updates PageStates, ActionEdges, KnowledgeNodes
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  Step 7: Feature         │  Classifies newly added screens and workflows
│  Classifier Agent        │  into feature groups (Checkout, Auth, etc.)
└─────────────────────────┘
```

### Why Split Agent and Executor?

A key design decision: the **Graph Merger Agent** does NOT directly write to the database. Instead it produces a `MergePlan` — a structured JSON document describing what should happen.

The **Plan Executor** then reads that plan and writes to the database deterministically (no AI involved).

**Why?** This makes the system:
- **Auditable**: You can inspect the plan before it's executed
- **Recoverable**: If execution fails, the plan still exists
- **Testable**: You can unit-test the executor without needing an LLM
- **Safe**: The LLM cannot make arbitrary database mutations

---

## 6. Versions: v2 vs v3 — What Changed and Why

Sherlock is currently on version 3 (v3). Understanding why it changed helps you understand the core technical challenge.

### v2 — The DOM Signature Approach

**How it worked:**
Every webpage has HTML (the code that defines the page structure). v2 computed a **structural signature** — a SHA256 hash of the cleaned DOM (HTML Document Object Model). Same HTML structure = same screen.

```python
# v2 screen identification
signature = sha256(cleaned_dom)[:16]  # "a3f7b2c1..."
# If two screens have the same signature → they're the same screen
```

**The Problem:**
Enterprise apps (the kind Testsigma customers use) often have many screens that look structurally identical in HTML but are completely different in meaning.

```
"Create User" form      →  Signature: "a3f7b2c1"
"Edit User" form        →  Signature: "a3f7b2c1"  ← SAME! (both are just forms)
"Create Product" form   →  Signature: "a3f7b2c1"  ← SAME! 
```

**The result:** Sherlock v2 detected only **10 unique screens** in a recording that actually had **30+ distinct screens**. The knowledge graph was wrong.

### v3 — The Semantic (AI-Powered) Approach

**How it works:**
Instead of hashing HTML, Sherlock v3 uses **Gemini** (Google's AI model with vision capability) to actually *watch the video* and understand what each screen IS.

```python
# v3 screen identification
# Gemini watches the video and says:
# "At timestamp 00:23, I see a form with fields for username, email,
#  and password. The page title says 'Create New User'. This is a
#  user creation form."
screen_identity = {
    "screen_name": "Create User Form",
    "screen_purpose": "Allows admins to create new user accounts",
    "key_elements": ["username field", "email field", "password field", "Create button"]
}
```

**The result:** Sherlock can now correctly distinguish "Create User" from "Edit User" even though they have identical HTML structure, because it understands the *semantic meaning* of the screen.

### Comparison Table

| Aspect | v2 (Signature-based) | v3 (Agentic/Semantic) |
|--------|---------------------|----------------------|
| How screens are identified | SHA256 hash of HTML structure | LLM watches video, understands meaning |
| Screen matching | Exact hash match | Semantic similarity matching |
| Merge decisions | Deterministic algorithm | AI generates a `MergePlan` |
| Context across steps | Each step is isolated | Context accumulates through all steps |
| Problem | Misidentifies similar-looking screens | Computationally more expensive |
| Screens found (test) | ~10 (wrong) | ~30+ (correct) |

> **Note:** v3 kept the structural signatures from v2 — they're still used as a fast pre-filter before semantic matching. If two screens have completely different DOM structures, there's no need to do expensive semantic comparison.

---

## 7. Architecture: The Three-Layer System

Sherlock stores its knowledge in three complementary layers:

```
┌─────────────────────────────────────────────────────────────┐
│                    INTENT LAYER (Layer 1)                    │
│   What: KnowledgeNode objects (user workflows/goals)         │
│   Storage: Qdrant vector database                            │
│   Why: Semantic search — "find flows similar to checkout"    │
│   Format: 768-dimensional Gemini embeddings                  │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                  TOPOLOGICAL LAYER (Layer 2)                 │
│   What: PageState nodes ──ActionEdge──▶ PageState nodes      │
│   Storage: NetworkX DiGraph (in-memory, saved to JSON)       │
│            OR Neo4j (for production/persistence)             │
│   Why: Graph traversal — "find path from Login to Checkout"  │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                   COMPONENT LAYER (Layer 3)                  │
│   What: Reverse index — selector → [KnowledgeNode IDs]       │
│   Storage: JSON file (impact_index.json)                     │
│   Why: Impact analysis — "what breaks if #submit-btn changes"│
└─────────────────────────────────────────────────────────────┘
```

### Why Three Layers?

Each layer optimizes for a different type of query:

| Query Type | Layer Used | Why |
|-----------|-----------|-----|
| "Find workflows similar to X" | Intent (Qdrant) | Vector similarity search |
| "Get path from screen A to B" | Topological (NetworkX) | Graph traversal algorithm |
| "What breaks if selector X changes?" | Component (JSON index) | Direct lookup |

---

## 8. The Planner Agent — Using Sherlock for Test Planning

The **Planner Agent** is a separate AI system that uses Sherlock's knowledge graph to answer the question: *"Given this user story, which tests should we run?"*

Without the Planner Agent, a QA lead would spend 2-4 hours manually picking tests. With it, this takes 30-120 seconds.

### How It Works

```
User Story: "Update the payment processing timeout from 30s to 60s"
         │
         ▼
Planning Agent analyzes: What is this about? Payment. What might be affected?
         │
         ▼ (4 sub-agents run in parallel)
         │
         ├──▶ Discovery Agent
         │    Semantic search: "payment timeout", "payment processing", "checkout payment"
         │    Returns: Test cases directly about payment functionality
         │
         ├──▶ Impact Agent
         │    Traverses Sherlock's graph: Payment → Checkout → Subscriptions → Refunds
         │    Returns: All tests for features connected to Payment
         │
         ├──▶ Failure Agent
         │    Checks execution history: Which payment tests have been failing recently?
         │    Returns: Flaky or recently broken payment tests
         │
         └──▶ Critical Path Agent
              Gets critical flows from Sherlock: is_critical_path = true
              Returns: Complete Purchase, Process Refund, Subscription Renewal
         │
         ▼
Merge & Prioritize: Combine, deduplicate, score, rank
         │
         ▼
Output: Ordered list of test case IDs
1. Complete Purchase [critical path, directly relevant] — score: 0.95
2. Process Refund [critical path, impacted] — score: 0.88
3. Subscription Renewal [impacted, recently failing] — score: 0.81
4. Apply Coupon + Checkout [impacted] — score: 0.72
...
```

### The Four Sub-Agents

| Agent | What It Does | What It Uses |
|-------|-------------|-------------|
| **Discovery Agent** | Finds tests *directly* related to the story | Semantic search on test case repository |
| **Impact Agent** | Finds tests *indirectly* affected via connections | Sherlock's topology graph traversal |
| **Failure Agent** | Surfaces tests that are currently unstable | Historical execution data + failure patterns |
| **Critical Path Agent** | Ensures critical flows are always included | Sherlock's `is_critical_path` flags |

### Test Type Configuration

The Planner Agent adapts its depth based on the type of test run:

| Test Type | What It Does | Impact Hops |
|-----------|-------------|-------------|
| **Smoke** | Fast, critical paths only | 1 hop |
| **Feature** | Direct impact focus | 1 hop |
| **Regression** | Balanced analysis (default) | 2 hops |
| **Deep Regression** | Comprehensive | 3 hops |

"Hops" means how many steps through the relationship graph the Impact Agent traverses. More hops = wider net = more tests included.

---

## 9. Technology Stack — What Each Tool Does

### Core AI Tools

| Tool | What It Is | Why Sherlock Uses It |
|------|-----------|---------------------|
| **Gemini (Google AI)** | Google's multimodal AI model (can process video + text) | Screen scanner watches recordings; goal extraction understands intent |
| **google-genai SDK** | Official Python SDK for Gemini | Makes API calls to Gemini |
| **Langchain** | Framework for building LLM-powered apps | Wraps LLM calls with retry, caching, prompt management |
| **Langfuse** | LLM observability platform | Tracks every AI call — latency, cost, quality (like Datadog for AI) |

### Storage Tools

| Tool | What It Is | Why Sherlock Uses It |
|------|-----------|---------------------|
| **Qdrant** | Vector database | Stores workflow embeddings for semantic search (Intent Layer) |
| **NetworkX** | Python graph library | In-memory knowledge graph for traversal queries (Topological Layer) |
| **Neo4j** | Graph database | Production-grade persistent graph storage (replaces NetworkX in prod) |
| **JSON files** | Simple file storage | Persists the topology graph and impact index between runs |
| **Google Cloud Storage (GCS)** | Cloud file storage | Stores video files before sending to Gemini for analysis |

### Infrastructure Tools

| Tool | What It Is | Why Sherlock Uses It |
|------|-----------|---------------------|
| **FastAPI** | Python web framework | Exposes Sherlock's tools as HTTP API endpoints |
| **Typer** | Python CLI framework | Makes `uv run sherlock ingest ...` command-line interface |
| **Pydantic** | Data validation library | Defines and validates schemas (PageState, ActionEdge, MergePlan, etc.) |
| **BeautifulSoup + lxml** | HTML parsing libraries | Parses HTML pages to extract DOM structure |
| **MoviePy** | Video processing library | Extracts frames from recordings |
| **structlog** | Structured logging | JSON logs for production observability |
| **uv** | Fast Python package manager | Dependency management |

---

## 10. Directory Structure Walkthrough

```
sherlock/                           ← Root of the repository
│
├── sherlock/                       ← Main Python package (same name as repo)
│   │
│   ├── main.py                     ← CLI entry point (run: uv run sherlock --help)
│   ├── config.py                   ← All settings (reads from environment variables)
│   │
│   ├── agents/                     ← ALL AI AGENTS LIVE HERE
│   │   ├── base.py                 ← BaseAgent: Gemini client, video upload to GCS
│   │   ├── screen_scanner.py       ← ScreenScannerAgent: watches video, names screens
│   │   ├── segmentation.py         ← SegmentationAgent: splits recording into episodes
│   │   ├── her.py                  ← HERAgent: extracts user goal from each episode
│   │   ├── graph_merger.py         ← GraphMergerAgent: decides how to merge into graph
│   │   ├── feature_classifier.py   ← FeatureClassifierAgent: groups screens into features
│   │   └── tools/
│   │       └── graph_tools.py      ← Read-only tools the GraphMergerAgent can call
│   │
│   ├── graph/                      ← KNOWLEDGE GRAPH STORAGE
│   │   ├── topology.py             ← NetworkX graph wrapper (in-memory, dev)
│   │   └── neo4j_topology.py       ← Neo4j graph wrapper (persistent, prod)
│   │
│   ├── pipeline/                   ← PIPELINE ORCHESTRATION
│   │   ├── agentic_orchestrator.py ← Main entry point: runs all 7 steps in sequence
│   │   ├── plan_executor.py        ← Reads MergePlan, writes to graph (deterministic)
│   │   └── error_handlers.py       ← Recovery strategies when steps fail
│   │
│   ├── schemas/                    ← DATA MODELS (Pydantic)
│   │   ├── events.py               ← RawUIEvent, NormalizedUIEvent
│   │   ├── episodes.py             ← ProcessedEpisode
│   │   ├── graph.py                ← PageState, ActionEdge
│   │   ├── knowledge.py            ← KnowledgeNode, MergeDecision
│   │   ├── screen.py               ← ScreenScanResult, ScreenIdentity
│   │   ├── element.py              ← ElementIdentity (enriched click/fill events)
│   │   ├── feature.py              ← Feature, ClassificationResult
│   │   ├── merge_plan.py           ← MergePlan, EpisodePlan, StatePlan, EdgePlan
│   │   └── events.py               ← Raw and normalized UI event types
│   │
│   ├── services/                   ← SUPPORTING SERVICES
│   │   ├── ingestion.py            ← Parses recording directories, normalizes events
│   │   ├── event_parser.py         ← Parses tab-*.json browser event files
│   │   ├── dom_cleaner.py          ← Strips unstable CSS classes, computes signatures
│   │   ├── embedding.py            ← Wraps Gemini embedding API calls
│   │   ├── intent_engine.py        ← Vector store wrapper + HER goal extraction
│   │   ├── impact_indexer.py       ← Builds selector → affected flows reverse index
│   │   ├── video.py                ← Video validation, duration extraction
│   │   └── gcs.py                  ← Google Cloud Storage file upload helper
│   │
│   ├── storage/                    ← VECTOR STORE
│   │   ├── base.py                 ← Abstract storage interface
│   │   └── qdrant.py               ← Qdrant vector store implementation
│   │
│   └── utils/
│       └── logging.py              ← structlog setup
│
├── tests/
│   ├── unit/                       ← Tests for individual components
│   │   ├── test_dom_cleaner.py
│   │   ├── test_topology.py
│   │   ├── test_graph_merger.py
│   │   ├── test_impact_indexer.py
│   │   ├── test_feature_classifier.py
│   │   └── test_feature.py
│   └── integration/
│       └── test_flow_merging.py    ← End-to-end flow merging tests
│
├── data/
│   ├── topology_graph.json         ← Persisted NetworkX graph (dev mode)
│   └── impact_index.json           ← Persisted selector reverse index
│
├── docs/                           ← Documentation (read these!)
├── CLAUDE.md                       ← Developer quick reference (read this first)
├── HANDOVER.md                     ← Context on v3 implementation (read second)
├── PURPOSE.md                      ← What Sherlock is for and integration guide
├── SPEC_V3.md                      ← Full v3 technical specification
├── PLANNER_AGENT_ARCHITECTURE.md   ← Planner agent design doc
├── pyproject.toml                  ← Dependencies and project config
└── main.py                         ← Alternative entry point
```

### Recommended Reading Order

1. `CLAUDE.md` — Quick reference, architecture overview, key concepts
2. `PURPOSE.md` — Why Sherlock exists, use cases, Atto integration
3. `HANDOVER.md` — v2 vs v3, what's being built, implementation roadmap
4. `SPEC_V3.md` — Full technical spec (detailed)
5. `sherlock/pipeline/agentic_orchestrator.py` — The main pipeline (actual code)
6. `sherlock/agents/base.py` — How agents work
7. `sherlock/schemas/graph.py` — PageState, ActionEdge data models

---

## 11. How to Get Started Locally

### Prerequisites

- **Python 3.12** — check with `python --version`
- **uv** (Python package manager) — install with `curl -Ls https://astral.sh/uv/install.sh | sh`
- **Google API Key** — you need access to Gemini (ask your team lead)
- **Docker** (optional) — for running Neo4j locally

### Setup

```bash
# Navigate to sherlock directory
cd /Users/bharath.bhaktha/Documents/AI-Stuff/sherlock

# Make sure you're on the right branch (main branch only has README!)
git checkout feat/sherlocks-palace

# Install dependencies
uv sync

# Set up environment variables
# Create a .env file or export variables in your shell:
export SHERLOCK_GOOGLE_API_KEY="your-gemini-api-key"    # Required
export SHERLOCK_GEMINI_MODEL="gemini-1.5-pro"           # Default
export SHERLOCK_QDRANT_USE_MEMORY="true"                # Use in-memory for dev
export SHERLOCK_DATA_DIR="./data"                       # Where to save graph files
export SHERLOCK_LOG_LEVEL="info"                        # info / debug

# Verify the CLI works
uv run sherlock --help
```

### Running the Pipeline

```bash
# Process a screen recording
uv run sherlock ingest path/to/recording/ --use-video

# The recording directory must contain:
# ├── session.webm          (video file)
# ├── tab-events.json       (browser events)
# └── pagesources/          (HTML snapshots, optional)

# Search the knowledge graph
uv run sherlock search "How does the checkout flow work?"

# Impact analysis — what flows use this selector?
uv run sherlock impact "#submit-btn"

# View statistics
uv run sherlock stats         # Knowledge nodes (workflows)
uv run sherlock graph-stats   # PageStates and edges in topology
```

### Running Tests

```bash
# Run all tests
uv run pytest

# Run with coverage report
uv run pytest --cov=sherlock

# Run one specific test file
uv run pytest tests/unit/test_graph_merger.py

# Run one specific test
uv run pytest tests/unit/test_graph_merger.py::TestGraphMerger::test_bootstrap_creates_new_flow -v
```

### Key Environment Variables

| Variable | Default | What It Does |
|----------|---------|-------------|
| `SHERLOCK_GOOGLE_API_KEY` | (required) | Gemini API key — needed for all AI agents |
| `SHERLOCK_GEMINI_MODEL` | `gemini-1.5-pro` | Which Gemini model to use |
| `SHERLOCK_EMBEDDING_MODEL` | `models/embedding-001` | Gemini embedding model (768-dim vectors) |
| `SHERLOCK_QDRANT_USE_MEMORY` | `true` | `true` = in-memory Qdrant (lost on restart); `false` = persistent |
| `SHERLOCK_DATA_DIR` | `./data` | Where to save topology graph and impact index |
| `SHERLOCK_LOG_LEVEL` | `info` | `debug` for verbose output during development |

---

## 12. Key Concepts You Must Know

### Structural Signature

A DOM fingerprint — SHA256 hash of the cleaned HTML structure. Used as a fast pre-filter.

The DOM cleaner **removes unstable parts** before hashing:
- CSS class names with random suffixes (`css-xk7m2a`, `sc-abc123`, `emotion-0`) — these change every build
- Angular (`ng-`), Vue (`v-`) framework classes
- Text content (a form with "Create" and one with "Edit" should match)

**What remains:** HTML tag names + stable CSS classes + ARIA attributes (`role`, `type`, `id`, `name`).

### Semantic Anchor Merging (v2 concept, still relevant)

When a new recording arrives, Sherlock tries to connect it to existing knowledge rather than creating everything from scratch:

1. Extract the goal from the new recording
2. Find similar existing flows in Qdrant (vector search)
3. Use that matching flow's screens as "anchors"
4. "Zipper" the new recording onto the existing flow — verifying each step matches an anchor

This prevents the graph from having 50 duplicate "Login" nodes.

### HER — Hindsight Experience Replay

HER is the name of the goal extraction agent. The name comes from reinforcement learning — "hindsight experience replay" is a technique where you look at what actually happened and figure out what goal was being pursued.

In Sherlock's context: the HERAgent watches a segment of recording and asks Gemini: *"What was this user trying to accomplish? What was the outcome?"*

### MergePlan

A structured JSON document output by the Graph Merger Agent. It describes exactly what should happen to the graph without actually changing anything.

```json
{
  "recording_id": "session_abc",
  "total_episodes": 2,
  "episodes": [
    {
      "episode_id": "ep_001",
      "goal": "User logs into the application",
      "match_decision": {
        "decision_type": "REUSE_EXISTING",
        "matched_flow_id": "flow_login_123"
      },
      "state_plans": [
        {
          "action": "REUSE",
          "state_id": "state_login_page",
          "screen_name": "Login Page"
        }
      ],
      "edge_plans": [
        {
          "action": "CREATE",
          "source_id": "state_login_page",
          "target_id": "state_dashboard",
          "selector": "#login-btn"
        }
      ]
    }
  ]
}
```

The Plan Executor reads this and does the actual writes.

### Feature (in Sherlock vs in Testsigma)

In Sherlock, a **Feature** is a logical grouping of screens and workflows that belong together. Examples:
- "Authentication" → Login, Register, Forgot Password screens
- "Checkout" → Cart, Shipping, Payment, Confirmation screens
- "User Management" → User List, User Detail, Create User, Edit User screens

The Feature Classifier Agent automatically assigns screens to features using Gemini — it looks at the screen names and purposes and decides which logical feature they belong to.

### Episode Boundary

When Sherlock processes a recording, a user may do many different things in one session. An **episode** is one distinct task within that session.

Example: A 10-minute recording might contain:
- Episode 1 (0:00 - 2:30): Login
- Episode 2 (2:30 - 5:00): Search for a product
- Episode 3 (5:00 - 8:00): Add to cart and checkout
- Episode 4 (8:00 - 10:00): View order history

The Segmentation Agent identifies these boundaries.

### Impact Index

A reverse index from CSS selectors to the workflows that use them. Stored as `data/impact_index.json`.

```json
{
  "#submit-btn": ["flow_login", "flow_checkout", "flow_create_user"],
  ".nav-checkout": ["flow_cart", "flow_checkout"],
  "#email-input": ["flow_login", "flow_register", "flow_forgot_password"]
}
```

When a developer changes `#submit-btn`, you can instantly know: this affects login, checkout, and user creation flows — prioritize those tests.

---

## 13. Recent Changes and Latest Updates

The active branch is `feat/sherlocks-palace`. There have been 6 commits (the project is in early/active development). Here's what the recent activity shows:

### Latest Commits (Most Recent First)

#### 1. Refactor event parsing for improved readability and structure
- `event_parser.py` and `ingestion.py` were cleaned up for clarity
- Likely improved how browser events (clicks, typing, navigation) are parsed from `tab.json` files

#### 2. Refactor ingestion and event parsing for improved video handling and element identity
- Significant refactor to how video data is handled in the pipeline
- Introduction or improvement of **ElementIdentity** — the concept that each click/fill event now carries semantic information about the element that was interacted with (not just its CSS selector)
- This is directly tied to the **Step 2b element annotation enrichment** step in the pipeline — the ScreenScannerAgent identifies elements visually, and that info gets merged into the event log

#### 3. Refactor video handling in ingestion and agent pipeline
- Video utility code was extracted into its own service (`video.py`)
- Makes video validation and duration extraction reusable across agents

#### 4. Add Feature Layer for Planner Agent Integration (v3.1)
**This is the biggest recent addition — v3.1**

Three major additions:
- **FeatureClassifierAgent** (`agents/feature_classifier.py`) — New agent that groups screens/flows into Features using Gemini
- **Feature schema** (`schemas/feature.py`) — New data model for Feature nodes
- **Step 7 in the pipeline** — Feature classification now runs automatically after every ingestion

This is called v3.1 because it builds on v3's semantic matching (v3.0) by adding the Feature grouping layer. Now screens aren't just identified and matched — they're also organized into logical features automatically.

#### 5. Add debugging notes for edge index issue, enhance Langfuse integration
- Debugging investigation around an "edge index issue" (edges not being created correctly)
- Langfuse integration improvements — better tracing of AI calls for observability

#### 6. Initial commit
- First working version of the agentic pipeline

### What This Tells You About Where Development Is

- The team is actively building and iterating on v3
- The pipeline is functional but being refined (refactors suggest the API is stabilizing)
- Feature classification (v3.1) was just added — this is fresh code that may have rough edges
- Langfuse integration means AI call quality is being monitored — this is a sign of maturity

### What Is Not Done Yet (Based on Documentation)

Looking at `HANDOVER.md` and `PHASED_IMPLEMENTATION_V3.md`, these areas are documented as planned but may be in progress:
- Neo4j production backend (currently NetworkX/JSON is the default)
- Full Planner Agent integration (architecture documented, implementation TBD)
- Persistent Qdrant mode (currently defaults to in-memory for dev)

---

## 14. Glossary

| Term | Meaning |
|------|---------|
| **Sherlock** | Testsigma's AI system that builds a knowledge graph of an app from screen recordings |
| **Knowledge Graph** | A database of nodes (things) and edges (relationships) — represents how the app works |
| **PageState** | A unique screen in the application (identified by structure and semantic meaning) |
| **ActionEdge** | A user action that transitions from one PageState to another |
| **KnowledgeNode** | A complete user workflow (goal + steps + anchor screens) |
| **Feature** | A logical grouping of related screens and workflows (e.g., "Checkout") |
| **DOM** | Document Object Model — the internal tree structure of an HTML page |
| **Structural Signature** | SHA256 hash of cleaned HTML DOM — a fingerprint for quick screen identification |
| **Semantic Matching** | Using AI to match screens by meaning rather than by exact structure |
| **MergePlan** | Structured JSON output from the Graph Merger Agent describing what to write to the graph |
| **Plan Executor** | Deterministic component that reads a MergePlan and writes to the graph |
| **Episode** | One distinct task within a recording (a recording may contain multiple episodes) |
| **HER (Hindsight Experience Replay)** | The goal extraction agent — figures out what the user was trying to do in an episode |
| **Segmentation Agent** | Identifies where one episode ends and another begins in a recording |
| **Screen Scanner Agent** | Watches the full video and identifies every unique screen (name, purpose, elements) |
| **Graph Merger Agent** | Decides how to merge new recording data into the existing knowledge graph |
| **Feature Classifier Agent** | Groups screens and workflows into logical features |
| **Impact Index** | Reverse index from CSS selector → workflows that use it (for impact analysis) |
| **Qdrant** | Vector database storing workflow embeddings for semantic search |
| **NetworkX** | Python library for in-memory graph storage and traversal |
| **Neo4j** | Production graph database (replaces NetworkX for persistence) |
| **Embedding** | A list of numbers representing the semantic meaning of text (768-dim for Gemini) |
| **Vector Search** | Finding items whose embeddings are closest in meaning to a query |
| **Gemini** | Google's AI model used for video analysis, goal extraction, and embeddings |
| **Langfuse** | LLM observability platform — tracks every AI call's cost, latency, and quality |
| **GCS** | Google Cloud Storage — where video files are uploaded before Gemini analysis |
| **Planner Agent** | AI system that uses Sherlock's graph to plan which tests to run for a user story |
| **Discovery Agent** | Sub-agent of Planner — finds test cases directly related to a user story |
| **Impact Agent** | Sub-agent of Planner — finds test cases affected via graph relationships |
| **Failure Agent** | Sub-agent of Planner — surfaces currently failing/flaky tests |
| **Critical Path** | A user journey that is essential — if it breaks, the product fails |
| **Smoke Test** | A minimal set of tests that verify the most critical functionality |
| **Regression Test** | Tests that verify existing functionality still works after a change |
| **Parametric Edge** | An ActionEdge where the input varies (e.g., a text field — value changes each time) |
| **Occurrence Count** | How many times a screen or workflow has been observed across all recordings |
| **uv** | Fast Python package manager (used instead of pip) |
| **Pydantic** | Python library for defining and validating data models with type hints |
| **structlog** | Structured logging library — produces JSON logs for Langfuse/observability tools |
| **Typer** | Python library for building CLI tools (powers `uv run sherlock ...` commands) |

---

*This document was generated from analysis of the `feat/sherlocks-palace` branch of the Sherlock repository as of April 2026. For the latest details, always check `CLAUDE.md`, `HANDOVER.md`, and `SPEC_V3.md` in the repository root.*
