# Testsigma AI System — Architecture Flowcharts

> Eight focused diagrams + quick-reference tables.  
> Every diagram is sized to render correctly on GitHub.

---

## 1 · Master System Overview

How the three systems connect, where workflows begin, and how data flows between them.

```mermaid
flowchart TD
    UI([Testsigma Web UI])
    REC([Screen Recording\n.webm + tab.json])
    BE([Testsigma Backend])

    SQS[AWS SQS\nalpha-sync.fifo]
    ALPHA[Alpha — AI Engine\nFastAPI :8000]
    SHERLOCK[Sherlock — Knowledge Graph\nCLI Tool]
    ATTO[Atto — Browser Agent\nFastAPI :7979]

    CLAUDE[Anthropic Claude\nSonnet + Haiku]
    GEMINI[Google Gemini 2.5 Flash\n+ text-embedding-004]
    PORTKEY[Portkey AI Gateway]
    LANGFUSE[Langfuse\nLLM Observability]
    JAVA[Java Mobile Agent\nTestsigma Device Farm]
    PLAYWRIGHT[Playwright\nChromium Browser]

    QDA[(Qdrant Alpha\nTest Embeddings)]
    MYSQL[(MySQL\nMetadata + Keys)]
    FS[(Filesystem\nSigmaLang XML)]
    NEO[(Neo4j Cloud\nKnowledge Graph)]
    QDS[(Qdrant Sherlock\nKnowledgeNodes)]
    GCS[(GCS + S3\nVideos + Files)]

    UI -->|POST /generation/stream| ALPHA
    BE --> SQS -->|TEST_CASE events| ALPHA
    REC -->|sherlock ingest| SHERLOCK

    ALPHA -->|trigger browser run| ATTO
    ATTO -->|LEARN locators back| ALPHA
    SHERLOCK -.->|Planner Agent API — future| ALPHA

    ALPHA --> CLAUDE
    ALPHA --> PORTKEY
    ALPHA --> QDA
    ALPHA --> MYSQL
    ALPHA --> FS

    SHERLOCK --> GEMINI
    SHERLOCK --> LANGFUSE
    SHERLOCK --> NEO
    SHERLOCK --> QDS
    SHERLOCK --> GCS

    ATTO --> PORTKEY
    ATTO --> JAVA
    ATTO --> PLAYWRIGHT
```

### Explanation

This diagram shows the complete system as three cooperating subsystems:

- `Alpha` is the main AI generation engine. It receives requests from the Testsigma UI and backend, talks to LLM providers, searches prior test cases, stores metadata, and writes SigmaLang XML test cases to disk.
- `Sherlock` is the learning and knowledge-graph system. It starts from recordings and event logs, extracts screens, flows, and goals, and stores that knowledge in graph and vector databases.
- `Atto` is the execution agent. It actually runs tests in a browser or on mobile, observes what happened, and sends learned locator information back to Alpha.

The most important paths are:

1. A user in the web UI calls `POST /generation/stream`, which enters `Alpha`.
2. The backend can also push test-related sync events through `AWS SQS`, which Alpha consumes asynchronously.
3. Screen recordings are sent to `Sherlock`, which converts raw interaction history into structured product knowledge.
4. Alpha can trigger `Atto` when it needs real browser execution or locator learning.
5. Atto returns learned locators back to Alpha so the generated XML becomes more reliable in actual execution.

The external dependencies matter because each one supports a different type of intelligence:

- `Claude` is used mainly for Alpha’s orchestration and test-generation logic.
- `Gemini` is used mainly in Sherlock for video understanding, segmentation, embedding, and graph reasoning.
- `Portkey` acts as a gateway and proxy layer for LLM calls.
- `Langfuse` tracks Sherlock’s LLM activity for observability.
- `Playwright` powers browser execution, while the Java mobile agent powers device execution.

The storage layers are also split by purpose:

- `Qdrant Alpha` stores embeddings of test cases so Alpha can semantically search similar cases.
- `MySQL` stores operational metadata such as keys, interactions, tasks, and metrics.
- The filesystem stores SigmaLang XML artifacts.
- `Neo4j` and `Qdrant Sherlock` together store Sherlock’s product knowledge as graph structure plus semantic vectors.
- `GCS` and `S3` store heavier assets like recordings and uploaded files.

In simple terms: `Alpha` creates or updates tests, `Atto` proves them in execution, and `Sherlock` learns the application itself from observed behavior.

---

## 2 · Alpha — Request to Test Case (Core Pipeline)

From a user query all the way to generated SigmaLang XML files on disk.

```mermaid
flowchart TD
    REQ([User Query\nor User Story])
    STREAM[POST /generation/stream\nServer-Sent Events]
    GEN[POST /generation/generate\nnon-streaming]
    DISPATCH[AIEngineAction Dispatcher\nmaps action string to service]

    subgraph orchestrators[Orchestrators — Claude Agent SDK]
        OC1[AttoChatOrchestrator\nGENERATION workflow]
        OC2[AttoEditOrchestrator\nEDIT one test case]
        OC3[AttoLearnOrchestrator\nLEARN from browser run]
        OC4[AttoCombineOrchestrator\nCOMBINE test cases]
    end

    subgraph subagents[Subagents in .claude/agents/]
        SA1[sigmalang-batch-generator\nClaude Haiku\ngenerates 3-5 XML files]
        SA2[sigmalang-individual-generator\nClaude Haiku\ngenerates 1 XML file]
    end

    subgraph mcp[MCP Servers]
        MCP1[vector-store\nsearch_test_cases\nread_test_case_by_id]
        MCP2[salesforce\nfetch_object_metadata\nfetch_object_fields]
    end

    subgraph hooks[Lifecycle Hooks]
        HK1[pre_write — validate file path]
        HK2[post_write — validate XML\nupdate session manifest]
        HK3[post_read — log test case name]
        HK4[pre_delete — validate path]
        HK5[post_delete — cleanup file]
    end

    SESS[SessionManager\ncopy repr dir to session dir\ncreate .claude/agents/ YAMLs]
    CLAUDE[Anthropic Claude API\nSonnet for orchestrators\nHaiku for subagents]
    PORTKEY[Portkey AI\nLLM Gateway + Tracing]
    QDA[(Qdrant Alpha\nTest Case Embeddings)]
    FS[(Filesystem\n/opt/atto_fs\nSigmaLang XML)]
    MYSQL[(MySQL\nInteractions + Metrics)]

    REQ --> STREAM & GEN --> DISPATCH
    DISPATCH -->|ATTO_CHAT_TRIGGER_V2| OC1
    DISPATCH -->|ATTO_EDIT_TEST_CASE_V2| OC2
    DISPATCH -->|ATTO_LEARN_TEST_CASE_V2| OC3
    DISPATCH -->|ATTO_COMBINE_TEST_CASES_V2| OC4

    OC1 --> SESS --> FS
    OC1 --> SA1 & SA2
    OC1 --> MCP1 & MCP2
    OC1 --> HK1 & HK2 & HK3 & HK4 & HK5

    SA1 & SA2 -->|Read Write Edit XML| FS
    MCP1 -->|semantic search| QDA

    OC1 & OC2 & OC3 & OC4 --> CLAUDE
    OC1 --> PORTKEY
    OC1 --> MYSQL
```

### Explanation

This is the core generation pipeline inside Alpha. It starts with a user request and ends with SigmaLang XML files on disk.

The entry points are:

- `POST /generation/stream` for streamed generation with Server-Sent Events.
- `POST /generation/generate` for a normal non-streaming response.

Both routes eventually reach the `AIEngineAction Dispatcher`. That dispatcher reads the requested action string and decides which orchestrator should run:

- `AttoChatOrchestrator` handles new test generation.
- `AttoEditOrchestrator` modifies an existing test case.
- `AttoLearnOrchestrator` incorporates learning from actual execution.
- `AttoCombineOrchestrator` merges multiple test cases into one.

The generation path is centered on `AttoChatOrchestrator`. Its work is broader than just “call a model”:

1. It creates a session workspace through `SessionManager`.
2. It copies a representation directory into a session-specific working directory.
3. It creates `.claude/agents/` YAML definitions so subagents can run in a controlled session.
4. It decides whether to use a batch generator or an individual generator.
5. It reads and writes SigmaLang XML files in the filesystem.

The subagents are specialized XML producers:

- `sigmalang-batch-generator` creates several files at once, useful when the system wants multiple candidate cases or a multi-case response.
- `sigmalang-individual-generator` creates one file at a time, useful for focused generation or refinement.

The MCP servers provide extra knowledge beyond the immediate prompt:

- `vector-store` lets Alpha search semantically similar prior test cases and read them back by ID.
- `salesforce` lets Alpha enrich generated tests with domain metadata such as objects and fields when Salesforce-specific context is needed.

The hook layer is there to keep file operations safe and consistent:

- `pre_write` prevents invalid file paths.
- `post_write` validates XML and updates the session manifest.
- `post_read` logs what file was read.
- `pre_delete` and `post_delete` validate and clean up removal operations.

The data flow is therefore:

1. Request arrives.
2. Dispatcher selects an orchestrator.
3. The orchestrator calls Claude for reasoning.
4. It may query prior test knowledge through Qdrant-backed search.
5. It may fan work out to subagents.
6. Those agents write XML files to `/opt/atto_fs`.
7. Interaction and tracing information is stored in MySQL and Portkey.

The key idea is that Alpha is not a single prompt-response call. It is an orchestration system that manages sessions, tools, retrieval, guarded file writes, and multiple agent roles before a test case is finalized.

---

## 3 · Alpha — All HTTP Endpoints

Every route, method, and what it does.

```mermaid
flowchart TD
    subgraph generation[Generation — core AI trigger]
        G1[POST /generation/generate\nnon-streaming response]
        G2[POST /generation/stream\nServer-Sent Events SSE]
    end

    subgraph attor[Atto — /atto]
        A1[GET /atto/test-case/repo/case\nfetch XML as manual steps]
        A2[POST /atto/store-test-case-manual-edits]
        A3[POST /atto/accept-test-case\nadd to vector ignore list]
    end

    subgraph syncr[Sync — /ts_sync and /ts_sync_public]
        S1[POST /ts_sync/sync_test_repository/id]
        S2[POST /ts_sync/sync_test_case/id]
        S3[DELETE /ts_sync/delete_test_repository/id]
        S4[POST /ts_sync/copy_test_repository/src/tgt]
        S5[GET /ts_sync_public/task/uuid\ncheck sync task status]
    end

    subgraph tasksr[Tasks + Analytics — /tasks /jarvis]
        T1[GET /tasks\nfilter by type and entity_id]
        T2[GET /tasks/task_uuid]
        T3[GET /jarvis/dashboard/stats\nfrom_ts to_ts]
        T4[GET /jarvis/dashboard/token_usage]
        T5[GET /jarvis/logs\npage + size + timezone]
    end

    subgraph byokr[BYOK — /byok — Bring Your Own LLM Key]
        B1[POST /byok/llm_key]
        B2[GET /byok/get_all_llm_keys]
        B3[GET /byok/llm_key/key_id]
        B4[PUT /byok/llm_key]
        B5[DELETE /byok/llm_key/key_id]
        B6[POST /byok/validate_llm_key]
        B7[POST /byok/map_feature_key]
        B8[GET /byok/get_user_llm_config]
    end

    subgraph otherr[Other Endpoints]
        O1[GET /health]
        O2[GET /models/\ntenant available models]
        O3[GET /filesystem/tree]
        O4[GET /filesystem/file-content]
        O5[GET /filesystem/ui\nfilesystem browser UI]
        O6[PATCH /interaction/uuid/feedback]
        O7[GET /interaction/uuid]
        O8[api_route /gateway/bedrock/path\nAWS Bedrock proxy]
        O9[api_route /gateway/path\nPortkey proxy]
    end
```

### Explanation

This section is a route inventory for Alpha. The diagram groups endpoints by responsibility so the API surface is easier to reason about.

`Generation` routes are the main AI entry points:

- `POST /generation/generate` is used when the caller wants the complete result at once.
- `POST /generation/stream` is used when the caller wants incremental progress or partial outputs as they are generated.

`/atto` routes support test-case manipulation around the generated XML:

- fetching a stored test case and converting it into manual-step form,
- saving manual edits,
- accepting a test case and optionally updating vector-store behavior such as ignore lists.

`/ts_sync` and `/ts_sync_public` handle repository synchronization:

- syncing a whole test repository,
- syncing a single test case,
- deleting a repository copy,
- copying repositories,
- checking the status of background sync tasks.

`/tasks` and `/jarvis` provide operational visibility rather than generation:

- task lookup,
- task status inspection,
- dashboard statistics,
- token usage,
- logs with paging and timezone filters.

`/byok` is the Bring Your Own Key area. This is where tenants manage their own LLM credentials and mappings:

- create, list, fetch, update, and delete keys,
- validate whether a key works,
- map keys to features,
- fetch the active user LLM configuration.

The `Other Endpoints` bucket contains supporting infrastructure:

- health checks,
- model listing for a tenant,
- filesystem browsing and content fetch,
- interaction feedback endpoints,
- gateway proxy routes for Bedrock and Portkey.

The practical takeaway is that Alpha is not only a test-generation service. It also acts as an operational backend for syncing content, managing model keys, browsing generated artifacts, and exposing analytics.

---

## 4 · Sherlock — 7-Step Ingestion Pipeline

From a raw screen recording to a fully populated knowledge graph.

```mermaid
flowchart TD
    IN([Recording Directory\nvideo.webm\ntab-events.json\npagesources/\nscreenshots/])

    ST1[Step 1 — IngestionService\nparse tab-events.json into RawUIEvents\nclean selectors\ndeduplicate within 100ms window\ncompute structural_signature SHA256\nextract route_pattern from URL]

    ST2[Step 2 — ScreenScannerAgent\nGemini 2.5 Flash — single batch call\nwatches full video at once\nidentifies every unique screen:\nscreen_name + screen_purpose + key_elements\nproduces event_to_screen mapping]

    ST2B[Step 2b — Element Annotation\nmerge LLM-named element labels\ninto NormalizedUIEvent.element_identity\nenriches click and fill records]

    ST3[Step 3 — SegmentationAgent\nGemini 2.5 Flash\nidentifies where one task ends\nand the next begins\nidle gap heuristic: force break after 30s]

    ST4[Step 4 — HERAgent — Goal Extraction\nGemini 2.5 Flash\nfor each episode segment:\nwhat was the user trying to do?\nproduces goal + goal_summary + steps\nmax 2 retries if JSON invalid]

    ST5[Step 5 — GraphMergerAgent\nGemini 2.5 Flash + function calling\nagentic loop max 10 iterations\ncalls graph tools to inspect existing graph\nproduces MergePlan JSON\nvalidates episode-local edge indices\nmax 2 validation retries]

    ST6[Step 6 — PlanExecutor — deterministic\nno LLM involved\nPhase 1: create or reuse PageStates in Neo4j\nPhase 2: create or update ActionEdges in Neo4j\nPhase 3: embed goal text and store KnowledgeNode in Qdrant\nPhase 4: cross-episode edges + mark entry points]

    ST7[Step 7 — FeatureClassifierAgent\nGemini 2.5 Flash\ngets all unclassified PageStates\ngroups them into logical Features\ncreates new Feature nodes if needed\nassigns feature_id to states and flows]

    NEO[(Neo4j Cloud\nPageState + ActionEdge + Feature nodes)]
    QDS[(Qdrant Sherlock\nKnowledgeNode vectors 768-dim)]
    GCS[(GCS alpha-staging/sherlock/\nvideo files for Gemini)]
    LANGFUSE[Langfuse\ntrace every LLM call]

    IN --> ST1 --> ST2 --> ST2B --> ST3 --> ST4 --> ST5 --> ST6 --> ST7

    ST2 -->|upload video| GCS
    ST5 -->|LLM call tracing| LANGFUSE
    ST6 -->|write nodes and edges| NEO
    ST6 -->|store goal embedding| QDS
    ST7 -->|assign feature_id| NEO
```

### Explanation

This is Sherlock’s full ingestion pipeline. Its job is to turn noisy observational data into reusable product knowledge.

The input directory contains several complementary signals:

- `video.webm` gives the visual progression of the UI.
- `tab-events.json` provides structured interaction timing and raw browser events.
- `pagesources/` provides HTML snapshots.
- `screenshots/` provides image-level evidence for important moments.

Each step narrows raw evidence into more structured meaning:

1. `IngestionService` normalizes the event stream.
   It parses raw events, removes duplicates within a short time window, cleans selectors, computes a structural fingerprint, and derives route patterns from URLs. This turns low-level noise into stable normalized events.
2. `ScreenScannerAgent` watches the full recording and identifies distinct screens.
   Instead of thinking in clicks only, Sherlock first asks: “What screens exist in this recording?” It names each screen, describes its purpose, identifies key elements, and maps events to those screens.
3. `Element Annotation` enriches interactions with semantic element identities.
   This makes later reasoning more understandable because the system now knows not just that “a selector was clicked,” but that “the Save button” or “the Email field” was involved.
4. `SegmentationAgent` decides where one user task ends and the next begins.
   This is critical because a single recording may contain several mini-flows. The 30-second idle gap rule is a deterministic safeguard to force likely boundaries.
5. `HERAgent` extracts the user’s intent for each segment.
   At this point Sherlock moves from “what happened?” to “what was the user trying to accomplish?” It produces goals, summaries, and step lists for each episode.
6. `GraphMergerAgent` compares the extracted episode with what the graph already knows.
   It decides whether the episode represents a brand-new flow or should be merged into an existing one. This is the intelligence that prevents the graph from exploding with duplicates.
7. `PlanExecutor` applies those decisions deterministically.
   It creates or reuses `PageState` nodes, updates `ActionEdge` relationships, stores a semantic `KnowledgeNode`, and marks cross-episode structure such as entry points.
8. `FeatureClassifierAgent` groups states and flows into larger product features.
   This lifts the graph from screen-level navigation into feature-level understanding.

The storage outputs are split intentionally:

- `Neo4j` holds explicit graph structure: screens, transitions, features.
- `Qdrant Sherlock` holds embeddings of goal-level knowledge so semantic search is possible.
- `GCS` stores video artifacts used during Gemini analysis.
- `Langfuse` keeps traces of LLM reasoning for debugging and observability.

In plain language, Sherlock watches what a user did, determines what screens they visited, infers what task they were performing, and inserts that flow into a growing knowledge graph of the application.

---

## 5 · Sherlock — Agents and Graph Tools

What each AI agent does and which tools the GraphMergerAgent can call.

```mermaid
flowchart TD
    subgraph agents[Five AI Agents — all use Gemini 2.5 Flash]
        AG1[ScreenScannerAgent\nInput: video + NormalizedUIEvents\nOutput: ScreenScanResult\n  screens list of ScreenIdentity\n  event_to_screen dict\n  element_annotations list]

        AG2[SegmentationAgent\nInput: video + events + ScreenScanResult\nOutput: list of SegmentBoundary\n  start_time_ms end_time_ms\n  start_event end_event reason]

        AG3[HERAgent\nInput: video + segments + screens\nOutput: GoalExtractionResult\n  episodes list of EnrichedEpisode\n  goal + steps + variable_slots]

        AG4[GraphMergerAgent\nInput: full context + read-only tools\nOutput: MergePlan JSON\n  per episode: state_plans + edge_plans\n  match_decision: new_flow or zipper_merge]

        AG5[FeatureClassifierAgent\nInput: unclassified PageStates + existing Features\nOutput: ClassificationResult\n  classifications: state_id to feature_id\n  new_features list]
    end

    subgraph tools[Graph Tools available to GraphMergerAgent only]
        T1[find_similar_flows\ngoal_text + top_k\nQdrant semantic search\nReturns: FlowMatch list\n  flow_id + score + goal + anchor_count]

        T2[get_flow_anchors\nflow_id\nNeo4j lookup\nReturns: FlowDetails\n  anchors + edges]

        T3[find_states\nroute_pattern or screen_name_contains\nNeo4j query\nReturns: StateSummary list]

        T4[get_state_details\nstate_id\nReturns: full PageState fields]

        T5[find_features\nname_contains + is_critical filter\nReturns: FeatureSummary list]

        T6[analyze_impact\nquery text\nReturns: ImpactAnalysisResult\nblast radius of a change]
    end

    AG4 -->|calls iteratively| T1 & T2 & T3 & T4 & T5 & T6
```

### Explanation

This diagram zooms into Sherlock’s AI agents and, in particular, the tool-using behavior of `GraphMergerAgent`.

The five agents each solve a distinct problem:

- `ScreenScannerAgent` understands the visual structure of the recording and names the screens.
- `SegmentationAgent` divides a long session into meaningful episode boundaries.
- `HERAgent` reconstructs user intent and converts segments into explicit goals and steps.
- `GraphMergerAgent` determines how the new episode should relate to the existing knowledge graph.
- `FeatureClassifierAgent` groups lower-level states into higher-level product features.

`GraphMergerAgent` is the most graph-aware component. It does not blindly create new nodes. Instead, it queries tools repeatedly before producing a `MergePlan`.

Those tools answer different questions:

- `find_similar_flows`: “Have we already seen a semantically similar goal?”
- `get_flow_anchors`: “What are the key states and edges inside that known flow?”
- `find_states`: “Do we already have states matching this route or screen name?”
- `get_state_details`: “What exactly do we know about this specific state?”
- `find_features`: “Does this belong to an existing feature grouping?”
- `analyze_impact`: “If we merge or update here, what else does it affect?”

This matters because Sherlock is trying to maintain a stable long-lived graph. Without these tools, the system would likely duplicate screens, fragment flows, or attach episodes to the wrong feature.

The output, `MergePlan JSON`, is therefore a decision artifact, not the final write itself. It typically says things like:

- create a new state,
- reuse an existing state,
- connect these two states with an edge,
- zipper-merge this episode into an existing flow,
- attach the result to a known feature.

The separation between agentic planning and deterministic execution is a design strength. The LLM decides *what should happen*; the executor later decides *how to apply it safely*.

---

## 6 · Atto — All HTTP Endpoints and Agent Lifecycle

Every route and the AgentHandle state machine.

```mermaid
flowchart TD
    subgraph endpoints[11 HTTP Endpoints — FastAPI :7979]
        E1[GET /health]
        E2[GET /version]
        E3[POST /run-agent\nbody: task_list + attoSessionId + testCaseId\n+ token + app_url + app_type\n+ mobile_session_id + agent_url + tenant_id\nReturns: task_uuid]
        E4[GET /task-list/task_uuid\nreturns live task-list.json]
        E5[POST /pause-agent/task_uuid\nReturns: recording_session_id]
        E6[POST /resume-agent/task_uuid\nbody: human_answer + inserted_steps]
        E7[POST /stop-agent/task_uuid]
        E8[POST /stop-all-agents\nReturns: stopped_count]
        E9[GET /user-actions/task_uuid/recording-session/id]
        E10[POST /captured-event\nbody: context + value + task_uuid\ncontext = screenshot or mobile_recorded_action]
        E11[POST /edit-task-list\nbody: steps + instruction + app_url + token\ncalls LLM to edit task steps\nReturns: EditedStepsList]
    end

    subgraph lifecycle[AgentHandle State Machine]
        RUNNING[RUNNING\nagent executing steps]
        PAUSED[PAUSED\nwait_while_paused blocks loop]
        WAITING[WAITING_FOR_HUMAN\nask_human tool called\nrecording_session_id set]
        STOPPED[STOPPED\nunregistered from AgentRegistry]
        COMPLETED[COMPLETED\nall tasks done\nfinalize_agent_run_common called]

        RUNNING -->|POST /pause-agent| PAUSED
        PAUSED -->|POST /resume-agent| RUNNING
        RUNNING -->|ask_human tool| WAITING
        WAITING -->|POST /resume-agent with human_answer| RUNNING
        RUNNING -->|POST /stop-agent| STOPPED
        RUNNING -->|all tasks done| COMPLETED
    end

    E3 -->|app_type = WEB| RUNNING
    E3 -->|app_type = MOBILE| RUNNING
    E5 --> PAUSED
    E6 --> RUNNING
    E7 --> STOPPED
```

### Explanation

This section combines two views of Atto:

- the HTTP API surface,
- the runtime state machine of an executing agent.

The API endpoints can be read as lifecycle controls around a long-running execution session.

The most important route is `POST /run-agent`. It starts a new execution using:

- a task list,
- Atto session identifiers,
- Alpha app URL and token,
- app type information,
- optional mobile session and agent URLs,
- tenant context.

The route returns a `task_uuid`, which becomes the handle for all further operations.

The remaining endpoints support live control and observation:

- fetch the current `task-list.json`,
- pause an agent,
- resume an agent and optionally inject human input or new steps,
- stop one agent or all agents,
- fetch recorded user actions,
- submit captured events,
- edit the task list using an LLM.

The state machine explains what those controls actually do internally:

- `RUNNING` means the agent is actively executing steps.
- `PAUSED` means the loop is intentionally suspended.
- `WAITING_FOR_HUMAN` means the agent requested user intervention through a tool call and is blocked until a response arrives.
- `STOPPED` means execution has been terminated and deregistered.
- `COMPLETED` means the run reached a natural end and finalized cleanly.

The important transitions are:

1. `POST /run-agent` creates a running session for either web or mobile execution.
2. `POST /pause-agent` moves a running session to paused.
3. `POST /resume-agent` returns a paused or waiting session to active execution.
4. `ask_human` inside the agent loop moves the session into `WAITING_FOR_HUMAN`.
5. `POST /stop-agent` forces termination.
6. Finishing all tasks leads to `COMPLETED`.

This design tells you that Atto is not just a fire-and-forget runner. It is built for interactive, inspectable, recoverable execution.

---

## 7 · Atto — Web Agent Loop

How a web test executes step-by-step using browser-use and Playwright.

```mermaid
flowchart TD
    START([POST /run-agent\napp_type = WEB])

    AUTH[fetch_alpha_auth\nGET app_url/alpha/auth\nReturns: token + is_byok_mode]

    BUILD[build_llm\nmodel: gemini-2.5-pro default\nbase_url: alpha_url/gateway\nheaders: x-trace-id x-proxy-key\nx-alpha-token x-byok-mode alpha-action]

    AGENT[browser_use.Agent created\ntask = task context string\nllm = LangChain LLM\ncontroller = task_done + modify_task_list\nhooks = AgentHooks instance]

    LOOP{Agent loop\nmax 200 steps}

    STEP_START[on_step_start hook\ncheck consecutive_failures\nif failures >= 3: inject escalation\nensure JS injected via locator_service\nverify page responsive]

    LLM_CALL[LLM decides next action\nbrowser screenshot + DOM\nreturns action: click fill navigate scroll assert]

    EXECUTE[Execute action in Playwright\nCapture result]

    STEP_END[on_step_end hook\ncount failures\nextract screenshot + page_source\nextract goal + evaluation + memory\nback-fill previous step evaluation\nenrich ActionRecord with locators\npersist to task-list.json\nemit StepCompletedEvent]

    LOCATOR[LocatorService.find_locators\nprimary XPath\nbackup_locators: xpath + csspath list\nlocator_tree for Testsigma]

    PERSIST[Write to disk\noutput_dir/browser-agent-data/step_uuid/\nscreenshot.png\npage-source.html\nbackup-locators.json\nlocator-tree.json\ntask-list.json]

    DONE{all tasks\ndone?}

    LEARN[LEARN back to Alpha\nPOST alpha_url/generation/generate\naction: ATTO_LEARN_TEST_CASE_V2\nAlpha updates XML with real locators]

    COMPLETED([Session COMPLETED\nfinalize_agent_run_common])

    START --> AUTH --> BUILD --> AGENT --> LOOP
    LOOP --> STEP_START --> LLM_CALL --> EXECUTE --> STEP_END
    STEP_END --> LOCATOR --> PERSIST --> DONE
    DONE -->|no| LOOP
    DONE -->|yes| LEARN --> COMPLETED
```

### Explanation

This is the detailed execution loop for a web test run in Atto.

The process begins when a caller starts `POST /run-agent` with `app_type = WEB`. After that, Atto performs several setup steps before any actual browser action is executed.

`fetch_alpha_auth` retrieves authentication and mode information from Alpha. This is important because Atto may need special headers, tokens, or BYOK behavior before it can call Alpha’s gateway safely.

`build_llm` configures the model runtime, typically using `gemini-2.5-pro`, with Alpha’s gateway URL and tracing/auth headers. This means the web agent does not talk to the model provider in an ad hoc way; it uses Alpha’s governed access path.

Then Atto constructs a `browser_use.Agent` with:

- the task context,
- an LLM wrapper,
- controller tools,
- execution hooks.

The hook system is central to reliability:

- `on_step_start` checks whether failures are accumulating, injects escalation hints if needed, ensures JavaScript helpers are present, and verifies the page is still responsive.
- The LLM then observes the current page and decides the next action.
- Playwright executes the chosen action.
- `on_step_end` records what happened, updates evaluations, persists artifacts, and enriches action history with locator information.

The locator phase is especially important for Testsigma because execution should not end with just “the step worked once.” The system also extracts:

- a primary XPath,
- backup locators,
- a locator tree suitable for future automation stability.

Those artifacts are written to disk step by step, including screenshots, HTML page source, and locator metadata.

The `DONE` decision point checks whether the full task list has been completed. If not, the loop repeats.

If yes, Atto performs a crucial feedback step: `LEARN back to Alpha`. This sends real execution results and locators to Alpha using the `ATTO_LEARN_TEST_CASE_V2` action. Alpha can then update the generated XML with execution-proven locators instead of keeping only speculative ones.

So the web agent is not only an executor. It is also a refinement loop that improves the test artifact after observing the live application.

---

## 8 · Atto — Mobile Agent Loop

How a native Android or iOS test executes without a browser.

```mermaid
flowchart TD
    START([POST /run-agent\napp_type = ANDROID_NATIVE or IOS_NATIVE])

    AUTH[fetch_alpha_auth\nGET app_url/alpha/auth]

    SETUP[Build LLM + MobileControllerRuntime\njava_base = agent_url\nsession_id = mobile_session_id\nplatform = Android or iOS\nLoad mobile system prompt]

    LOOP{Step loop\nfor each step}

    PAUSE_CHECK{Agent\npaused?}
    WAIT[wait_while_paused\nblock until resumed or stopped]

    ESCALATION{consecutive\nfailures >= 3?}
    INJECT[Prepend escalation text\nto observation]

    OBSERVE[Build observation\nformat DOM tree + screenshot\nenvironment variables\npending user intervention]

    LLM[LLM invoke\nReturns AgentOutput\nwith list of actions]

    ACTION_LOOP[For each action in AgentOutput]

    subgraph mobile_actions[22 Available Mobile Actions]
        MA1[tap_element_by_index]
        MA2[input_text]
        MA3[clear_input]
        MA4[swipe + swipe_with_coordinates]
        MA5[go_back + press_home]
        MA6[change_orientation]
        MA7[hide_keyboard]
        MA8[assert_element_exists]
        MA9[assert_element_not_exists]
        MA10[assert_element_exists_by_text]
        MA11[assert_text_equals]
        MA12[assert_text_contains]
        MA13[assert_element_state]
        MA14[get_element_text]
        MA15[tap_at_coordinates]
        MA16[switch_to_context]
        MA17[switch_to_window_handle]
        MA18[wait]
        MA19[ask_human_action]
        MA20[done — marks task complete]
    end

    JAVA_CALL[POST agent_url/agent/api/v1/mobile_ai/execute\nbody: sessionId + platform + action + arguments\nReturns: toolResult + screenshot + involvedElement\n+ promptElementTree + pageSource]

    PERSIST[Build ActionRecord\nwrite screenshot.png + page-source.html\nappend to task-list.json\nemit StepCompletedEvent]

    DONE{is_done\ntrue?}
    COMPLETED([emit TaskCompletedEvent\ncleanup Java session\nDELETE agent_url/agent/api/v2/session_actions/id\nfinalise run])

    START --> AUTH --> SETUP --> LOOP
    LOOP --> PAUSE_CHECK
    PAUSE_CHECK -->|yes| WAIT --> LOOP
    PAUSE_CHECK -->|no| ESCALATION
    ESCALATION -->|yes| INJECT --> OBSERVE
    ESCALATION -->|no| OBSERVE
    OBSERVE --> LLM --> ACTION_LOOP --> MA1
    ACTION_LOOP --> JAVA_CALL --> PERSIST --> DONE
    DONE -->|yes| COMPLETED
    DONE -->|no| LOOP
```

### Explanation

This is the native mobile execution loop. It follows the same broad idea as the web agent but uses a different runtime and action vocabulary.

The run starts with `POST /run-agent` where `app_type` is `ANDROID_NATIVE` or `IOS_NATIVE`.

After authentication, Atto builds two important things:

- the LLM runtime that will reason about the next action,
- the `MobileControllerRuntime` that actually talks to the external Java/mobile automation layer.

The loop contains several control checks before the model acts:

- `PAUSE_CHECK` determines whether the agent should remain blocked.
- `WAIT` keeps the loop suspended until it is resumed or stopped.
- `ESCALATION` detects repeated failures and injects extra guidance into the observation so the model can change strategy.

The observation in mobile mode is broader than a simple screenshot. It may include:

- a UI tree or DOM-like representation,
- screenshot evidence,
- environment variables,
- pending human intervention context.

The LLM responds with an `AgentOutput` containing one or more actions. Those actions come from a constrained mobile action set such as tapping, input, swiping, orientation changes, assertions, context switching, waiting, or asking for human action.

Atto then calls the Java agent endpoint with the selected action and arguments. The Java layer returns execution evidence such as:

- tool result text,
- a screenshot,
- the involved element,
- an element tree suitable for prompting,
- page source.

Atto persists this into an `ActionRecord` and updates the task list. The `DONE` check then determines whether the task is complete.

If the task is complete, Atto emits completion events, cleans up the Java session, and finalizes the run.

The core difference from the web loop is that the browser/Playwright environment is replaced by a mobile controller and a fixed set of mobile actions. The architectural pattern, however, remains the same: observe, reason, act, persist, and repeat.

---

## 9 · Data Schemas

Key data models across all three systems and where each one lives.

```mermaid
flowchart TD
    subgraph sigmalang[SigmaLang XML — Filesystem /opt/atto_fs]
        SL1[TestCase\nID: int\nname: str\nplatform: web android ios]
        SL2[TestStep\ndescription: str\nelement: csspath or xpath selector\ntestData: str]
        SL3[Action types:\nNavigateTo\nClick\nEnterText\nVerifyText\nWaitForElement\nSelectOption]
        SL1 -->|1 to N| SL2
        SL2 -->|uses one| SL3
    end

    subgraph sherlock_models[Sherlock Schemas — Neo4j and Qdrant]
        PS[PageState — Neo4j node\nid: state_uuid\nscreen_name: str\nscreen_purpose: str\nkey_elements: list of str\nstructural_signature: SHA256 str\nroute_pattern: str\noccurrence_count: int\nfeature_id: str or None\nis_entry_point: bool]

        AE[ActionEdge — Neo4j relationship\nid: edge_uuid\nsource_id: PageState id\ntarget_id: PageState id\naction_type: click fill navigation\nselector: str\nis_parametric: bool\nobserved_values: list of str]

        KN[KnowledgeNode — Qdrant vector\nid: node_uuid\ngoal_text: str\nanchor_state_ids: list of PageState id\npath_edge_ids: list of ActionEdge id\nsteps: list of str\nfeature_id: str or None\nis_smoke_test: bool\nis_critical_path: bool\noccurrence_count: int]

        FT[Feature — Neo4j node\nid: feat_uuid\nname: str\ndescription: str\nis_critical: bool\nstate_count: int\nflow_count: int]

        FT -->|groups| KN
        FT -->|spans| PS
        KN -->|anchors| PS
        AE -->|source| PS
        AE -->|target| PS
    end

    subgraph atto_models[Atto Schemas — in-memory + JSON files]
        TL[TaskList\ntasks: list of TaskItem\ncurrent_task_id: str\nstatus: RUNNING PAUSED COMPLETED STOPPED ERROR]
        TI[TaskItem\nid + index + description\nstatus: PENDING IN_PROGRESS COMPLETED SKIPPED REMOVED\nevaluation: str\nadded_by: user agent recorded]
        AR[ActionRecord\nuuid + action_name + title\nparams: dict\nstatus: SUCCESS FAIL RUNNING\nresult + evaluation\nbackup_locators: list\nscreenshot: file path\npage_source: inline HTML]
        TL -->|contains| TI
        TI -->|contains| AR
    end
```

### Explanation

This final diagram summarizes the most important data models used across the system. It is easiest to understand by separating them by subsystem.

`SigmaLang XML` is Alpha’s final test artifact format:

- A `TestCase` is the top-level object.
- Each `TestCase` contains many `TestStep` entries.
- Each `TestStep` uses an action type such as navigation, click, text entry, verification, waiting, or selection.

This is the representation that ultimately matters for generated automation content. If Alpha succeeds, these XML files are what downstream systems consume.

`Sherlock` uses a graph-plus-vector representation:

- `PageState` represents a screen or stable UI state.
- `ActionEdge` represents a transition from one state to another caused by a user action.
- `KnowledgeNode` represents a higher-level goal or flow and stores a semantic embedding for search.
- `Feature` groups states and flows into larger product capabilities.

The relationship model is the key:

- features span many page states,
- knowledge nodes anchor onto page states,
- action edges connect source and target states.

This lets Sherlock answer both structural questions and semantic questions. For example:

- “Which screen follows this one when the user clicks Save?”
- “Have we seen a flow similar to ‘create a new lead’ before?”
- “Which feature does this screen belong to?”

`Atto` uses runtime execution models rather than durable knowledge models:

- `TaskList` is the full execution plan and current run status.
- `TaskItem` is one logical step or task in that plan.
- `ActionRecord` is the detailed record of what actually happened while trying to execute a task item.

This hierarchy is important:

1. A run has one `TaskList`.
2. That list contains multiple `TaskItem`s.
3. Each task item may accumulate multiple `ActionRecord`s as the agent tries, retries, or completes actions.

Together these schemas show the division of responsibilities:

- Alpha produces declarative automation artifacts.
- Sherlock builds reusable product knowledge.
- Atto stores operational execution history and evidence.

---

## Quick Reference — All AI Agents

| System | Agent | Model | Input | Output |
|--------|-------|-------|-------|--------|
| Alpha | AttoChatOrchestrator | Claude Sonnet | User query + context | SigmaLang XML files |
| Alpha | AttoEditOrchestrator | Claude Sonnet | Test case ID + edit instructions | Updated XML |
| Alpha | AttoLearnOrchestrator | Claude Sonnet | Browser run locator results | Updated XML locators |
| Alpha | AttoCombineOrchestrator | Claude Sonnet | Multiple test case IDs | Single merged XML |
| Alpha | sigmalang-batch-generator | Claude Haiku | test-cases-plan.json | 3–5 XML files |
| Alpha | sigmalang-individual-generator | Claude Haiku | test-cases-plan.json | 1 XML file |
| Sherlock | ScreenScannerAgent | Gemini 2.5 Flash | Video + events | ScreenScanResult |
| Sherlock | SegmentationAgent | Gemini 2.5 Flash | Video + events + screens | SegmentBoundary list |
| Sherlock | HERAgent | Gemini 2.5 Flash | Video + segments + screens | GoalExtractionResult |
| Sherlock | GraphMergerAgent | Gemini 2.5 Flash | Full context + graph tools | MergePlan JSON |
| Sherlock | FeatureClassifierAgent | Gemini 2.5 Flash | PageStates + features | ClassificationResult |
| Atto | Web Agent (browser-use) | Gemini 2.5 Pro | Task list + browser state | Actions + screenshots |
| Atto | Mobile Runner (LLM loop) | Gemini 2.5 Pro | Task list + device state | 22 mobile actions |

## Quick Reference — All Data Stores

| Store | Technology | Used By | What It Holds |
|-------|-----------|---------|---------------|
| `/opt/atto_fs` | OS filesystem | Alpha | SigmaLang XML, session working dirs |
| Qdrant Alpha | Qdrant vector DB | Alpha MCP | Test case embeddings for semantic search |
| MySQL | Relational DB | Alpha | Interactions, LLM keys, tasks, models, metrics |
| Neo4j Cloud | Graph DB | Sherlock | PageState, ActionEdge, Feature nodes |
| Qdrant Sherlock | Qdrant vector DB | Sherlock | KnowledgeNode goal embeddings 768-dim |
| GCS `alpha-staging/sherlock/` | Google Cloud Storage | Sherlock | Video files uploaded for Gemini analysis |
| S3 / GCS | Object storage | Alpha | Uploaded documents, PDFs, Figma files |
| `data/topology_graph.json` | JSON file | Sherlock dev | NetworkX graph persisted locally |
| `data/impact_index.json` | JSON file | Sherlock dev | selector → flow IDs reverse index |
