# Atto AI Agents — Complete Intern Guide

> **Who this is for:** New team members (interns, junior engineers) with no prior knowledge of AI agents, LLMs, or the Testsigma platform. This document explains everything from first principles.

---

## Table of Contents

1. [What Is Testsigma and What Problem Does It Solve?](#1-what-is-testsigma-and-what-problem-does-it-solve)
2. [What Are AI Agents? (Plain English)](#2-what-are-ai-agents-plain-english)
3. [The Two Repositories Explained](#3-the-two-repositories-explained)
4. [How Atto Agents Work — Architecture Deep Dive](#4-how-atto-agents-work--architecture-deep-dive)
5. [Key Concepts You Must Know](#5-key-concepts-you-must-know)
6. [Technology Stack — What Each Tool Does](#6-technology-stack--what-each-tool-does)
7. [How to Get Started Locally](#7-how-to-get-started-locally)
8. [Directory Structure Walkthrough](#8-directory-structure-walkthrough)
9. [Important Configuration Files](#9-important-configuration-files)
10. [Recent Changes and Latest Updates](#10-recent-changes-and-latest-updates)
11. [Glossary](#11-glossary)

---

## 1. What Is Testsigma and What Problem Does It Solve?

**Testsigma** is a software testing platform. When developers build software (websites, mobile apps, etc.), they need to make sure their software works correctly. This is done through "testing" — running automated checks that verify the software behaves as expected.

Traditionally, writing these tests required software engineers to write a lot of code manually. **Testsigma's goal is to make testing easier**, especially for people who aren't expert programmers.

**Atto** is Testsigma's AI layer — a set of AI agents that can:
- **Generate test cases** from plain English descriptions
- **Execute tests** in a real browser without human intervention
- **Learn** from how tests actually run and fix broken test steps automatically

Think of Atto as a smart assistant that understands what you want to test, writes the test, and then also runs it for you.

---

## 2. What Are AI Agents? (Plain English)

Before diving into the code, you need to understand what an "AI agent" is.

### The Difference Between an LLM and an Agent

| Concept | What It Is | Example |
|---------|-----------|---------|
| **LLM (Large Language Model)** | An AI model that reads text and generates text in response. It cannot take actions. | ChatGPT answering a question |
| **AI Agent** | An LLM that is also given **tools** — abilities to actually *do* things (read files, search the web, run code, click buttons) | An AI that can *open* a browser, *navigate* to a website, and *click* a button |

An **AI agent** works in a loop:
1. The LLM receives a task (e.g., "Write a test that logs into Google")
2. The LLM decides what tool to use (e.g., "I'll use the `read_file` tool to check existing test examples")
3. The tool runs and returns a result
4. The LLM reads the result and decides the next step
5. This loop repeats until the task is done

This is called the **agentic loop** or **ReAct loop** (Reason + Act).

### What Does "Orchestrator + Subagents" Mean?

When a task is complex, one agent is not enough. Atto uses a pattern where:
- An **Orchestrator** (the main agent) receives the high-level task and breaks it down
- **Subagents** (specialist agents) each handle one part of the task
- The orchestrator collects results from all subagents and combines them

This is exactly like how a manager delegates work to different team members.

---

## 3. The Two Repositories Explained

You have two repositories combined in `/Documents/AI-Stuff/`:

```
AI-Stuff/
├── alpha/                    ← Repository 1: The "brain" (test generation)
└── atto-browser-agent-v2/    ← Repository 2: The "hands" (browser execution)
```

### Repository 1: `alpha/` — The AI Engine

**What it does:** This is the backend service that uses Claude (Anthropic's AI model) to:
- Understand what tests a user wants
- Generate test cases as structured XML files
- Edit existing test cases
- Answer questions about test coverage

**Analogy:** Think of `alpha` as a very smart writer who knows how to write test scripts. You give it instructions in plain English, and it writes the test for you.

**Built with:** Python, FastAPI, Claude Agent SDK

---

### Repository 2: `atto-browser-agent-v2/` — The Browser Agent

**What it does:** This service takes test steps (like "click the login button") and actually executes them in a real web browser or mobile device.

**Analogy:** Think of `atto-browser-agent-v2` as a robot that can control a computer. It opens a browser, navigates to websites, clicks buttons, fills in forms — all automatically.

**Built with:** Python, FastAPI, Playwright (for browser control), browser-use (AI browser automation library)

---

### How They Work Together

```
User
  │
  │  "Create a test that logs into our app"
  ▼
alpha (AI Engine)
  │
  │  Generates test steps as XML:
  │  1. Navigate to /login
  │  2. Enter username
  │  3. Click submit
  │
  ▼
atto-browser-agent-v2 (Browser Agent)
  │
  │  Opens Chrome, goes to /login, types username, clicks submit
  │
  ▼
Test Result: PASS / FAIL + screenshots
```

---

## 4. How Atto Agents Work — Architecture Deep Dive

### 4.1 The Alpha Orchestrator (Test Case Generator)

When a user asks Atto to generate test cases, this happens:

```
User Request
     │
     ▼
AttoCoderOrchestrator  ← Main orchestrator (Claude Sonnet/Opus)
     │
     │  Analyzes: What tests are needed?
     │  Searches: Are there existing similar tests? (vector search)
     │  Decides: How many subagents to spawn?
     │
     ├──► Subagent 1 (sigmalang-batch-generator)   ← Claude Haiku
     │         Generates 3-5 related test cases
     │
     ├──► Subagent 2 (sigmalang-individual-generator) ← Claude Haiku
     │         Generates one complex test case
     │
     └──► Results collected → Test cases written to filesystem as XML
```

**Why two models (Sonnet/Opus for orchestrator, Haiku for subagents)?**
- The orchestrator needs to be smart and reason about the big picture → uses a more capable (expensive) model
- Subagents do repetitive, structured work → use a faster, cheaper model (Haiku)

### 4.2 The Three Workflow Types

#### Workflow 1: GENERATION (Creating test cases)
This is a **3-phase process**:

| Phase | What Happens |
|-------|-------------|
| **Phase 1: Analysis** | Orchestrator reads existing test cases, understands the current test suite, and plans what new tests to create |
| **Phase 2: Generation** | Subagents generate the actual test case XML files in parallel |
| **Phase 3: Review** | Orchestrator reviews generated tests, ensures quality, fixes issues |

#### Workflow 2: EDIT (Modifying one test case)
When a user wants to change an existing test:
1. Orchestrator reads the existing test case XML
2. Understands what changes are needed
3. Makes targeted edits (using file editing tools)
4. Saves the updated test case

#### Workflow 3: LEARN (Learning from actual browser execution)
This is the most interesting workflow:
1. The browser agent executes a test in a real browser
2. It captures what actually happened (which HTML elements were clicked, what locators worked)
3. This information is sent back to Alpha
4. Alpha's "learn" orchestrator updates the test case to reflect what actually works

This makes the system **self-improving** — tests get better over time based on real execution.

### 4.3 The Browser Agent Loop (atto-browser-agent-v2)

When the browser agent runs a test, it follows this loop:

```
Start: Receive task list (list of test steps)
          │
          ▼
    Take screenshot of current browser state
          │
          ▼
    LLM analyzes: "What do I see? What's the next step?"
          │
          ▼
    LLM decides on an action:
    - click(element)
    - type(text)
    - navigate(url)
    - scroll()
    - assert(condition)
          │
          ▼
    Execute action in browser (via Playwright)
          │
          ▼
    Record result (screenshot, locator info, pass/fail)
          │
          ▼
    Repeat until all steps complete
          │
          ▼
    Send results back to Alpha
```

#### Web vs Mobile Execution

| Aspect | Web (Browser) | Mobile (Native App) |
|--------|--------------|---------------------|
| **Tool** | Playwright + browser-use | Java Mobile Controller |
| **Interaction** | Click, type, navigate URLs | Tap, swipe, enter text |
| **Vision** | Screenshots of webpage | Screenshots of device screen |
| **Locators** | CSS selectors, XPath | Accessibility IDs, XPath |
| **Special** | DOM analysis for element finding | Native Android/iOS element detection |

---

## 5. Key Concepts You Must Know

### 5.1 SigmaLang / DSL (Domain-Specific Language)

Test cases in Atto are stored as **XML files** in a custom format called SigmaLang. This is a "language" designed specifically for describing test steps.

Example SigmaLang test case:
```xml
<TestCase ID="98" name="User Login Test">
  <TestStep description="Navigate to login page">
    <NavigateTo testData="https://app.example.com/login"/>
  </TestStep>
  <TestStep description="Enter email address">
    <EnterText element="csspath,#email-input" testData="user@example.com"/>
  </TestStep>
  <TestStep description="Enter password">
    <EnterText element="csspath,#password-input" testData="secret123"/>
  </TestStep>
  <TestStep description="Click the login button">
    <Click element="csspath,#submit-btn"/>
  </TestStep>
  <TestStep description="Verify login was successful">
    <VerifyText element="csspath,.welcome-message" testData="Welcome back"/>
  </TestStep>
</TestCase>
```

**Key elements:**
- `element="csspath,#email-input"` → tells the agent HOW to find the element on the page
- `testData="..."` → the actual value to use (text to type, URL to visit, etc.)
- `description="..."` → plain English description of what this step does

### 5.2 Locators

A **locator** is a way to identify a specific element on a webpage. Atto uses several types:

| Locator Type | Example | When Used |
|-------------|---------|-----------|
| **CSS Selector** | `csspath,#login-btn` | Most web elements |
| **XPath** | `xpath,//button[@id='submit']` | Complex HTML structures |
| **Accessibility ID** | `id,loginButton` | Mobile apps |
| **Text** | `text,Login` | Buttons with visible text |

The browser agent automatically finds the best locator for each element and saves it for future runs.

### 5.3 Vector Search / Embeddings

When you ask Atto to create a test, it doesn't start from scratch. It first searches for **similar existing tests** to use as inspiration.

**How this works:**
1. Every test case is converted into a "vector" (a list of numbers that represents its meaning)
2. These vectors are stored in **Qdrant** (a vector database)
3. When a new request comes in, it's also converted to a vector
4. Qdrant finds the most similar existing tests
5. The orchestrator uses those as examples to guide generation

**Analogy:** It's like a search engine, but instead of matching keywords, it matches *meaning*. So searching for "login test" would also find tests about "authentication" and "sign-in flow."

### 5.4 MCP (Model Context Protocol)

MCP is a standard protocol that lets AI models use external tools. In Alpha, the vector search tools are exposed as MCP tools, meaning the Claude agent can call them just like any other tool.

You'll see MCP configured in the orchestrator — it registers a local MCP server that provides `search_test_cases` and `read_test_case_by_id` tools.

### 5.5 Hooks (Tool Lifecycle Events)

The orchestrator registers **hooks** — functions that run automatically before or after a tool is called.

```python
# PreToolUse hook: runs BEFORE the agent uses a tool
def before_tool_use(tool_name, tool_input):
    # Log what tool is about to be called
    # Block dangerous operations
    # Inject additional context

# PostToolUse hook: runs AFTER the agent uses a tool
def after_tool_use(tool_name, tool_result):
    # Log results
    # Transform output
    # Update session state
```

These hooks are how Atto monitors what the agent is doing, enforces safety (e.g., preventing the agent from writing outside its working directory), and streams progress to the user.

### 5.6 Sessions

Each agent run is tracked as a **session** with a unique ID. Sessions store:
- The working directory for this run
- Current progress and state
- Partial results (so work isn't lost if interrupted)
- The ability to **pause** and **resume**

The pause/resume feature is important for the browser agent — if a test needs human input (like solving a CAPTCHA), the agent can pause, wait for the human to act, and then continue.

### 5.7 Multi-tenancy

Testsigma is a SaaS product used by many companies ("tenants"). Each tenant's data must be completely isolated from other tenants. In Alpha:

- All filesystem paths include the `tenant_id`: `/opt/atto_fs/tenant/{tenant_id}/test-repo-{repo_id}/`
- Vector store searches always filter by `tenant_id`
- Database queries are scoped to the tenant

This ensures Company A can never see or access Company B's test cases.

---

## 6. Technology Stack — What Each Tool Does

### Core AI/LLM Tools

| Tool | What It Is | Why It's Used |
|------|-----------|---------------|
| **Anthropic Claude** | The AI model (Claude Haiku, Sonnet, Opus) | The "brain" that generates test cases and makes decisions |
| **Claude Agent SDK** | Anthropic's SDK for building multi-agent systems | Manages the orchestrator + subagent pattern |
| **browser-use** | Python library for LLM-controlled browser automation | Connects an LLM to Playwright so AI can control a browser |
| **Portkey AI** | AI gateway/proxy | Routes API calls to different LLMs, handles retries and logging |

### Infrastructure

| Tool | What It Is | Why It's Used |
|------|-----------|---------------|
| **FastAPI** | Python web framework | Exposes HTTP API endpoints for both Alpha and the browser agent |
| **Playwright** | Browser automation library | Controls Chrome/Firefox programmatically (web agent) |
| **Qdrant** | Vector database | Stores and searches test case embeddings |
| **MySQL** | Relational database | Stores structured data (test metadata, tenant info) |
| **AWS SQS** | Message queue | Syncs test cases from Testsigma platform to Alpha |
| **AWS S3 / GCS** | File storage | Stores large files (documents, PDFs, screenshots) |
| **Hazelcast** | Distributed cache | Caches frequently-accessed data (currently disabled in prod) |
| **Datadog** | Observability platform | Logs, metrics, and traces for monitoring in production |

### Python Tooling

| Tool | What It Is | Why It's Used |
|------|-----------|---------------|
| **uv** | Fast Python package manager | Replaces pip, much faster dependency installation |
| **pyproject.toml** | Python project config file | Defines dependencies and project metadata |
| **structlog** | Structured logging library | Produces JSON logs for easier querying in Datadog |
| **pydantic** | Data validation library | Validates API request/response data models |
| **tenacity** | Retry library | Automatically retries failed API calls |

---

## 7. How to Get Started Locally

### Prerequisites

Make sure you have these installed:
- **Python 3.12** (`python --version`)
- **uv** (Python package manager): `curl -Ls https://astral.sh/uv/install.sh | sh`
- **Docker** and **Docker Compose** (for local services like Qdrant)
- **Git** (`git --version`)

---

### Setting Up `alpha` (Test Case Generator)

```bash
# Navigate to the alpha directory
cd /Users/bharath.bhaktha/Documents/AI-Stuff/alpha

# Install Python dependencies
uv sync

# Set up your local environment file
cp .env.local .env
# Then open .env and fill in the required values (ask your team lead for keys)

# Start the vector database (Qdrant) — required for vector search
cd dev-setup/docker/qdrant
docker compose up -d
cd ../../..

# Run the Alpha service
uv run main.py
```

The Alpha API will be available at `http://localhost:8000`.

**Key environment variables you need to set:**
```
ANTHROPIC_API_KEY=       # Your Claude API key
OPENAI_API_KEY=          # OpenAI key (used for some embeddings)
MYSQL_HOST=              # Database host
MYSQL_USER=              # Database username
MYSQL_PASSWORD=          # Database password
QDRANT_URL=              # Vector database URL
ATTO_DATA_DIR=           # Local path for test case filesystem
```

---

### Setting Up `atto-browser-agent-v2` (Browser Agent)

```bash
# Navigate to the browser agent directory
cd /Users/bharath.bhaktha/Documents/AI-Stuff/atto-browser-agent-v2

# Install Python dependencies
uv sync

# Install Playwright browsers (Chrome, Firefox, etc.)
uv run playwright install chromium

# Set up your local environment file
cp .env.example .env
# Open .env and fill in required values

# Start the browser agent server (with hot-reload for development)
./run.sh
```

The browser agent API will be available at `http://127.0.0.1:7979`.

**Key environment variables:**
```
ALPHA_URL=http://localhost:8000   # URL of the Alpha service
DEFAULT_MODEL=gemini-2.5-pro      # The LLM to use for browser decisions
HEADLESS=false                    # Set to true to hide the browser window
```

---

### How to Test the Browser Agent

Once both services are running, you can start a browser agent run:

```bash
curl -X POST http://127.0.0.1:7979/run-agent \
  -H "Content-Type: application/json" \
  -d '{
    "task_uuid": "my-test-run-001",
    "steps": [
      "Navigate to https://example.com",
      "Verify the page title says Example Domain"
    ]
  }'
```

Check the status:
```bash
curl http://127.0.0.1:7979/task-list/my-test-run-001
```

---

## 8. Directory Structure Walkthrough

### Alpha (`alpha/`)

```
alpha/
│
├── alpha/                          ← Main Python package (same name as repo)
│   │
│   ├── aiengine/                   ← Non-agent AI features
│   │   ├── actions/                ← Pluggable action modules
│   │   │   ├── atto/               ← Atto-specific actions (recorder, etc.)
│   │   │   ├── figma_actions/      ← Import designs from Figma
│   │   │   ├── swagger_actions/    ← Import API specs from Swagger
│   │   │   └── visual_analysis/    ← Screenshot comparison
│   │   ├── llm/                    ← Raw LLM API wrappers
│   │   └── prompts/                ← Prompt templates for non-agent flows
│   │
│   ├── app/                        ← FastAPI Application
│   │   ├── routers/                ← HTTP endpoints (URLs)
│   │   │   ├── atto.py             ← POST /atto/generate, etc.
│   │   │   ├── generation.py       ← Other generation endpoints
│   │   │   └── ts_sync.py          ← Sync endpoints
│   │   │
│   │   ├── service/                ← Business logic
│   │   │   ├── atto/               ← THE MAIN ATTO AGENT CODE
│   │   │   │   ├── orchestrator/   ← Claude SDK orchestrators (entry points)
│   │   │   │   │   ├── orchestrator.py        ← GENERATION workflow
│   │   │   │   │   ├── edit_orchestrator.py   ← EDIT workflow
│   │   │   │   │   ├── learn_orchestrator.py  ← LEARN workflow
│   │   │   │   │   └── combine_orchestrator.py ← COMBINE workflow
│   │   │   │   ├── subagents.py    ← Subagent definitions (batch & individual)
│   │   │   │   ├── prompts.py      ← System prompts (59KB! Very important)
│   │   │   │   ├── message_handler.py ← Streams events to frontend
│   │   │   │   ├── hooks/          ← PreToolUse / PostToolUse hooks
│   │   │   │   ├── mcp/            ← MCP server for vector search tools
│   │   │   │   ├── filesystem.py   ← Resolves tenant file paths
│   │   │   │   └── session.py      ← Session state management
│   │   │   │
│   │   │   ├── generation/         ← Entry points for generation flows
│   │   │   │   └── atto/
│   │   │   │       ├── atto_test_case_service_v2.py  ← GENERATION entry
│   │   │   │       ├── atto_test_case_edit_service.py ← EDIT entry
│   │   │   │       └── test_case_learn_service.py    ← LEARN entry
│   │   │   │
│   │   │   └── messaging/          ← SQS message consumers
│   │   │
│   │   └── models/                 ← Pydantic request/response models
│   │
│   ├── atlas/                      ← Data layer (storage abstractions)
│   │   ├── vector_store/           ← Qdrant client wrapper
│   │   ├── storage/                ← S3/GCS file storage wrapper
│   │   ├── messaging/              ← SQS message queue wrapper
│   │   └── cache/                  ← Hazelcast cache wrapper
│   │
│   └── settings/                   ← App configuration (reads from .env)
│
├── docs/atto/                      ← DOCUMENTATION (read these first!)
│   ├── overview.md                 ← Start here
│   ├── agent-architecture.md       ← Orchestrator pattern details
│   ├── generation-workflow.md      ← How generation works step by step
│   ├── edit-workflow.md            ← How editing works
│   ├── learn-workflow.md           ← How learning works
│   └── filesystem.md               ← File structure explanation
│
├── data/dsl/                       ← SigmaLang examples (VERY useful to read)
│   ├── llm_examples-web.md         ← Web test examples
│   ├── llm_examples-android.md     ← Android test examples
│   └── llm_examples-ios.md         ← iOS test examples
│
├── .env.production                 ← Prod config (reference only, never edit)
├── pyproject.toml                  ← Dependencies
└── main.py                         ← App entry point (run this to start)
```

**Recommended reading order for new team members:**
1. `docs/atto/overview.md`
2. `docs/atto/agent-architecture.md`
3. `data/dsl/llm_examples-web.md`
4. `docs/atto/generation-workflow.md`
5. Source: `alpha/app/service/atto/orchestrator/orchestrator.py`

---

### Atto Browser Agent (`atto-browser-agent-v2/`)

```
atto-browser-agent-v2/
│
├── src/atto/
│   │
│   ├── agent/                      ← THE CORE AGENT CODE
│   │   ├── service.py              ← AgentHandle: manages one agent run's lifecycle
│   │   ├── hooks.py                ← Web browser lifecycle hooks (after each step)
│   │   ├── session.py              ← Session state (pause/resume support)
│   │   ├── task_list_controller.py ← Manages the list of test steps
│   │   ├── step_common.py          ← Logic shared between web and mobile steps
│   │   ├── step_persistence.py     ← Saves step results to database
│   │   ├── mobile_runner.py        ← The mobile agent loop (no browser-use)
│   │   └── mobile_controller.py    ← Tool definitions for mobile actions
│   │
│   ├── api/
│   │   └── app.py                  ← All HTTP endpoints (run-agent, pause, resume)
│   │
│   ├── dom/
│   │   └── compressor.py           ← Shrinks HTML before sending to LLM (saves tokens)
│   │
│   ├── tasklist/                   ← Task list data models and management
│   ├── events/                     ← Server-sent events for real-time progress
│   ├── locators/                   ← XPath/CSS locator enrichment
│   ├── output/                     ← Screenshots, GIF recording, eval tracking
│   └── extensions/                 ← Chrome extension loading
│
├── docs/                           ← Documentation
│   ├── local-setup.md              ← Start here for local dev
│   ├── api-reference.md            ← All API endpoints with examples
│   ├── environment-variables.md    ← Every .env variable explained
│   └── agentic-loop-web-mobile.md  ← Web vs mobile architecture
│
├── .env.example                    ← Template for your .env file
├── pyproject.toml                  ← Dependencies
└── run.sh                          ← Start dev server (use this, not python directly)
```

---

## 9. Important Configuration Files

### Understanding `.env` Files

Environment variables (`.env` files) configure the app without hardcoding secrets in the code. **Never commit `.env` files to Git** — they contain API keys and passwords.

The pattern is:
- `.env.example` or `.env.local` — a template with fake/empty values → safe to commit
- `.env` — your actual local config → **never commit this**
- `.env.production` — production config → managed by DevOps team

### Alpha Key Configuration

```env
# Which environment are we in?
ENVIRONMENT=development          # or: production, staging

# Which AI model to use for orchestration
# (subagents are hardcoded to Haiku in the code)
ORCHESTRATOR_MODEL=claude-sonnet-4-5

# Where test case files are stored (on the local filesystem)
ATTO_DATA_DIR=/tmp/atto_fs       # dev: use /tmp; prod: /opt/atto_fs

# Vector database
QDRANT_URL=http://localhost:6333  # dev: local Docker
QDRANT_API_KEY=                   # only needed for cloud Qdrant

# Database
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_DATABASE=alpha_db
MYSQL_USER=root
MYSQL_PASSWORD=yourpassword

# AI API Keys
ANTHROPIC_API_KEY=sk-ant-...     # Get from team lead
OPENAI_API_KEY=sk-...            # For embeddings

# File storage
STORAGE_PROVIDER=LOCAL           # dev: LOCAL; prod: S3
FILE_STORAGE_PATH=/tmp/ts_data

# Messaging
MESSAGE_QUEUE_PROVIDER=LOCAL     # dev: LOCAL; prod: SQS

# Logging
LOG_LEVEL=DEBUG                  # dev: DEBUG; prod: INFO
```

### Browser Agent Key Configuration

```env
# Server config
HOST=0.0.0.0
PORT=7979

# REQUIRED: URL of the Alpha service
ALPHA_URL=http://localhost:8000

# Which LLM the browser agent uses to make decisions
DEFAULT_MODEL=gemini-2.5-pro     # or: claude-3-5-haiku, gpt-4o

# Browser settings
HEADLESS=false           # false = you can see the browser; true = background
WINDOW_WIDTH=1920
WINDOW_HEIGHT=1080
MAX_STEPS=200            # Maximum agent steps before giving up

# Where to save screenshots and temp files
TEMP_DIR=/tmp/browser-agent

# Logging
LOG_LEVEL=INFO
LOG_JSON=false           # true = JSON format for Datadog; false = human readable
```

---

## 10. Recent Changes and Latest Updates

This section covers what has been actively worked on in both repositories recently. Understanding recent changes tells you where the team's focus is.

---

### Recent Changes in `alpha` (Test Case Generator)

#### Datadog Integration (Very Recent)
The team has just integrated **Datadog** for production observability. This means:
- Logs are now structured (JSON format) and sent to Datadog
- You can search logs by `tenant_id`, `session_id`, `workflow_type` etc.
- Environment variable `LOG_LEVEL` now controls log verbosity across all environments

**What this means for you:** When debugging, logs are now queryable in Datadog. In local dev, logs go to stdout. You can set `LOG_LEVEL=DEBUG` to see detailed agent decision-making.

#### Parallel Tool Calls (Recent Performance Improvement)
The orchestrator was updated to allow **parallel tool calls**. Previously, the agent called tools one at a time. Now, when the agent needs to read multiple files, it can request all of them simultaneously.

**Impact:** Faster test generation, especially when the agent needs to read many existing test cases for context.

#### Security Improvements (Recent)
The team added security hardening:
- File system hooks now strictly enforce that agents can only read/write within their assigned `working_dir`
- An attempt to access files outside the tenant directory is blocked and logged

#### GitHub App Token Generation (Bug Fix)
A fix was applied to GitHub App token generation — this likely affects the Jira/GitHub integration where Atto can read GitHub issues to generate test cases.

---

### Recent Changes in `atto-browser-agent-v2` (Browser Agent)

#### Mobile Agent — Major Feature Addition
The biggest recent addition is **native mobile app testing support**. Previously, the browser agent only supported web browsers. Now it also supports:
- Native Android apps
- Native iOS apps

The mobile agent has a completely separate execution loop (`mobile_runner.py`) because it doesn't use a browser — instead it sends commands to a Java-based mobile controller that controls a real device.

**Key files added/changed:**
- `src/atto/agent/mobile_runner.py` — The mobile agent loop
- `src/atto/agent/mobile_controller.py` — Tool registry for mobile actions (tap, swipe, etc.)
- `src/atto/agent/mobile_prompts.py` — Mobile-specific system prompts
- `docs/agentic-loop-web-mobile.md` — Architecture documentation

#### Mobile Session as Input (Very Recent)
The mobile agent was updated to **accept an existing mobile session** as input instead of always creating a new one. This allows the mobile agent to connect to an already-opened app session.

**Why this matters:** In practice, the Testsigma platform pre-launches the mobile app on a device, then passes the session ID to the browser agent. The agent connects to the already-running session instead of starting from scratch.

#### Pause/Resume Feature (Recent)
Both web and mobile agents now support **pause/resume**:
1. `POST /pause-agent/{task_uuid}` — Agent pauses between steps
2. Human performs an action (fills CAPTCHA, clicks a popup, etc.)
3. `POST /captured-event` — Records what the human did
4. `POST /resume-agent/{task_uuid}` — Agent continues with updated context

This is critical for testing scenarios that require human intervention (2FA, CAPTCHAs, SSO logins).

#### Learning Flow Fixes (Recent)
The learning flow — where the browser agent reports back to Alpha what locators it actually used — had bugs fixed recently. This ensures that after a test run, Alpha correctly updates test case XML with working locators.

---

### Timeline of Changes (Approximate)

```
[Most Recent]
    │
    ├── Alpha: Datadog logs integration
    ├── Alpha: Env-level log level control
    ├── Browser Agent: Mobile prompt build fix
    ├── Browser Agent: Get mobile session as input
    ├── Browser Agent: TP-1876 mobile bug fix
    │
    ├── Alpha: Parallel tool calls improvement
    ├── Alpha: Security hardening (filesystem isolation)
    ├── Browser Agent: Pause/resume implementation
    ├── Browser Agent: Learning flow fixes
    │
    ├── Browser Agent: Mobile agent loop added (major feature)
    ├── Browser Agent: Mobile controller + tools
    │
    ├── Alpha: GitHub App token fix
    │
[Older]
```

---

## 11. Glossary

| Term | Meaning |
|------|---------|
| **Atto** | Testsigma's AI agent framework (both Alpha + browser agent combined) |
| **Alpha** | The backend service that generates and manages test cases using Claude |
| **Browser Agent** | The service that executes test steps in a real browser or mobile device |
| **LLM** | Large Language Model — the AI model (Claude, GPT, Gemini) |
| **Agent** | An LLM that can use tools to take real actions |
| **Orchestrator** | The main agent that manages subagents and coordinates complex tasks |
| **Subagent** | A specialist agent spawned by the orchestrator for a specific sub-task |
| **SigmaLang** | Testsigma's XML-based language for describing test steps |
| **DSL** | Domain-Specific Language — a language designed for one specific purpose (SigmaLang is a DSL for tests) |
| **Locator** | A way to identify an HTML element on a page (CSS selector, XPath, etc.) |
| **Vector** | A list of numbers that represents the "meaning" of text |
| **Qdrant** | The vector database that stores test case embeddings for semantic search |
| **Embedding** | The process of converting text into a vector |
| **MCP** | Model Context Protocol — a standard for giving AI models tools to use |
| **Hook** | A function that runs automatically before/after an event (like a tool call) |
| **Session** | A tracked unit of work with an ID, state, and ability to pause/resume |
| **Tenant** | A company/organization using Testsigma (multi-tenant = many companies share one platform) |
| **SaaS** | Software as a Service — software hosted in the cloud and used by many customers |
| **FastAPI** | Python web framework used to create HTTP APIs |
| **Playwright** | Library for programmatic browser control |
| **browser-use** | Library that connects an LLM to Playwright for AI-powered web automation |
| **Haiku / Sonnet / Opus** | Claude model tiers — Haiku is fastest/cheapest, Opus is most capable/expensive |
| **Portkey** | An AI gateway that routes requests to different LLM providers |
| **SQS** | AWS Simple Queue Service — a message queue for async communication |
| **S3** | AWS Simple Storage Service — file/blob storage |
| **Datadog** | Observability platform for logs, metrics, and traces |
| **uv** | Fast Python package manager (like pip but much faster) |
| **pyproject.toml** | Python project configuration file (dependencies, scripts, etc.) |
| **Pydantic** | Python library for data validation using type hints |
| **FIFO Queue** | First In, First Out queue — messages are processed in the order they arrive |
| **Agentic Loop** | The Reason → Act → Observe → Repeat cycle that agents follow |
| **ReAct** | "Reasoning + Acting" — the core pattern behind AI agents |
| **Parallel Tool Calls** | When an agent calls multiple tools simultaneously instead of one at a time |

---

*This document was generated from analysis of the `alpha` and `atto-browser-agent-v2` repositories as of April 2026. For the latest updates, check the `docs/` directory in each repository and ask your team lead.*
